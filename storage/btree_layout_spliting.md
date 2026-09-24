# Mental Model: B Tree vs. B+ Tree and Node Splitting Mechanics.

Background: Some notes concerning the development of the indexing engine for VectorDB.

## Why I Stuck with a Pure B-Tree over a B+ Tree:

If you consider production grade storage engines such as InnoDB or SQLite almost everybody ends up using a B+ tree. . However, when developing this engine I did implement a Pure Classic B-Tree. Instead of seeing this merely as a departure from what is taught in textbooks, it is worthwhile examining the specific structural compromises that occur on disk.

## The Core Difference:

- In a B+ Tree, keys are duplicated; the leaf nodes contain all the actual keys, whereas the internal parent nodes only contain copies of the keys to use as routing indicators. In order to read any record, you generally have to go all the way down to a leaf page.

- In a Pure B-Tree, each key is located in exactly one position. When a child node splits, the middle key is promoted to the parent and is then completely removed from the child level.

## The Engineering Trade-off:
The main benefit of using a pure B-Tree is seen in point lookups (search(key)); if a key that is frequently accessed is located near the root or in one of the upper internal pages, the query will match right away and will not need to retrieve the leaf pages from disk, thus ensuring that the best-case point lookups require fewer page reads. Moreover, since internal nodes do not contain duplicate keys, no space is wasted on repeated index values.

**The problem is with range queries.** In a B+ Tree, range searches are quick since all the leaves are linked together in a linear list; in a pure B-Tree, however, you can't simply scan across the leaves sideways, as you have to go up and down through the parent and child nodes in order, which introduces extra traversal overhead.

It was entirely reasonable to stick with a pure B-Tree since my main concern was improving the speed of point lookups and achieving the highest possible density of unique keys per page

## Mapping the Node Structure to a 4 KB Page:

Operating systems read from and write to the disk in fixed-size 4 KB pages. When you allocate tree nodes dynamically on the heap using dynamic memory pointers, you end up with fragmented memory and a great deal of page waste. In order to correct this, the node layout is directly mapped to a 4096-byte memory boundary by using mmap.

```cpp
constexpr int MAX_KEYS = 454; // Upper limit of keys to keep the struct under 4KB

struct BTreeNode {
    bool     is_leaf;               // 1 byte: true if it's a leaf node, false if internal
    uint16_t num_keys;              // 2 bytes: tracks current number of keys present in this node
    int32_t  keys[MAX_KEYS];        // 454 * 4 = 1816 bytes: stores the actual sorted key values
    bool     tombstone[MAX_KEYS];   // 454 * 1 = 454 bytes: soft-delete flag (true means logically deleted)
    uint32_t children[MAX_KEYS + 1];// 455 * 4 = 1820 bytes: stores child page IDs for navigation
};
static_assert(sizeof(BTreeNode) <= 4096, "the node layout exceeds the 4KB page size!");
```
***The Byte Math***
- 454 keys × 4 bytes = 1816 bytes
- 454 tombstones × 1 byte = 454 bytes
- 455 child page IDs × 4 bytes = 1820 bytes
- Metadata flags (is_leaf, num_keys) + compiler padding ≈ 6 bytes
  
***Total size: 4096 bytes***
Setting the value of MAX_KEYS to 454 fills the 4 KB page almost entirely. It would cause memory to be corrupted if an insertion attempted to add a 455th key without first splitting.

## Exact Page Splitting Mechanics:
B-Trees differ from ordinary binary search trees in that they grow upwards rather than downwards; when a page attains its limit of 454 keys, the split_child() function carries out a proactive split before the new record is inserted.

### What takes place in a split?

- **To find the median:** calculate the middle index, which is MAX_KEYS / 2 (index 227).
- **Promote the Median Key:** take the key located at mid and move it up to the parent page. The key will then be removed from the lower level.
- **Cut the left child:** the original full child retains the keys from index 0 up to mid - 1 (a total of 227 keys) and its num_keys counter is changed to 227.
- **Assign the appropriate sibling page:** a new page of 4 KB is allocated on the disk and the keys in the index from position mid + 1 to MAX_KEYS - 1 (a total of 226 keys) are copied into this new sibling page.

## Visualizing the Split:
To make this easy to trace, here is what happens during a split using a tiny node size (MAX_KEYS = 4):

<img width="1536" height="1024" alt="image1 (2)" src="https://github.com/user-attachments/assets/b449c624-8ee3-43fd-a9b8-9a756228ae18" />


```
SPLIT EXECUTION ON CHILD PAGE 1:
- mid = 4 / 2 = 2 (Key at index 2 is [30])
- Key [30] gets PROMOTED up into Parent Page 0
- Child Page 1 keeps keys before 30 -> [10, 20]
- New Page 3 takes keys after 30 -> [40]
```

## AFTER SPLIT:

<img width="1536" height="1024" alt="image2" src="https://github.com/user-attachments/assets/2068e962-f201-4dfb-82bf-ec5c56a0a54e" />

You can see that key 30 is now entirely contained within Parent Page 0 and is not present in Page 1 or Page 3. When a subsequent search for key 30 takes place, the traversal goes directly to Page 0 and returns at once without ever getting to the leaf pages. 







