# Function & Architecture Documentation

This document provides an in-depth description of every function and data structure in the Routing Data Simulator.  
For a high-level overview and usage instructions, see [README.md](README.md).

---

## Table of Contents

1. [Header – `header.h`](#header--headerh)
   - [Macros](#macros)
   - [Structs & Typedefs](#structs--typedefs)
2. [Packet Functions (`prog.c`)](#packet-functions-progc)
   - [`packet_read`](#packet_read)
   - [`packet_write`](#packet_write)
   - [`checksum_check`](#checksum_check)
3. [BST Routing-Table Functions (`prog.c`)](#bst-routing-table-functions-progc)
   - [`create_node`](#create_node)
   - [`addItemToTreeRec`](#additemtotreerec)
   - [`add_route`](#add_route)
   - [`LowestDa`](#lowestda)
   - [`delete_route`](#delete_route)
   - [`search_route`](#search_route)
   - [`print_routing_table`](#print_routing_table)
   - [`build_route_table`](#build_route_table)
   - [`free_tree`](#free_tree)
4. [Queue / List Functions (`prog.c`)](#queue--list-functions-progc)
   - [`add_last`](#add_last)
   - [`enque_pkt`](#enque_pkt)
   - [`deque_pkt`](#deque_pkt)
   - [`sort_list`](#sort_list)
5. [Entry Point (`main prog.c`)](#entry-point-main-progc)
   - [`main`](#main)

---

## Header – `header.h`

### Macros

```c
#define OUTP1 "port1.out"
#define OUTP2 "port2.out"
#define OUTP3 "port3.out"
#define OUTP4 "port4.out"
```

String constants for the four output file names. Used in `main` to open output files without hard-coding the names inline.

---

### Structs & Typedefs

#### `packet`

```c
typedef struct packet {
    unsigned int   time;         // arrival timestamp
    unsigned char  Da;           // destination address
    char           Sa;           // source address
    char           Prio;         // priority: 0 = high, 1 = low
    char           Data_length;  // number of payload bytes
    unsigned char *data;         // dynamically allocated payload
    unsigned char  Checksum;     // XOR integrity checksum
} packet;
```

Represents one network packet.  The `data` field is heap-allocated by `packet_read` with `malloc(Data_length)` and must be freed when the packet is discarded or after it is written to the output file.

---

#### `S_node` / `route_node`

```c
typedef struct route_node {
    unsigned char  da;           // destination address (BST key)
    char           output_port;  // output port number (1–4)
    struct route_node *left;     // left child  (da values ≤ current)
    struct route_node *right;    // right child (da values >  current)
} S_node;
```

A single node in the Binary Search Tree (BST) routing table.  The tree is ordered by `da`; smaller or equal values go left, larger values go right.

---

#### `BST`

```c
typedef struct {
    S_node *root;
} BST;
```

Thin wrapper holding the root pointer of the routing BST.  Passed around by value in `main`; the root pointer is updated whenever `add_route` or `delete_route` is called.

---

#### `S_pkt` / `pkt_node`

```c
typedef struct pkt_node {
    packet          *pkt;   // pointer to the wrapped packet
    struct pkt_node *next;  // next node in the linked list
} S_pkt;
```

A singly-linked-list node that wraps a `packet *`.  Chains of `S_pkt` nodes form the priority sub-queues inside `S_Out_Qs_mgr`.

---

#### `S_Out_Qs_mgr` / `Out_Qs_mgr`

```c
typedef struct Out_Qs_mgr {
    struct pkt_node *head_p1, *tail_p1;  // priority-1 (low)  queue
    struct pkt_node *head_p0, *tail_p0;  // priority-0 (high) queue
} S_Out_Qs_mgr;
```

Manages **two** FIFO sub-queues for one output port:

| Sub-queue | Priority value | Written to output file |
|-----------|---------------|------------------------|
| `head_p0` / `tail_p0` | `0` (high) | First  |
| `head_p1` / `tail_p1` | `1` (low)  | Second |

All four head/tail pointers start as `NULL` (initialised by `calloc` in `main`).

---

#### `Bool` enum

```c
typedef enum Bool { FALSE, TRUE } Bool;
```

Simple boolean type.  Used as the return type of `checksum_check`.

---

## Packet Functions (`prog.c`)

### `packet_read`

```c
void packet_read(FILE *fp, packet *pkt);
```

**Purpose:** Read one packet from an already-open text file into the `pkt` struct.

**How it works:**

1. Peeks at the next two characters. If it finds a newline immediately followed by EOF the file is exhausted; `pkt` is set to `NULL` and the function returns early.
2. Seeks back one byte so the real data is not consumed by the peek.
3. Reads the five header fields in order using `fscanf`: `time`, `Da`, `Sa`, `Prio`, `Data_length`.
4. Allocates `Data_length` bytes on the heap for `pkt->data` (asserts success).
5. Reads each payload byte into `pkt->data[i]` in a loop.
6. Reads the trailing `Checksum` field.

**Parameters:**

| Name | Description |
|------|-------------|
| `fp` | Open file pointer positioned at the start of a packet line |
| `pkt`| Caller-allocated `packet` struct to fill in |

**Return value:** None (void).  Sets `pkt` to `NULL` internally on EOF, but because the pointer is passed by value the caller detects EOF by checking `pkt->data == NULL` after the call.

**Notes:**
- The file must already be open.
- Memory for `pkt->data` is owned by the caller after this call and must be freed with `free(pkt->data)`.

---

### `packet_write`

```c
void packet_write(FILE *fp, const packet *pkt);
```

**Purpose:** Write one packet to an already-open text file in the same format as the input files.

**How it works:**

Prints every field space-separated using `fprintf`, and ends the line with `\n`:

```
<time> <Da> <Sa> <Prio> <Data_length> <data[0]> ... <data[N-1]> <Checksum>\n
```

**Parameters:**

| Name | Description |
|------|-------------|
| `fp` | Open, writable file pointer |
| `pkt`| Packet to write (read-only) |

**Return value:** None (void).

---

### `checksum_check`

```c
bool checksum_check(const packet *pkt);
```

**Purpose:** Verify the integrity of a packet by re-computing its checksum and comparing against the stored value.

**How it works:**

XORs together the four header bytes and every payload byte:

```
result = Da ^ Sa ^ Prio ^ Data_length ^ data[0] ^ data[1] ^ ... ^ data[N-1]
```

Returns `TRUE` if `result == pkt->Checksum`, `FALSE` otherwise.

**Parameters:**

| Name | Description |
|------|-------------|
| `pkt`| Packet to validate (read-only) |

**Return value:** `TRUE` (1) if valid, `FALSE` (0) if corrupted.

**Notes:** Packets that fail this check are dropped (freed) in `enque_pkt` without being queued.

---

## BST Routing-Table Functions (`prog.c`)

### `create_node`

```c
S_node *create_node(unsigned char da, char Pout);
```

**Purpose:** Allocate and initialise a new leaf node for the BST.

**How it works:**

1. `calloc`s one `S_node` (all bytes zeroed).
2. Sets `da`, `output_port` (`Pout`), and both child pointers to `NULL`.
3. Returns the pointer.

**Parameters:**

| Name   | Description |
|--------|-------------|
| `da`   | Destination address to store in this node |
| `Pout` | Output port number associated with this address |

**Return value:** Pointer to the newly created node.

---

### `addItemToTreeRec`

```c
void addItemToTreeRec(S_node *root, unsigned char da, char output_port);
```

**Purpose:** Recursively insert a new `(da, output_port)` pair into a non-empty BST.

**How it works:**

- If `da ≤ root->da`, traverse left:
  - If the left child is `NULL`, create a new leaf there.
  - Otherwise recurse into the left sub-tree.
- If `da > root->da`, traverse right similarly.

This is a standard recursive BST insertion.

**Parameters:**

| Name          | Description |
|---------------|-------------|
| `root`        | Current subtree root (never `NULL`) |
| `da`          | Destination address to insert |
| `output_port` | Output port for this address |

**Return value:** None (void); the tree is modified in place.

---

### `add_route`

```c
S_node *add_route(S_node *root, char da, char output_port);
```

**Purpose:** Public entry point for BST insertion; handles the empty-tree case.

**How it works:**

- If `root == NULL` (empty tree), calls `create_node` to make the first node.
- Otherwise delegates to `addItemToTreeRec`.
- Returns the (possibly new) root.

**Parameters:**

| Name          | Description |
|---------------|-------------|
| `root`        | Current root of the BST (may be `NULL`) |
| `da`          | Destination address to add |
| `output_port` | Matching output port |

**Return value:** Updated root pointer (important: may differ from input when tree was empty).

---

### `LowestDa`

```c
S_node *LowestDa(S_node *node);
```

**Purpose:** Find the node with the **smallest** `da` value in a sub-tree (its in-order successor).

**How it works:**

Walks down the left spine of the sub-tree until a node with no left child is found, then returns it.

**Parameters:**

| Name   | Description |
|--------|-------------|
| `node` | Root of the sub-tree to search |

**Return value:** Pointer to the leftmost (minimum `da`) node.

**Notes:** Used exclusively by `delete_route` to find the in-order successor when deleting a node with two children.

---

### `delete_route`

```c
S_node *delete_route(S_node *root, char da);
```

**Purpose:** Remove the routing entry for destination address `da` from the BST.

**How it works (standard BST deletion):**

1. If `root == NULL`, the address was not found; return `NULL`.
2. If `da < root->da`, recurse into the left subtree.
3. If `da > root->da`, recurse into the right subtree.
4. If `da == root->da` (found):
   - **No left child:** replace node with its right child, free the node.
   - **No right child:** replace node with its left child, free the node.
   - **Two children:** find the in-order successor (`LowestDa(root->right)`), copy its `da` and `output_port` into the current node, then recursively delete the successor from the right subtree.

**Parameters:**

| Name   | Description |
|--------|-------------|
| `root` | Current subtree root |
| `da`   | Destination address to remove |

**Return value:** Updated root pointer for the sub-tree.

---

### `search_route`

```c
S_node *search_route(const S_node *root, char da);
```

**Purpose:** Find the BST node whose `da` matches the given destination address.

**How it works:**

Performs a **full recursive tree traversal** (not a standard BST O(log n) search): checks if the current node matches, then recursively searches the left subtree, and if not found there, searches the right subtree.

**Parameters:**

| Name   | Description |
|--------|-------------|
| `root` | Current subtree root |
| `da`   | Destination address to search for |

**Return value:** Pointer to the matching `S_node`, or `NULL` if not found.

**Notes:** Because the search checks both subtrees rather than using the BST ordering property, it runs in **O(n)** worst-case time. This is safe but less efficient than a standard BST lookup.

---

### `print_routing_table`

```c
void print_routing_table(const S_node *root);
```

**Purpose:** Print all destination addresses in the BST to standard output in ascending order (in-order traversal).

**How it works:**

Standard recursive in-order traversal: left → root → right. Prints `root->da` (as a decimal integer) followed by a space.

**Parameters:**

| Name   | Description |
|--------|-------------|
| `root` | Root of the BST to print |

**Return value:** None (void).

**Notes:** Called by `build_route_table` after the full routing table has been loaded to provide a diagnostic view of the tree.

---

### `build_route_table`

```c
S_node *build_route_table(FILE *fp, S_node *root);
```

**Purpose:** Parse the entire `route.txt` file and build the final BST routing table.

**How it works:**

1. Seeks to the beginning of the file.
2. Reads one character at a time:
   - `'a'` → reads `da` and `output_port`, calls `add_route`.
   - `'d'` → reads `da` only, calls `delete_route`.
   - Anything else → stops (treats as end of commands).
3. After each command, seeks 2 bytes forward (skipping the newline / separator).
4. Prints the final BST via `print_routing_table`.
5. Returns the updated root.

**Parameters:**

| Name   | Description |
|--------|-------------|
| `fp`   | Open file pointer to `route.txt` |
| `root` | Initial BST root (usually `NULL`) |

**Return value:** Root pointer of the fully-built BST.

---

### `free_tree`

```c
void free_tree(S_node *node);
```

**Purpose:** Recursively free all nodes in a BST sub-tree.

**How it works:**

Post-order traversal (right → left → node): frees right sub-tree, then left sub-tree, then the node itself, so no pointer is used after it is freed.

**Parameters:**

| Name   | Description |
|--------|-------------|
| `node` | Root of the sub-tree to free |

**Return value:** None (void).

---

## Queue / List Functions (`prog.c`)

### `add_last`

```c
S_Out_Qs_mgr *add_last(S_pkt *packet_node, S_Out_Qs_mgr *List);
```

**Purpose:** Append a packet node to the tail of the correct priority sub-queue.

**How it works:**

Checks `packet_node->pkt->Prio`:

- **`Prio == 0` (high):** appends to the priority-0 sub-queue (`head_p0` / `tail_p0`).
- **`Prio != 0` (low):** appends to the priority-1 sub-queue (`head_p1` / `tail_p1`).

In both cases, if the relevant head pointer is `NULL` the new node becomes both head and tail; otherwise it is linked onto the current tail.

**Parameters:**

| Name          | Description |
|---------------|-------------|
| `packet_node` | Already-allocated `S_pkt` node to append |
| `List`        | Queue manager for the target output port |

**Return value:** The same `List` pointer (no allocation, just modification in place).

---

### `enque_pkt`

```c
void enque_pkt(S_Out_Qs_mgr *QM_ptr, packet *pkt);
```

**Purpose:** Validate a packet and, if valid, wrap it in an `S_pkt` node and add it to the appropriate queue.

**How it works:**

1. If `pkt == NULL`, returns immediately.
2. Calls `checksum_check(pkt)`.
   - **Valid:** allocates an `S_pkt` node, sets `pkt` and `next = NULL`, calls `add_last`.
   - **Invalid:** frees `pkt->data` and `pkt` (drops the packet).

**Parameters:**

| Name     | Description |
|----------|-------------|
| `QM_ptr` | Queue manager for the output port this packet is destined for |
| `pkt`    | Packet to enqueue (heap-allocated by the caller) |

**Return value:** None (void).

---

### `deque_pkt`

```c
packet *deque_pkt(S_Out_Qs_mgr *QM_ptr, char priority);
```

**Purpose:** Remove and return the front packet from the specified priority sub-queue (FIFO).

**How it works:**

Based on `priority`:

- **`0`:** checks `head_p0`; if non-NULL, takes its `pkt` pointer, advances `head_p0` to `head_p0->next`, returns the packet.
- **non-0:** same logic for `head_p1`.

If the requested sub-queue is empty, prints `"\nList is Empty ..."` and returns `NULL`.

**Parameters:**

| Name       | Description |
|------------|-------------|
| `QM_ptr`   | Queue manager to dequeue from |
| `priority` | Sub-queue to dequeue from (`0` = high, `1` = low) |

**Return value:** Pointer to the dequeued `packet`, or `NULL` if the queue was empty.

**Notes:** The `S_pkt` wrapper node is not freed here — only the contained `packet *` is returned. Because `head` is advanced but the old node is never freed, this implementation has a **minor memory leak** on the `S_pkt` nodes.

---

### `sort_list`

```c
void sort_list(S_pkt *head);
```

**Purpose:** Sort a linked-list queue in ascending order of packet arrival time (`pkt->time`).

**How it works:**

Bubble sort over the linked list: repeatedly traverses the list swapping adjacent `pkt` pointers (not node pointers) whenever `location->pkt->time > next->pkt->time`.  Uses the `changed` flag to detect when a full pass completes without any swap (early exit).

**Parameters:**

| Name   | Description |
|--------|-------------|
| `head` | Head node of the linked list to sort (must not be `NULL` to enter the sort) |

**Return value:** None (void); the list is sorted in place by swapping `pkt` pointers inside the existing nodes.

**Time complexity:** O(n²) worst-case (bubble sort).

---

## Entry Point (`main prog.c`)

### `main`

```c
int main(int argc, char *argv[]);
```

**Purpose:** Orchestrate the full simulation: load the routing table, read all input packets, route and queue them, sort the queues, and write output files.

**Command-line arguments:**

```
argv[1]  path to port 1 input file (e.g. files/port1.in)
argv[2]  path to port 2 input file
argv[3]  path to port 3 input file
argv[4]  path to port 4 input file
argv[5]  path to routing table file (e.g. files/route.txt)
```

**Step-by-step behaviour:**

1. **Load routing table**
   - Opens `argv[5]` (`route.txt`).
   - Calls `build_route_table` to populate the BST.
   - Closes the file.

2. **Open input port files**
   - Opens `argv[1]` through `argv[4]` in read mode.
   - All four files must exist (asserted).

3. **Allocate output queue managers**
   - `calloc`s one `S_Out_Qs_mgr` per output port (4 total).
   - All head/tail pointers start as `NULL`.

4. **Read, route, and queue packets**
   - Iterates over all four input files.
   - For each file, reads packets in a loop until EOF (`feof`).
   - Each packet is `calloc`d, filled by `packet_read`, then processed:
     - If `pkt->data == NULL` (EOF sentinel): frees the empty struct.
     - Otherwise: looks up `pkt->Da` in the BST with `search_route`.
       - **Route found:** calls `enque_pkt` on the matching `lists[port-1]`.
       - **No route:** drops the packet (`free(pkt->data)`, `free(pkt)`).
   - Closes each input file after it is fully read.

5. **Sort all queues**
   - Calls `sort_list` on both sub-queues of each of the four output-port managers (8 sort calls total).

6. **Open output files**
   - Creates/truncates `port1.out` through `port4.out` using the `OUTP1`–`OUTP4` macros.

7. **Write output files**
   - For each output port, drains the priority-0 queue first (high priority), then the priority-1 queue.
   - Each packet is written by `packet_write`, then its `data` and the struct itself are freed.

8. **Clean up**
   - Closes all output files.
   - Frees the four `S_Out_Qs_mgr` allocations.
   - Calls `free_tree` to release the entire BST.
