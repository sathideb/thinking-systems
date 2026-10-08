# Tombstone Soft Deletion and `vacuum()` in VeloDB

## The problem I'm solving

In a textbook B-Tree, deleting a key can mean **borrowing from a sibling**, **merging nodes**, or **rewriting parents** all the way up. That is slow and needs a lot of locking.

Here I  take a lazier approach(Tombstone): **don't remove the key, just mark it dead.** Clean up later.

## How a node looks

Each 4KB node (**BTreeNode**) has a **tombstone** array that maps 1-to-1 onto `keys`:

```
Index:      [ 0     | 1    | 2     | 3     ]
Keys:       [ 10    | 20   | 30    | 40    ]
Tombstone:  [ false | true | false | false ]
```

- **`tombstone[i] == false`**: the key is alive.
- **`tombstone[i] == true`**: the key is logically deleted but still physically in its slot.

Here, key **20** is soft-deleted.

## What each operation does

### Delete(20)

Binary search (`std::lower_bound`) finds index 1, then I flip `tombstone[1] = true`.

- **Time:** O(log N) to find, O(1) to flip
- **No shifting, no page changes** during live traffic

### Search(20) and Search(30)

`lower_bound` still finds index 1 for 20, but I return `!tombstone[1]`, which is `false`, so the user sees **not found**. Search(30) returns `true`.

**Why keep the dead key in the array?** Binary search and subtree routing need the sorted order intact. A dead 20 still sits correctly between 10 and 30.

### Re-insert(20) after deleting it (`revive_if_present`)

Without a check, I would insert a second 20:

```
Keys:       [ 10 | 20 | 20 | 30 ]
Tombstone:  [ F  | T  | F  | F  ]
```

`lower_bound` always lands on the **first** 20 (the tombstoned one), so search says "not found" even though a live 20 exists. The new copy is unreachable.

So instead, I check if 20 is already present as a tombstone and flip it back to `false`. **One in-place write, no duplicate.**

## Why `vacuum()` is needed

Dead keys waste space. Say a leaf holds 4 keys max:

```
Keys: [ 10, 20(T), 30, 40(T) ]   num_keys = 4  -> FULL
```

If I insert 25, the leaf thinks it is full and **splits**, even though half of it is garbage. That is a wasted page split.

## How `vacuum()` works on leaves

It is a two-pointer pass:

- **`read_idx`** scans every slot
- **`write_idx`** marks where the next live key goes

Whenever `read_idx` sees a live key, copy it to `write_idx` and advance both. Dead keys are skipped. Then clear the trailing slots and reduce `num_keys`.

```
Before: [ 10, 20(T), 30, 40(T) ]   num_keys = 4
After:  [ 10, 30, --, -- ]         num_keys = 2
```

Now inserting 25 gives `[10, 25, 30, --]`. It fits in the same page, **no split needed**.

## Why internal nodes are NOT vacuumed

In internal nodes, keys are **signposts for routing**, not just data. An internal node with **N keys must keep N+1 child pointers**.

```
        [ 50 (T) ]
        /        \
   Page 2        Page 3
  [10, 25]      [60, 80]
  (keys < 50)   (keys >= 50)
```

Suppose the user deletes key 50, so it is tombstoned. If `vacuum()` removed it, the node would have **0 keys but still 2 children**. Nothing would say where Page 2 ends and Page 3 begins, and one subtree would become unreachable. That is **live data lost**.

So 50 stays. It still does its routing job, and `search(50)` returns "not found" because of the tombstone flag.

**Note: The design and implementation details documented here may evolve in future iterations as I explore and benchmark better approaches, but the core architectural concepts and trade-offs remain the same.**
