# Thread-pool Server

This folder implements a server that uses a fixed-size thread pool (`ExecutorService`) to process incoming connections. This is the recommended approach for moderate to high concurrency.

Files
- `Server.java` — accepts sockets and submits handler tasks to a fixed `ThreadPoolExecutor`.

Design notes

- The server accepts connections and immediately submits a worker `Runnable` to the executor.
- If all pool threads are busy, tasks queue up until a worker is available (or until the queue is full, depending on implementation).

Flowchart

```mermaid
flowchart TD
  A[accept()] --> B{executor.accepts task}
  B -- yes --> C[task queued or running]
  C --> D[worker thread handles request]
  B -- no --> E[RejectedExecutionHandler]
```

Tuning parameters

- **pool size** — set to ~number-of-cores * (1 + targetBlockingFactor). For CPU-bound tasks keep near cores; for blocking I/O increase accordingly.
- **queue** — a bounded queue prevents unlimited memory use; tune size based on expected backlog.
- **rejection policy** — handle with `CallerRunsPolicy` or a custom handler to provide backpressure.

Pros

- Controlled number of threads and predictable resource usage.
- Better throughput than thread-per-connection at scale.

Cons

- Requires tuning (pool size, queue) for optimal performance.

Run

```
cd ThreadPool
javac Server.java
java Server

# Use Client.java from other folders or create multiple clients to benchmark
```

Benchmark tips

- Use `ab` (ApacheBench) or `wrk` to generate concurrent load and compare latency/throughput.
- Measure heap and thread counts (e.g., `jcmd <pid> VM.system_properties`, `jstack`) while testing.
