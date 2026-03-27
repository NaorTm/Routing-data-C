# Routing Data Simulator (C)

A C program that simulates a simple network switch/router with **4 input ports**. Packets are read from plain-text files, routed through a Binary Search Tree (BST) routing table, queued by priority into per-port linked-list queues, sorted by arrival time, and finally written to output port files.

---

## Table of Contents

1. [Overview](#overview)
2. [Project Structure](#project-structure)
3. [Data Structures](#data-structures)
4. [Input File Formats](#input-file-formats)
   - [Packet Input Files](#packet-input-files-port1in--port4in)
   - [Routing Table File](#routing-table-file-routetxt)
5. [Processing Rules](#processing-rules)
6. [Output Files](#output-files)
7. [Build and Run](#build-and-run)
8. [Example Walkthrough](#example-walkthrough)

---

## Overview

The simulator models the forwarding plane of a network switch:

1. A **routing table** is loaded from `route.txt` and stored as a BST keyed by destination address.
2. Packets are read one-by-one from four input-port files.
3. Each packet's destination address is looked up in the BST to find the correct output port.
4. Valid packets (passing checksum) are enqueued into the matching output-port queue, separated into **priority 0** (high) and **priority 1** (low) sub-queues.
5. Each sub-queue is sorted by arrival timestamp.
6. All queues are drained to four output files: priority-0 packets first, then priority-1 packets.
7. Packets that fail the checksum check or have no matching route are silently dropped.

---

## Project Structure

```
Routing-data-C/
├── header.h          # All type definitions and function prototypes
├── prog.c            # Implementation of all helper functions
├── main prog.c       # Entry point – orchestrates the full simulation
└── files/
    ├── port1.in      # Packet input for port 1
    ├── port2.in      # Packet input for port 2
    ├── port3.in      # Packet input for port 3
    ├── port4.in      # Packet input for port 4
    └── route.txt     # Routing table commands
```

---

## Data Structures

### `packet`
Represents a single network packet read from an input file.

| Field         | Type            | Description                                      |
|---------------|-----------------|--------------------------------------------------|
| `time`        | `unsigned int`  | Arrival timestamp (used for ordering)            |
| `Da`          | `unsigned char` | Destination address                              |
| `Sa`          | `char`          | Source address                                   |
| `Prio`        | `char`          | Priority: `0` = high priority, `1` = low priority|
| `Data_length` | `char`          | Number of bytes in the data payload              |
| `data`        | `unsigned char*`| Dynamically allocated payload bytes              |
| `Checksum`    | `unsigned char` | XOR checksum for integrity validation            |

### `S_node` (BST routing node)
One node in the Binary Search Tree routing table.

| Field         | Type            | Description                             |
|---------------|-----------------|-----------------------------------------|
| `da`          | `unsigned char` | Destination address (BST key)           |
| `output_port` | `char`          | Output port number (1–4) for this route |
| `left`        | `S_node *`      | Left child (smaller `da` values)        |
| `right`       | `S_node *`      | Right child (larger `da` values)        |

### `BST`
Thin wrapper that holds the `root` pointer of the routing-table BST.

### `S_pkt` (packet linked-list node)
Wraps a `packet *` with a `next` pointer to form a singly-linked list queue.

### `S_Out_Qs_mgr` (output queue manager)
Manages **two** priority sub-queues for a single output port.

| Field      | Description                            |
|------------|----------------------------------------|
| `head_p1` / `tail_p1` | Head and tail of priority-1 (low) queue  |
| `head_p0` / `tail_p0` | Head and tail of priority-0 (high) queue |

---

## Input File Formats

### Packet Input Files (`port1.in` – `port4.in`)

Each line describes one packet. Fields are space-separated integers:

```
<time> <Da> <Sa> <Prio> <Data_length> <data_byte_0> ... <data_byte_N-1> <Checksum>
```

| Field           | Description                                               |
|-----------------|-----------------------------------------------------------|
| `time`          | Unsigned integer arrival timestamp                        |
| `Da`            | Destination address (0–255)                               |
| `Sa`            | Source address                                            |
| `Prio`          | `0` = high priority (written first), `1` = low priority  |
| `Data_length`   | Number of data bytes that follow (`N`)                    |
| `data_byte_0..N-1` | Exactly `Data_length` unsigned byte values            |
| `Checksum`      | Single byte; must equal `Da ^ Sa ^ Prio ^ Data_length ^ data[0] ^ ... ^ data[N-1]` |

**Example line:**
```
1 128 55 1 4 1 2 3 4 182
```
- Arrives at time `1`, destination address `128`, source `55`, priority `1` (low), 4 data bytes `[1,2,3,4]`, checksum `182`.

### Routing Table File (`route.txt`)

Each line is a routing command:

| Command      | Syntax            | Effect                                              |
|--------------|-------------------|-----------------------------------------------------|
| Add route    | `a <da> <port>`   | Insert destination address `da` → output port `port` into the BST |
| Delete route | `d <da>`          | Remove the entry for destination address `da` from the BST |

**Example:**
```
a 128 4    ← route packets with Da=128 to output port 4
a 33  1    ← route packets with Da=33  to output port 1
d 1        ← delete the route for Da=1
a 1  4     ← re-add Da=1 pointing to port 4
```

Routes are processed sequentially from top to bottom; later commands override earlier ones for the same address.

---

## Processing Rules

1. **Build routing table** – the entire `route.txt` file is processed first to produce the final BST state.
2. **Read packets** – each of the four input port files is read in order (port 1 → port 4). Within one file, packets are read until EOF.
3. **Route lookup** – the BST is searched for the packet's `Da`. If no matching node exists the packet is **dropped** (freed).
4. **Checksum validation** – before enqueuing, `checksum_check` verifies integrity. A failing packet is **dropped**.
5. **Enqueue** – a valid packet is appended to the tail of the appropriate priority sub-queue of the resolved output port.
6. **Sort** – after all packets are read, each priority sub-queue is sorted by `time` using bubble sort so packets are emitted in arrival order.
7. **Output** – for each output port (1–4), priority-0 packets are written first (in time order), followed by priority-1 packets (in time order).
8. **Output file names** are fixed: `port1.out`, `port2.out`, `port3.out`, `port4.out`.

---

## Output Files

Each output file (`port1.out` – `port4.out`) contains the forwarded packets for that port in the same text format as the input files:

```
<time> <Da> <Sa> <Prio> <Data_length> <data bytes...> <Checksum>
```

Priority-0 (high-priority) packets appear first, sorted by `time`. Priority-1 packets follow, also sorted by `time`.

---

## Build and Run

**Requirements:** a C99-compatible compiler (GCC, Clang, or MSVC).

**Compile with GCC:**
```sh
gcc "main prog.c" prog.c -o router
```

**Run:**
```sh
./router <port1.in> <port2.in> <port3.in> <port4.in> <route.txt>
```

Using the provided sample files:
```sh
./router files/port1.in files/port2.in files/port3.in files/port4.in files/route.txt
```

Output files are written to the **current working directory**: `port1.out`, `port2.out`, `port3.out`, `port4.out`.

> **Windows note:** The binary is `router.exe` and the MSVC macro `_CRT_SECURE_NO_WARNINGS` is defined in the source to suppress deprecation warnings for standard I/O functions.

---

## Example Walkthrough

Given the first line of `port1.in`:
```
1 128 55 1 4 1 2 3 4 182
```
And `route.txt` contains `a 128 4` (among others), the simulation will:

1. Build the BST; destination address `128` maps to output port `4`.
2. Read the packet: time=1, Da=128, Sa=55, Prio=1, Data=[1,2,3,4], Checksum=182.
3. Verify checksum: `128 ^ 55 ^ 1 ^ 4 ^ 1 ^ 2 ^ 3 ^ 4 = 182` ✓
4. Look up Da=128 → output port 4.
5. Enqueue into `lists[3]` (port 4), priority-1 sub-queue.
6. After all packets are processed, sort the priority-1 sub-queue of port 4 by time.
7. Write the packet to `port4.out` after all priority-0 packets for port 4.
