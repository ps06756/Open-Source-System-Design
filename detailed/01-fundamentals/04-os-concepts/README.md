# Operating System Concepts

Understanding how operating systems manage processes, threads, memory, and I/O is crucial for building high-performance systems and making informed design decisions.

## Table of Contents
1. [Processes and Threads](#processes-and-threads)
2. [Concurrency Models](#concurrency-models)
3. [I/O Models](#io-models)
4. [Memory Management](#memory-management)
5. [System Design Implications](#system-design-implications)
6. [Interview Questions](#interview-questions)

---

## Processes and Threads

### Process

A process is an instance of a running program with its own memory space.

```
┌─────────────────────────────────────────────────────────────┐
│                      Process Memory Layout                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  High Address                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                     Stack                               │ │
│  │              (local variables, function calls)          │ │
│  │                         ↓                               │ │
│  ├────────────────────────────────────────────────────────┤ │
│  │                                                         │ │
│  │                    (free space)                         │ │
│  │                                                         │ │
│  ├────────────────────────────────────────────────────────┤ │
│  │                         ↑                               │ │
│  │                       Heap                              │ │
│  │              (dynamic memory allocation)                │ │
│  ├────────────────────────────────────────────────────────┤ │
│  │                       BSS                               │ │
│  │           (uninitialized global variables)              │ │
│  ├────────────────────────────────────────────────────────┤ │
│  │                       Data                              │ │
│  │            (initialized global variables)               │ │
│  ├────────────────────────────────────────────────────────┤ │
│  │                       Text                              │ │
│  │                  (program code)                         │ │
│  └────────────────────────────────────────────────────────┘ │
│  Low Address                                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Thread

A thread is a lightweight unit of execution within a process. Threads share the process's memory space.

```
┌─────────────────────────────────────────────────────────────┐
│                  Process with Multiple Threads               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                        Process                           ││
│  │  ┌───────────────────────────────────────────────────┐  ││
│  │  │              Shared Resources                      │  ││
│  │  │  • Heap memory                                     │  ││
│  │  │  • Global variables                                │  ││
│  │  │  • File descriptors                                │  ││
│  │  │  • Code (text segment)                             │  ││
│  │  └───────────────────────────────────────────────────┘  ││
│  │                                                          ││
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐             ││
│  │  │ Thread 1│    │ Thread 2│    │ Thread 3│             ││
│  │  │         │    │         │    │         │             ││
│  │  │ Stack   │    │ Stack   │    │ Stack   │             ││
│  │  │ Regs    │    │ Regs    │    │ Regs    │             ││
│  │  │ PC      │    │ PC      │    │ PC      │             ││
│  │  └─────────┘    └─────────┘    └─────────┘             ││
│  │                                                          ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘

Each thread has its own:
• Stack (local variables)
• Registers
• Program Counter (PC)
```

### Process vs Thread Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Process vs Thread                                   │
├─────────────────────────────────┬───────────────────────────────────────┤
│           Process               │              Thread                    │
├─────────────────────────────────┼───────────────────────────────────────┤
│ Own memory space                │ Shared memory space                    │
│ Heavy context switch            │ Light context switch                   │
│ Process crash is isolated       │ Thread crash can crash process         │
│ Slower creation                 │ Faster creation                        │
│ IPC for communication           │ Shared memory for communication        │
│ More secure (isolation)         │ Less secure (shared memory)            │
├─────────────────────────────────┼───────────────────────────────────────┤
│ Use: Isolation needed           │ Use: Performance, shared state         │
│ Example: Chrome tabs            │ Example: Web server request handlers   │
└─────────────────────────────────┴───────────────────────────────────────┘
```

### Context Switching

```
┌─────────────────────────────────────────────────────────────┐
│                    Context Switch                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Process A                                   Process B       │
│    │                                            │            │
│    │  Running                                   │            │
│    ▼                                            │            │
│  ┌──────────────────┐                           │            │
│  │ 1. Save State A  │                           │            │
│  │    - Registers   │                           │            │
│  │    - PC          │                           │            │
│  │    - Stack ptr   │                           │            │
│  └────────┬─────────┘                           │            │
│           │                                     │            │
│           ▼                                     │            │
│  ┌──────────────────┐                           │            │
│  │ 2. Switch to     │                           │            │
│  │    Kernel mode   │                           │            │
│  └────────┬─────────┘                           │            │
│           │                                     │            │
│           ▼                                     │            │
│  ┌──────────────────┐                           │            │
│  │ 3. Load State B  │                           │            │
│  │    - Registers   │                           │            │
│  │    - PC          │                           │            │
│  │    - Stack ptr   │──────────────────────────▶│            │
│  └──────────────────┘                           ▼            │
│                                              Running         │
│                                                              │
│  Cost: ~1-10 microseconds                                    │
│  Thread switch is ~10x faster than process switch            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Concurrency Models

### Multi-Process Model

```
┌─────────────────────────────────────────────────────────────┐
│                   Multi-Process Model                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Master Process                      │  │
│  │                 (accepts connections)                  │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │ fork()                            │
│            ┌─────────────┼─────────────┐                    │
│            ▼             ▼             ▼                    │
│       ┌─────────┐   ┌─────────┐   ┌─────────┐              │
│       │Worker 1 │   │Worker 2 │   │Worker 3 │              │
│       │ (PID 1) │   │ (PID 2) │   │ (PID 3) │              │
│       └─────────┘   └─────────┘   └─────────┘              │
│                                                              │
│  Pros:                                                       │
│  ✓ Process isolation (crash doesn't affect others)          │
│  ✓ Can use multiple CPU cores                               │
│  ✓ Simple programming model                                 │
│                                                              │
│  Cons:                                                       │
│  ✗ High memory usage (each process has own memory)          │
│  ✗ Expensive context switches                               │
│  ✗ Complex IPC needed for communication                     │
│                                                              │
│  Example: Apache (prefork), PHP-FPM                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Multi-Thread Model

```
┌─────────────────────────────────────────────────────────────┐
│                   Multi-Thread Model                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                     Process                            │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │                 Shared Memory                    │  │  │
│  │  │          (connection pool, cache, etc.)          │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │                                                        │  │
│  │    ┌─────────┐   ┌─────────┐   ┌─────────┐           │  │
│  │    │Thread 1 │   │Thread 2 │   │Thread 3 │           │  │
│  │    └─────────┘   └─────────┘   └─────────┘           │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  Pros:                                                       │
│  ✓ Shared memory (efficient communication)                  │
│  ✓ Lower memory footprint                                   │
│  ✓ Fast context switches                                    │
│                                                              │
│  Cons:                                                       │
│  ✗ Race conditions, deadlocks                               │
│  ✗ One thread crash can crash process                       │
│  ✗ Complex synchronization needed                           │
│                                                              │
│  Example: Java servlets, .NET                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Event Loop Model

```
┌─────────────────────────────────────────────────────────────┐
│                   Event Loop Model                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Single Thread                       │  │
│  │                                                        │  │
│  │    ┌─────────────────────────────────────────────┐    │  │
│  │    │               Event Loop                     │    │  │
│  │    │                                              │    │  │
│  │    │   ┌─────────────────────────────────────┐   │    │  │
│  │    │   │           Event Queue               │   │    │  │
│  │    │   │ [HTTP] [DB] [File] [Timer] [HTTP]   │   │    │  │
│  │    │   └─────────────────────────────────────┘   │    │  │
│  │    │                    │                         │    │  │
│  │    │                    ▼                         │    │  │
│  │    │   while (hasEvents) {                       │    │  │
│  │    │       event = queue.pop()                   │    │  │
│  │    │       callback = event.callback             │    │  │
│  │    │       callback()  // Non-blocking!          │    │  │
│  │    │   }                                         │    │  │
│  │    │                                              │    │  │
│  │    └─────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  Pros:                                                       │
│  ✓ Very low memory per connection                           │
│  ✓ No thread synchronization needed                         │
│  ✓ High concurrency for I/O-bound tasks                     │
│                                                              │
│  Cons:                                                       │
│  ✗ CPU-bound tasks block everything                         │
│  ✗ Only uses one CPU core (need multiple processes)         │
│  ✗ Callback complexity ("callback hell")                    │
│                                                              │
│  Example: Node.js, Nginx, Redis                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Comparison

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Concurrency Model Comparison                         │
├──────────────────┬──────────────┬──────────────┬───────────────────────┤
│ Aspect           │Multi-Process │Multi-Thread  │Event Loop             │
├──────────────────┼──────────────┼──────────────┼───────────────────────┤
│ Memory per conn  │ ~10MB        │ ~1MB         │ ~10KB                 │
│ Max connections  │ ~1,000       │ ~10,000      │ ~1,000,000            │
│ CPU utilization  │ Good         │ Good         │ Single core           │
│ I/O handling     │ Blocking     │ Blocking     │ Non-blocking          │
│ CPU-bound work   │ Good         │ Good         │ Poor (blocks loop)    │
│ Complexity       │ Low          │ High         │ Medium                │
│ Isolation        │ High         │ Low          │ N/A                   │
├──────────────────┼──────────────┼──────────────┼───────────────────────┤
│ Examples         │ Apache,      │ Java,        │ Node.js, Nginx,       │
│                  │ PHP-FPM      │ Python       │ Redis                 │
└──────────────────┴──────────────┴──────────────┴───────────────────────┘
```

---

## I/O Models

### The C10K Problem

How to handle 10,000 concurrent connections on a single server?

```
Traditional approach: 1 thread per connection
10,000 connections = 10,000 threads
10,000 threads × 1MB stack = 10GB RAM (just for stacks!)

Solution: Non-blocking I/O with event loops
```

### Blocking vs Non-Blocking I/O

```
┌─────────────────────────────────────────────────────────────┐
│                    Blocking I/O                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Thread 1: read(socket)  ─────────[BLOCKED]─────────────▶   │
│                          (waiting for data)                  │
│                                                              │
│  Thread 2: read(socket)  ─────────[BLOCKED]─────────────▶   │
│                          (waiting for data)                  │
│                                                              │
│  Problem: Thread sits idle while waiting                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  Non-Blocking I/O                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Thread: read(socket)                                        │
│          │                                                   │
│          ├── Returns immediately (EAGAIN if no data)         │
│          │                                                   │
│          ├── Do other work...                                │
│          │                                                   │
│          ├── Check again later                               │
│          │                                                   │
│          └── Data available? Process it                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### I/O Multiplexing

```
┌─────────────────────────────────────────────────────────────┐
│                I/O Multiplexing (select/poll/epoll)          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Instead of one thread per connection:                       │
│  One thread monitors MANY connections                        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                  Event Loop Thread                      │ │
│  │                                                         │ │
│  │    events = epoll_wait(epoll_fd, timeout)              │ │
│  │                                                         │ │
│  │    for event in events:                                │ │
│  │        if event.is_readable:                           │ │
│  │            data = read(event.fd)                       │ │
│  │            process(data)                               │ │
│  │        if event.is_writable:                           │ │
│  │            write(event.fd, pending_data)               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│           │                                                  │
│           │ monitors                                         │
│           ▼                                                  │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐              │
│  │Sock 1│ │Sock 2│ │Sock 3│ │Sock 4│ │Sock N│              │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Evolution of I/O Multiplexing

```
┌─────────────────────────────────────────────────────────────┐
│           select → poll → epoll/kqueue                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  select (1983):                                              │
│  • Limited to 1024 file descriptors                         │
│  • O(n) scanning of all FDs                                 │
│  • Copies FD set to kernel each call                        │
│                                                              │
│  poll (1986):                                                │
│  • No FD limit                                               │
│  • Still O(n) scanning                                       │
│  • Still copies array each call                              │
│                                                              │
│  epoll (Linux, 2002) / kqueue (BSD):                        │
│  • No FD limit                                               │
│  • O(1) for events (only returns ready FDs)                 │
│  • Kernel maintains state (no copying)                      │
│  • Edge-triggered mode for efficiency                        │
│                                                              │
│  Performance with 10,000 connections:                        │
│  ┌────────────┬────────────┬────────────┐                   │
│  │  select    │   poll     │   epoll    │                   │
│  ├────────────┼────────────┼────────────┤                   │
│  │  O(10000)  │  O(10000)  │  O(ready)  │                   │
│  │  ~10ms     │  ~10ms     │  ~0.01ms   │                   │
│  └────────────┴────────────┴────────────┘                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Async I/O (io_uring)

```
┌─────────────────────────────────────────────────────────────┐
│                  io_uring (Linux 5.1+)                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  True async I/O with ring buffers:                          │
│                                                              │
│  ┌────────────┐         ┌─────────────┐                     │
│  │ User Space │         │ Kernel      │                     │
│  │            │         │             │                     │
│  │ ┌────────┐ │  mmap   │ ┌─────────┐ │                     │
│  │ │ Submit │◀┼─────────┼▶│ Submit  │ │                     │
│  │ │ Queue  │ │         │ │ Queue   │ │                     │
│  │ └────────┘ │         │ └─────────┘ │                     │
│  │            │         │             │                     │
│  │ ┌────────┐ │  mmap   │ ┌─────────┐ │                     │
│  │ │Complete│◀┼─────────┼▶│Complete │ │                     │
│  │ │ Queue  │ │         │ │ Queue   │ │                     │
│  │ └────────┘ │         │ └─────────┘ │                     │
│  └────────────┘         └─────────────┘                     │
│                                                              │
│  Benefits:                                                   │
│  • Zero-copy between user/kernel                            │
│  • Batch submissions                                         │
│  • True async (kernel completes I/O)                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Memory Management

### Virtual Memory

```
┌─────────────────────────────────────────────────────────────┐
│                    Virtual Memory                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Process A          Page Table          Physical RAM         │
│  ┌──────────┐      ┌──────────┐        ┌──────────┐         │
│  │ Virtual  │      │          │        │          │         │
│  │ Page 0   │─────▶│ Frame 5  │───────▶│ Frame 0  │         │
│  ├──────────┤      ├──────────┤        ├──────────┤         │
│  │ Page 1   │─────▶│ Frame 2  │───────▶│ Frame 1  │         │
│  ├──────────┤      ├──────────┤        ├──────────┤         │
│  │ Page 2   │─────▶│ (disk)   │        │ Frame 2  │◀── A.1  │
│  ├──────────┤      ├──────────┤        ├──────────┤         │
│  │ Page 3   │─────▶│ Frame 7  │        │ Frame 3  │         │
│  └──────────┘      └──────────┘        ├──────────┤         │
│                                        │ Frame 4  │         │
│                                        ├──────────┤         │
│                                        │ Frame 5  │◀── A.0  │
│                                        ├──────────┤         │
│                                        │ Frame 6  │         │
│                                        ├──────────┤         │
│                                        │ Frame 7  │◀── A.3  │
│                                        └──────────┘         │
│                                                              │
│  Benefits:                                                   │
│  • Isolation between processes                               │
│  • More memory than physical RAM (swap)                     │
│  • Memory-mapped files                                       │
│  • Shared libraries                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Page Faults

```
┌─────────────────────────────────────────────────────────────┐
│                     Page Fault Types                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Minor Page Fault (Soft):                                    │
│  • Page is in RAM but not mapped                            │
│  • Just update page table                                    │
│  • Cost: ~1μs                                               │
│                                                              │
│  Major Page Fault (Hard):                                    │
│  • Page must be loaded from disk                            │
│  • Read from swap or file                                    │
│  • Cost: ~10ms (10,000x slower!)                            │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Accessing memory address 0x7fff1234                   │  │
│  │                     │                                   │  │
│  │                     ▼                                   │  │
│  │  ┌─────────────────────────────────────────┐          │  │
│  │  │ Check page table for virtual address    │          │  │
│  │  └─────────────────────┬───────────────────┘          │  │
│  │           ┌────────────┴────────────┐                  │  │
│  │           ▼                         ▼                  │  │
│  │    ┌─────────────┐          ┌─────────────┐           │  │
│  │    │ Page mapped │          │ Page fault! │           │  │
│  │    │ Access RAM  │          │ Load page   │           │  │
│  │    └─────────────┘          └─────────────┘           │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Memory-Mapped Files

```python
# Memory-mapped file example
import mmap

# Map file to memory
with open('large_file.bin', 'r+b') as f:
    # Memory-map the file
    mm = mmap.mmap(f.fileno(), 0)

    # Access like an array (zero-copy!)
    data = mm[1000:2000]  # Read bytes 1000-2000

    # Modify in place
    mm[0:4] = b'TEST'

    # Changes are written to file
    mm.close()

# Benefits:
# - Zero-copy access to file data
# - Kernel handles caching
# - Multiple processes can share
# - Random access without seeking
```

---

## System Design Implications

### Choosing Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│              Concurrency Model Decision Tree                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Is workload I/O-bound or CPU-bound?                        │
│                │                                             │
│       ┌────────┴────────┐                                   │
│       ▼                 ▼                                   │
│   I/O-bound         CPU-bound                               │
│       │                 │                                    │
│       ▼                 ▼                                    │
│  Event loop +      Multi-process                            │
│  worker threads    (fork per core)                          │
│  (Node.js style)   (Python multiprocessing)                 │
│                                                              │
│  Examples:                                                   │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                                                          ││
│  │  Web Server (I/O-bound):                                ││
│  │  → Nginx (event loop)                                   ││
│  │  → Node.js (event loop + worker threads for CPU)        ││
│  │                                                          ││
│  │  Video Encoding (CPU-bound):                            ││
│  │  → FFmpeg (multi-process/thread)                        ││
│  │  → One process per CPU core                             ││
│  │                                                          ││
│  │  Database (Mixed):                                       ││
│  │  → PostgreSQL (multi-process)                           ││
│  │  → MySQL (multi-thread)                                 ││
│  │                                                          ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Connection Limits

```
┌─────────────────────────────────────────────────────────────┐
│                  Connection Scaling                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Per-connection overhead:                                    │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Model          │ Memory/conn │ Max connections          ││
│  ├─────────────────────────────────────────────────────────┤│
│  │ Thread/conn    │ ~1MB stack  │ ~1,000-10,000            ││
│  │ Process/conn   │ ~10MB       │ ~100-1,000               ││
│  │ Event loop     │ ~10KB       │ ~100,000-1,000,000       ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  System limits to consider:                                  │
│  • File descriptors (ulimit -n)                             │
│  • Ephemeral ports (for outbound connections)               │
│  • RAM for connection state                                 │
│  • CPU for context switching                                │
│                                                              │
│  Tuning commands:                                            │
│  $ sysctl net.core.somaxconn=65535                         │
│  $ sysctl net.ipv4.ip_local_port_range="1024 65535"        │
│  $ ulimit -n 1000000                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the difference between a process and a thread?
2. What is context switching and why is it expensive?
3. What is blocking vs non-blocking I/O?

### Intermediate
4. Explain the event loop model. What are its advantages and limitations?
5. What is the C10K problem and how is it solved?
6. Compare epoll and select. Why is epoll more efficient?

### Advanced
7. How would you design a web server to handle 1 million concurrent connections?
8. Explain virtual memory and page faults. How do they affect performance?
9. When would you choose multi-process over multi-thread architecture?

---

[← Previous: Networking Basics](../03-networking/README.md) | [Back to Module](../README.md) | [Next: Core Concepts →](../../02-core-concepts/README.md)
