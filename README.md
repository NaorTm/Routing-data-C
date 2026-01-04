# Routing Data Simulator (C)

Simulates a simple switch/router with 4 input ports. Packets are read from files, routed via a binary search tree table, queued by priority, sorted by time, and written to output port files.

## Features
- BST-based routing table (add/delete/search)
- Packet checksum validation
- Two priority queues per output port
- Deterministic output files (`port1.out` ... `port4.out`)

## Build and Run
Compile:

```sh
gcc "main prog.c" prog.c -o router
```

Run:

```sh
router.exe in1.in in2.in in3.in in4.in route.txt
```

Input files live in the `files/` folder.
