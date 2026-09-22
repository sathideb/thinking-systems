# B+ Tree Core System Mechanics

# 1.Why We Use B+ Trees:

In-memory data structures such as binary search trees (BSTs) are unable to exploit the favorable characteristics of disk-backed storage engines because their implementation details fundamentally violate physical hardware constraints like

- Disk Reads Are Page Based: Solid state drives (SSDs), NVMe drives, and the OS kernel do not read individual bytes from disk, but instead in units of pages (e.g. 4KB, 8KB, or 16KB).

- The BST Problem (Low Fan-Out): A given BST node only contains 1 key and 2 pointers (for a total of ~24-32B). Loading a single BST node into memory involves reading an entire 4KB page from disk – a waste of more than 99% I/O bandwidth. Due to BST’s random heap allocations, following a parent/child pointer results in random I/O accesses.
- The B+ Tree Solution (High Fan-Out): A B+ tree node stores hundreds of sorted keys and child pointers sequentially in a contiguous memory block (a.k.a. fat nodes). By aligning the size of tree nodes to the physical page size, the tree height is kept extremely shallow (3–4 levels for a 1M row table), providing $O(\log_B N)$ lookups within a 3–4 page range

<img width="2170" height="725" alt="image1" src="https://github.com/user-attachments/assets/948ab8cf-4642-405c-ac37-eae351626eaf" />

# 2.Node Types: Data Capacity and Splitting Rules
A B+ Tree utilizes two distinctly separated node types to store information with different data capacities and splitting rules.

- Leaf Node (External)Contents:Actual data records or the tuple attributes with pointers to the sibling leaf pages.
- Data Capacity:Compared to internal nodes, leaf nodes hold fewer entries. Storing entire row attributes leads to a significant increase in the payload size, so that the number of records stored on a single page is limited.
- Split Key Rule (Copy-Up):When a leaf node splits, the separator key is retained in the right leaf node and copied up to the parent.
- Reason for Copy-Up:In leaf nodes, actual data is physically stored in the tree. Consequently, deleting a key from a leaf node would result in the removal of a record from the database.

- Internal Node (Navigator)Contents:Set of compact separator keys and child page IDs ($P_0, K_1, P_1, \dots, K_n, P_n$).
- Data Capacity:Internal nodes possess a greater data capacity as internal node entries do not contain any payload. Therefore, a single page can hold hundreds of compact keys and page IDs.

- Split Key Rule (Push-Up):While splitting internal nodes, the median key is pushed up to the parent node, while it is deleted from the lower internal nodes.

- Reason for Push-Up:Internal keys only serve a navigational purpose, so there is no need to store duplicate keys in the lower internal nodes as well. This would unnecessarily waste the storage space of pages
