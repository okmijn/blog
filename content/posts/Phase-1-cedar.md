---
title: "Cedar: Evolution of a High-Performance KV Store"
date: 2026-06-15T00:00:00Z
---
# Building a High-Throughput Key-Value Store in Go

In this post, I will document the architectural journey of Cedar. We will explore how it evolved from a simple local script into a highly concurrent TCP server capable of handling nearly 190,000 requests per second, the tradeoffs made along the way, and the low-level Go patterns that made it possible.

## The Requirements

Before writing any code, it was critical to define what Cedar needed to be (and what it didn't need to be). Clear requirements direct the architecture and prevent feature creep.

*   **High Throughput:** The engine must handle massive read/write volumes with minimal latency.
*   **Concurrency:** It must be thread-safe. Multiple network clients must be able to read and write simultaneously without data races or corruption.
*   **Extensibility:** The execution logic must be decoupled from the transport layer. I wanted to be able to interact with the database via a local CLI or a TCP socket without rewriting the core logic.
*   **Durability:** Data cannot be ephemeral. The store must have a mechanism to save state to disk and restore it upon restart.

**Out of scope:**
*   Complex relations or SQL parsing.
*   Distributed consensus (Raft/Paxos) for this specific iteration.


> Code is mostly human generated. Unit/Benchmark test and doc is AI written.

## 1. The Core Engine and Thread Safety

At the heart of any in-memory KV store is a hash map. Go provides a highly efficient built-in `map[string]string`, giving us $O(1)$ average-time complexity for lookups and insertions.

However, standard Go maps are **not thread-safe**. If one goroutine writes to a map while another reads from it, the program will panic.

### The Concurrency Tradeoff
To solve this, we need a locking mechanism. The naive approach is a standard `sync.Mutex`. 
```go
// The Naive Approach
s.mu.Lock()
val := s.Data[key]
s.mu.Unlock()
```
The problem here is lock contention. If 1,000 clients try to read simultaneously, they form a single-file line.

Instead, Cedar uses a `sync.RWMutex` (Reader-Writer Mutex).

```go
type Store struct {
	mu           sync.RWMutex
	Data         map[string]string
	SnapshotPath string
}

func (s *Store) Get(key string) (string, bool) {
	s.mu.RLock() // Multiple readers allowed simultaneously
	defer s.mu.RUnlock()
	val, ok := s.Data[key]
	return val, ok
}

func (s *Store) Set(key, value string) {
	s.mu.Lock() // Exclusive lock. Blocks other writers AND readers.
	defer s.mu.Unlock()
	s.Data[key] = value
}
```
**The Tradeoff:** An `RWMutex` optimizes for read-heavy workloads. It allows infinite concurrent readers, making `GET` operations blazing fast. The tradeoff is that a write operation must wait for all active reads to finish before it can acquire the exclusive lock.

## 2. Decoupling Intent from Delivery

In the initial prototype, Cedar was a simple CLI tool. The logic to parse a command, execute it, and print the result to the terminal was all tightly coupled. 

When it came time to add a TCP server, this architecture broke down. I couldn't reuse the logic because it was hardcoded to use `fmt.Println` to `os.Stdout`.

### Interface Injection
To solve this, I employed the Strategy pattern by injecting the `io.Writer` interface into the core execution engine.

```go
// internal/engine/executor.go
type commandHandler func(s *store.Store, args []string, out io.Writer) error

func handleGet(s *store.Store, args []string, out io.Writer) error {
	val, ok := s.Get(args[0])
	if !ok {
		return fmt.Errorf("key not found")
	}
	fmt.Fprintln(out, val) // Writes seamlessly to wherever 'out' points
	return nil
}
```
Because a TCP connection (`net.Conn`) implements `io.Writer`, and the standard output (`os.Stdout`) also implements `io.Writer`, the engine no longer cares *who* is asking for the data.

### The Command Registry
As the number of commands grew (`SET`, `GET`, `DEL`, `SCAN`), a `switch` statement becameaO(N) lookup time and violated the Open-Closed principle.

<mark>This does not really make huge difference in terms of time becuase of limited features</mark>

I replaced the switch with a map-based **Command Registry**.

```go
var Registry = map[string]commandHandler{}

func init() {
	Registry["set"] = handleSet
	Registry["get"] = handleGet
    // Adding a new command is as simple as adding a line here.
}

func Execute(s *store.Store, parts []string, out io.Writer) error {
	cmdName := strings.ToLower(parts[0])
	handler, ok := Registry[cmdName]
	if !ok {
		return fmt.Errorf("invalid command")
	}
	return handler(s, parts[1:], out) // O(1) Dispatch
}
```

## 3. The TCP Server and Concurrency

With the engine decoupled, wrapping Cedar in a TCP server was remarkably simple. The heavy lifting is handled by Go's lightweight concurrency primitives: goroutines.

```go
// internal/tcp/tcp.go
func Start(s *store.Store, addr string) error {
	listener, err := net.Listen("tcp", addr)
	// ...
	for {
		conn, err := listener.Accept()
		// ...
		// Spawn a lightweight thread for every new client
		go handleConnection(s, conn)
	}
}

func handleConnection(s *store.Store, conn net.Conn) {
	defer conn.Close()
	scanner := bufio.NewScanner(conn)
	for scanner.Scan() {
		parts := strings.Fields(scanner.Text())
		
		// Pass the TCP connection directly into the engine as the io.Writer
		err := engine.Execute(s, parts, conn)
        // ... error handling
	}
}
```
This architecture allows Cedar to handle thousands of concurrent TCP connections, limited primarily by the OS's file descriptor limits and network bandwidth.

## 4. Durability: Snapshots vs. The Write-Ahead Log

A memory-only database is just a cache. To make Cedar a true database, data must survive a restart.

Currently, Cedar implements **Snapshotting**. When the process receives an interrupt signal (`SIGTERM`), it halts operations, serializes the entire `map[string]string`, and writes it to `snapshot.txt`.

### The Problem with Snapshots
While simple to implement, snapshotting has severe limitations:
1. **$O(N)$ Cost**: Writing 2 million keys to disk blocks the system.
2. **Data Loss**: If the server loses power unexpectedly *before* a snapshot occurs, all data since the last boot is lost.

### The Future: Write-Ahead Logging (WAL)
In the next major iteration, Cedar will transition to an **Append-Only Log (WAL)**. 
Instead of writing the whole map, every single `SET` or `DEL` operation will be appended to the end of a file in real-time.

```text
// Example WAL structure
SET:user:1=kartik
SET:user:2=daan
DEL:user:1
```
Because appending to a file is an $O(1)$ sequential I/O operation, it is drastically faster than a full snapshot. Upon reboot, the system simply "replays" the log line-by-line to rebuild the in-memory map.

To balance speed and safety, we will implement **Periodic `fsync`**, where the OS buffers the writes immediately (fast), and a background goroutine forces the data to the physical disk every 1 second (safe).


## Conclusion and Performance

Through careful use of Go's concurrency primitives, interface injection, and decoupling, Cedar has matured into a highly capable system.

In our stress tests using a pre-populated dataset of **1,000,000 keys**:
- Single-client `GET` latency hovers around **15µs**.
- With **100 concurrent clients**, the server processes over **180,000 requests per second** with a median latency of ~0.5ms.

Building Cedar has been an incredible deep-dive into the realities of database engineering. In future posts, we will explore implementing the WAL, and replacing the underlying Hash Map with a Radix Tree to optimize our $O(N)$ prefix scanning bottleneck.