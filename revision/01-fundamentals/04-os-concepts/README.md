# Operating System Concepts

Understanding OS fundamentals helps you reason about performance, concurrency, and resource management in distributed systems.

## Table of Contents
- [Processes and Threads](#processes-and-threads)
- [Memory Management](#memory-management)
- [I/O Models](#io-models)
- [File Systems](#file-systems)
- [Key Takeaways](#key-takeaways)

## Processes and Threads

### Process

A process is an instance of a running program with its own memory space.

```
┌─────────────────────────────────────────┐
│              Process                     │
├─────────────────────────────────────────┤
│  ┌─────────────────────────────────┐    │
│  │         Code (Text)             │    │
│  ├─────────────────────────────────┤    │
│  │         Data (Global vars)      │    │
│  ├─────────────────────────────────┤    │
│  │         Heap (Dynamic alloc)    │    │
│  │              ↓                  │    │
│  │                                 │    │
│  │              ↑                  │    │
│  │         Stack (Local vars)      │    │
│  └─────────────────────────────────┘    │
│                                         │
│  PID: 1234                              │
│  State: Running                         │
│  Open files: [fd0, fd1, fd2, ...]       │
└─────────────────────────────────────────┘
```

### Thread

A thread is a lightweight unit of execution within a process. Threads share the process's memory.

```
┌─────────────────────────────────────────────────────────────┐
│                         Process                              │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Shared Memory (Heap, Data)              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Thread 1   │  │  Thread 2   │  │  Thread 3   │         │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤         │
│  │   Stack     │  │   Stack     │  │   Stack     │         │
│  │   Registers │  │   Registers │  │   Registers │         │
│  │   PC        │  │   PC        │  │   PC        │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### Process vs Thread

| Aspect | Process | Thread |
|--------|---------|--------|
| Memory | Isolated | Shared |
| Creation cost | High (~10ms) | Low (~1ms) |
| Context switch | Expensive | Cheaper |
| Communication | IPC (pipes, sockets) | Shared memory |
| Crash isolation | Independent | Affects all threads |
| Scaling | Multi-core via fork | Multi-core via threads |

### Context Switching

```
CPU
 │
 │  ┌─────────────┐     Save state      ┌─────────────┐
 │  │  Process A  │ ─────────────────▶ │  PCB A      │
 │  │  (running)  │                     │  (saved)    │
 │  └─────────────┘                     └─────────────┘
 │         │
 │         │  Context Switch (~1-10μs for threads, ~10-100μs for processes)
 │         ▼
 │  ┌─────────────┐     Load state      ┌─────────────┐
 │  │  Process B  │ ◀───────────────── │  PCB B      │
 │  │  (running)  │                     │  (saved)    │
 │  └─────────────┘                     └─────────────┘
```

### Concurrency Patterns

**Multi-process (fork)**:
```
┌─────────────┐
│   Parent    │
│   Process   │
└──────┬──────┘
       │ fork()
       ▼
┌──────┴──────┐
│             │
▼             ▼
┌─────────┐ ┌─────────┐
│ Child 1 │ │ Child 2 │
└─────────┘ └─────────┘
```

**Multi-threaded**:
```
┌─────────────────────────────┐
│         Process             │
│  ┌───────┐ ┌───────┐       │
│  │Thread1│ │Thread2│ ...   │
│  └───────┘ └───────┘       │
└─────────────────────────────┘
```

**Event loop (single-threaded async)**:
```
┌─────────────────────────────────────────┐
│              Event Loop                  │
│  ┌─────┐                                │
│  │Queue│ ──▶ Handle ──▶ Handle ──▶ ... │
│  └─────┘     Event 1    Event 2         │
└─────────────────────────────────────────┘
```

### Thread Safety

**Race Condition Example**:
```
counter = 0

Thread A:              Thread B:
read counter (0)       read counter (0)
add 1                  add 1
write counter (1)      write counter (1)

Expected: 2
Actual: 1 (lost update)
```

**Solutions**:

1. **Mutex (Mutual Exclusion)**:
```python
mutex = Lock()

def increment():
    mutex.acquire()
    global counter
    counter += 1
    mutex.release()
```

2. **Atomic Operations**:
```python
from threading import atomic
counter = atomic.AtomicInteger(0)
counter.increment()  # Thread-safe
```

3. **Lock-free Data Structures**:
```
Compare-And-Swap (CAS):
if current_value == expected:
    current_value = new_value
    return True
return False
```

## Memory Management

### Memory Hierarchy

```
┌─────────────────┐  ◀── Fastest, smallest, most expensive
│   CPU Registers │      ~1ns, ~KB
├─────────────────┤
│    L1 Cache     │      ~2ns, ~64KB
├─────────────────┤
│    L2 Cache     │      ~10ns, ~256KB
├─────────────────┤
│    L3 Cache     │      ~40ns, ~MB
├─────────────────┤
│      RAM        │      ~100ns, ~GB
├─────────────────┤
│      SSD        │      ~100μs, ~TB
├─────────────────┤
│      HDD        │      ~10ms, ~TB
└─────────────────┘  ◀── Slowest, largest, cheapest
```

### Virtual Memory

Virtual memory allows processes to use more memory than physically available.

```
┌─────────────────┐          ┌─────────────────┐
│ Virtual Memory  │          │ Physical Memory │
│   (per process) │          │    (shared)     │
├─────────────────┤          ├─────────────────┤
│ Page 0  ────────┼────┐     │ Frame 0         │
│ Page 1  ────────┼──┐ │     │ Frame 1  ◀──────┼─┐
│ Page 2  ────────┼┐ │ │     │ Frame 2  ◀──────┼─┼─┐
│ Page 3 (disk)   │││ │ └───▶│ Frame 3         │ │ │
└─────────────────┘││ │      │ Frame 4         │ │ │
                   ││ └─────▶│ Frame 5         │ │ │
                   │└───────▶│ Frame 6         │ │ │
                   └────────▶│ Frame 7         │ │ │
                             └─────────────────┘ │ │
                                    ▲            │ │
┌─────────────────┐                 │            │ │
│ Process B       │                 │            │ │
│ Page 0  ────────┼─────────────────┘            │ │
│ Page 1  ────────┼──────────────────────────────┘ │
│ Page 2  ────────┼────────────────────────────────┘
└─────────────────┘
```

### Page Faults

When a process accesses a page not in RAM:

```
1. Process accesses virtual address
2. MMU checks page table
3. Page not in RAM → Page Fault
4. OS loads page from disk
5. Page table updated
6. Process resumes
```

**Impact**: Page faults are expensive (~1-10ms vs ~100ns for RAM access).

### Memory-Mapped Files

Map file contents directly to memory:

```python
import mmap

with open('large_file.bin', 'r+b') as f:
    mm = mmap.mmap(f.fileno(), 0)
    # Access file like memory
    data = mm[0:100]
    mm[0:5] = b'Hello'
    mm.close()
```

**Use cases**: Database storage engines, shared memory between processes.

## I/O Models

### Blocking I/O

```
Process          Kernel           Disk
   │                │               │
   │── read() ─────▶│               │
   │   (blocked)    │── request ───▶│
   │                │               │
   │                │◀── data ──────│
   │◀── data ───────│               │
   │                │               │
   │  (continues)   │               │
```

**Problem**: Thread is blocked, can't do other work.

### Non-blocking I/O

```
Process          Kernel           Disk
   │                │               │
   │── read() ─────▶│               │
   │◀── EAGAIN ─────│  (not ready)  │
   │                │               │
   │── read() ─────▶│               │
   │◀── EAGAIN ─────│  (not ready)  │
   │                │               │
   │── read() ─────▶│               │
   │◀── data ───────│◀── data ──────│
```

**Problem**: Busy waiting wastes CPU.

### I/O Multiplexing (select/poll/epoll)

```
Process          Kernel           Multiple FDs
   │                │               │
   │── epoll_wait()▶│               │
   │   (blocked)    │  (monitors    │
   │                │   all FDs)    │
   │                │◀── FD3 ready ─│
   │◀── [FD3] ──────│               │
   │                │               │
   │── read(FD3) ──▶│               │
   │◀── data ───────│               │
```

**Advantage**: Single thread handles thousands of connections.

### Comparison of I/O Multiplexing

| Method | Max FDs | Performance | Portability |
|--------|---------|-------------|-------------|
| select | 1024 | O(n) | Universal |
| poll | Unlimited | O(n) | POSIX |
| epoll | Unlimited | O(1) | Linux only |
| kqueue | Unlimited | O(1) | BSD/macOS |

### Async I/O

```
Process          Kernel           Disk
   │                │               │
   │── aio_read() ─▶│               │
   │◀── (returns) ──│── request ───▶│
   │                │               │
   │  (does other   │               │
   │   work)        │               │
   │                │◀── data ──────│
   │◀── signal ─────│               │
   │                │               │
   │  (handles      │               │
   │   completion)  │               │
```

### Event Loop Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Event Loop                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │                 Event Queue                      │    │
│  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐            │    │
│  │  │ E1 │ │ E2 │ │ E3 │ │ E4 │ │... │            │    │
│  │  └────┘ └────┘ └────┘ └────┘ └────┘            │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                                │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Event Handler                       │    │
│  │  while (true) {                                 │    │
│  │    event = queue.pop()                          │    │
│  │    handler = handlers[event.type]               │    │
│  │    handler(event)                               │    │
│  │  }                                              │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Used by**: Node.js, Nginx, Redis, Netty

## File Systems

### File System Basics

```
┌─────────────────────────────────────────┐
│              File System                 │
├─────────────────────────────────────────┤
│  Superblock    │ Metadata about FS      │
├─────────────────────────────────────────┤
│  Inode Table   │ File metadata          │
├─────────────────────────────────────────┤
│  Data Blocks   │ Actual file content    │
└─────────────────────────────────────────┘
```

### Inode Structure

```
┌─────────────────────────────────────┐
│              Inode                   │
├─────────────────────────────────────┤
│  Mode (permissions)                  │
│  Owner (UID, GID)                   │
│  Size                               │
│  Timestamps (atime, mtime, ctime)   │
│  Link count                         │
├─────────────────────────────────────┤
│  Direct blocks (12)     ──────────▶ Data
│  Indirect block         ──────────▶ Pointers ──▶ Data
│  Double indirect        ──────────▶ Pointers ──▶ Pointers ──▶ Data
│  Triple indirect        ──────────▶ ...
└─────────────────────────────────────┘
```

### Common File Systems

| File System | Max File Size | Features |
|-------------|---------------|----------|
| ext4 | 16 TB | Journaling, widely used on Linux |
| XFS | 8 EB | High performance, large files |
| Btrfs | 16 EB | Copy-on-write, snapshots |
| ZFS | 16 EB | Data integrity, compression |
| NTFS | 16 TB | Windows, ACLs |

### Sequential vs Random Access

```
Sequential Access:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │  ──▶ Fast (100+ MB/s HDD, 500+ MB/s SSD)
└───┴───┴───┴───┴───┴───┴───┴───┘

Random Access:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 5 │   │ 2 │   │ 8 │   │ 1 │   │  ──▶ Slow (100s IOPS HDD, 100K IOPS SSD)
└───┴───┴───┴───┴───┴───┴───┴───┘
```

### I/O Performance Numbers

| Operation | HDD | SSD | NVMe |
|-----------|-----|-----|------|
| Sequential read | 100-200 MB/s | 500 MB/s | 3000+ MB/s |
| Sequential write | 100-200 MB/s | 400 MB/s | 2000+ MB/s |
| Random read IOPS | 100-200 | 50K-100K | 500K+ |
| Random write IOPS | 100-200 | 30K-50K | 200K+ |
| Latency | 5-10ms | 0.1ms | 0.02ms |

### Write-Ahead Logging (WAL)

Databases use WAL for durability:

```
1. Write to log (sequential, fast)
2. Acknowledge to client
3. Apply to data files (background)
4. Checkpoint (mark log as applied)
```

```
┌─────────────────┐     ┌─────────────────┐
│   WAL Log       │     │   Data Files    │
│  (append-only)  │     │  (random I/O)   │
├─────────────────┤     ├─────────────────┤
│ TX1: INSERT ... │ ───▶│ Page updates    │
│ TX2: UPDATE ... │     │ applied in      │
│ TX3: DELETE ... │     │ background      │
└─────────────────┘     └─────────────────┘
```

## Key Takeaways

1. **Threads share memory** - Cheaper but need synchronization
2. **Processes are isolated** - Safer but more overhead
3. **Event loops scale** - Single thread handles many connections
4. **Memory hierarchy matters** - Cache misses are expensive
5. **Disk I/O is slow** - Batch writes, use SSDs, buffer in memory
6. **Sequential > Random** - Design for sequential access patterns

## Practice Questions

1. Why does Redis use a single-threaded event loop?
2. How does Nginx handle 10,000 concurrent connections?
3. What causes a page fault and how does it affect performance?
4. Why do databases use write-ahead logging?

## Further Reading

- [The Linux Programming Interface](https://man7.org/tlpi/)
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)

---

[Back to Module 1](../README.md) | Next: [Module 2: Core Concepts](../../02-core-concepts/README.md)
