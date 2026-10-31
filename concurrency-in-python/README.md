# Concurrency in Python

> *Concurrency is about dealing with lots of things at once*. 
>
> *Parallelism is about doing lots of things at once.*
>
> *Not the same, but related.*
> 
> *One is about structure, one is about execution.*
> 
> *Concurrency provides a way to structure a solution to solve a problem that may (but not necessarily) be parallelizable.* ~ Rob Pike, co-inventor of the Go language

Parallelism is a special case of concurrency. All parallel systems are concurrent, but not all concurrent systems are parallel. In the early 2000s we used single-core machines that handled 100 processes concurrently on GNU Linux. A modern laptop with 4 CPU cores is routinely running more than 200 processes at any given time under normal, casual use. To execute 200 tasks in parallel, you’d need 200 cores. So, in practice, most computing is concurrent and not parallel. The OS manages hundreds of processes, making sure each has an opportunity to make progress, even if the CPU itself can’t do more than four things at once.

## 1. A Bit of Jargon

### Process
* An instance of a computer program while it is running, using memory and a slice of the CPU time. Modern desktop operating systems routinely manage hundreds of processes concurrently, with **each process isolated in its own private memory space**.

* Processes communicate via **pipes**, **sockets**, or **memory mapped files** -- all of which can only carry **raw bytes**. Python objects must be serialized (converted) into raw bytes to pass from one process to another. This is costly, and not all Python objects are serializable. 

* A process can spawn subprocesses, each called a child process. **These are also isolated from each other and from the parent**. Processes allow **preemptive multitasking**: *the OS scheduler preempts—i.e., suspends—each running process periodically to allow other processes to run. This means that a frozen process can’t freeze the whole system—in theory*.

### Thread

* An execution unit within a single process. *When a process starts, it uses a single thread: the main thread*. A process can create more threads to operate concurrently by calling operating system APIs. 

* **Threads within a process share the same memory space, which holds live Python objects**. This allows easy data sharing between threads, but can also lead to corrupted data when more than one thread updates the same object concurrently. 

* Like processes, threads also enable *preemptive multitasking under the supervision of the OS scheduler*. A thread consumes less resources than a process doing the same job.

### Coroutine
* *A function that can suspend itself and resume later*. In Python, classic coroutines are built from generator functions, and native coroutines are defined with `async def`. 

* Python coroutines usually run within a single thread under the supervision of an **event loop**, also in the same thread. Asynchronous programming frameworks such as asyncio, Curio, or Trio provide an event loop and supporting libraries for nonblocking, coroutine-based I/O.

* ***Coroutines support cooperative multitasking***: *each coroutine must explicitly cede control with the `yield` or `await` keyword, so that another may proceed concurrently (but not in parallel)*. This means that any blocking code in a coroutine blocks the execution of the event loop and all other coroutines—in contrast with the preemptive multitasking supported by processes and threads. 

* On the other hand, each coroutine consumes less resources than a thread or process doing the same job.

### Queue
* The implementation of a queue varies according to the underlying concurrency model: the queue package in Python’s standard library provides queue classes to support `threads` usually `FIFO`, while the `multiprocessing` and `asyncio` packages implement their own queue classes. The queue and asyncio packages also include queues that are not FIFO: `LifoQueue` and `PriorityQueue`.

### Lock
* An object that execution units can use to synchronize their actions and avoid corrupting data. While updating a shared data structure, the running code should hold an associated lock. This signals other parts of the program to wait until the lock is released before accessing the same data structure.

* The simplest type of lock is also known as a mutex (for mutual exclusion).

* The implementation of a lock depends on the underlying concurrency model.

## 2. Python’s Infamous GIL

* Each instance of the Python interpreter is a process. Access to object reference counts and other internal interpreter state is controlled by a lock, the Global Interpreter Lock (GIL). 

* ***Only one Python thread can hold the GIL at any time.** This means that only one thread can execute Python code at any time, regardless of the number of CPU cores.*

* To prevent a Python thread from holding the GIL indefinitely, ***Python’s bytecode interpreter pauses the current Python thread every `5ms`(`sys.getswitchinterval()`)*** by default releasing the GIL. The thread can then try to reacquire the GIL, but if there are other threads waiting for it, the ***OS scheduler may pick one of them to proceed***.

* Every Python standard library function that makes a ***syscall** (a syscall is a call from user code to a function of the operating system kernel. I/O, timers, and locks are some of the kernel services available through syscalls.)* releases the GIL. This includes all functions that perform disk I/O, network I/O, and time.sleep(). Many CPU-intensive functions in the NumPy/SciPy libraries, as well as the compressing/decompressing functions from the zlib and bz2 modules, also release the GIL.

* The effect of the GIL on network programming with Python threads is relatively small, because the I/O functions release the GIL, and reading or writing to the network always implies high latency—compared to reading and writing to memory. Consequently, each individual thread spends a lot of time waiting anyway, so their execution can be interleaved without major impact on the overall throughput. ***That’s why David Beazley says: “Python threads are great at doing nothing.”***

* Contention over the GIL slows down compute-intensive Python threads. Sequential, single-threaded code is simpler and faster for such tasks. If you want your application to make better use of the computational resources of multicore machines, you are advised to use `multiprocessing` or `concurrent.futures.ProcessPoolExecutor`. However, threading is still an appropriate model if you want to run multiple I/O-bound tasks simultaneously.

## Why `threading` shines in I/O operations? 
> Suppose I'm downloading 20 high resolution images from internet using requests module, each takes roughly 1 sec., if I do it sequentially it would take 20 seconds. 
But if I use multithreading it finishes in 1 second? How? 
Threads do not do multi-processing. The only thing that we did was triggering all 20 downloads almost simultaneously using thread pool executor. So, it does not have to wait for the others to finish. 
But still if CPU is not involved in downloading these images then who's involved? And what about reading file from disk into memory instead network call? who's responsible there? will large 20 files each taking 1 sec sequentially would finish in 1sec if multithreading applied?

The main thing to answer is who is doing the work during that 1 second? because that explains when threading helps and when it doesn’t.

### Who’s involved in a download

Your CPU does only a tiny part of it: building the request, TLS encryption and decryption, and copying bytes around, which is maybe a few milliseconds. The rest of that “1 second” is time spent waiting on other things:

* The network round trip to the server (latency)
* The remote server finding and preparing the image
* The bytes traveling over the wire, through your NIC (network interface card), which writes incoming data directly into memory (DMA) and signals the OS when it arrives

When your thread calls `requests.get()`, it ends up in a blocking system call. The OS parks that thread, and it uses zero CPU while parked. Python also releases the GIL during that wait, so other threads can run and fire off their own requests. With 20 threads, you have 20 requests in flight and 20 threads parked, all waiting on the network at once. ***The waiting overlaps, which is why 20 × 1s becomes about 1s.***

### The catch: it only works if the wait isn’t a shared bottleneck

The 1 second may be made of different things:

* If it’s mostly latency and server processing time, 20 parallel requests don’t compete, so you get close to the 20x speedup.
If it’s mostly bandwidth (big images saturating your connection), parallel downloads share the same pipe. Twenty downloads at once each get 1/20th of the speed, so the total stays around 20 seconds.

* With high-res images you’re likely somewhere in between, so expect a big speedup but not a perfect 1 second.

Eg. 

Say each image is 5 MB, so 20 images are 100 MB total:
* On a 1 Gbps link, that’s about 0.8s of wire time, so you’d still see a large speedup.
* On a 100 Mbps connection (about 12.5 MB/s), it’s about 8s no matter how many threads you use. The pipe is full, and the 20 downloads just split it.

### Role of other componets: sockets, kernel etc.

Sockets are the layer where the “waiting” actually happens. `requests` is a high-level wrapper, and underneath it every download is done through a `socket`.

```
requests.get() → urllib3 → http.client → socket (Python) → OS syscalls → kernel TCP stack → NIC
```

At its core, a socket is an endpoint for sending and receiving data across a network. Think of it as an Twitter handle. On Linux, a handle is called a file descriptor (fd) and it’s local to the process. The name “file descriptor” is just a name; it has nothing to do with files, nor does it describe anything.

Roughly, this is what requests.get(url) does with a socket:

```Python
import socket, ssl

sock = socket.create_connection(("example.com", 443))   # connect(): TCP handshake, ~1 round trip, BLOCKS
sock = ssl.create_default_context().wrap_socket(sock, server_hostname="example.com")  # TLS handshake, more round trips, BLOCKS
sock.sendall(b"GET /img.jpg HTTP/1.1\r\nHost: example.com\r\n\r\n")  # send request, quick
while chunk := sock.recv(65536):                          # BLOCKS until data arrives
    data += chunk
```

The blocking points are `connect()` and `recv()`.

#### Who does what while the thread sleeps
* Packets arrive at the NIC, which DMAs them into memory and raises an interrupt.
* The kernel’s TCP stack reassembles them, handles ordering and acknowledgments, and puts the payload into that socket’s receive buffer(i.e. process memory identified using PORT/fd).
* The kernel sees that a thread is sleeping on that socket and wakes it up.
* recv() returns, and Python copies the bytes out of the kernel buffer.

> So the answer to “who’s involved if the CPU isn’t?” is the NIC, the kernel’s network stack, and the remote machine. Your Python code is just the one asleep on the socket.

### Disk reads

The same principle applies. When you read a file, the thread asks the OS, the OS tells the disk controller to fetch the data, and the controller copies it into memory via DMA. The CPU is idle in the meantime, and Python releases the GIL during the read, so threads can overlap disk waits too. But whether 20 files that each take 1 second would finish in 1 second depends on the device:

NVMe/SSD: these handle many requests in parallel internally, so you’d see a real speedup, though it flattens once you hit the drive’s maximum throughput.
HDD: there’s one physical read head, so parallel reads can actually be slower because the head keeps jumping between files.
Files already in the OS cache: reading is just a memory copy, which is CPU work, so threads give little benefit.

### The rule of thumb

Threads help when your program spends its time waiting on something that can serve many requests at once. They don’t help if the work is CPU-bound (that’s where multiprocessing comes in, since the GIL blocks threads from running Python code in parallel), or if the thing you’re waiting on is already maxed out (a saturated network link or a single HDD).

## 1. Core Concurrency Concepts: Concurrency, I/O-Bound, and CPU-Bound Workloads

Concurrency in Python allows code to execute non-sequentially, providing a mechanism to speed up programs that spend substantial time waiting for external operations. Understanding when and why to use threads requires distinguishing between synchronous execution, concurrent execution, and the underlying nature of the workload.

### Synchronous vs. Concurrent Execution

In standard synchronous execution, Python executes statements line-by-line in sequential order. When a function initiates an operation that involves waiting—such as a network call or a sleep timer—the script halts all execution until that operation completes before moving to the subsequent line.

In contrast, concurrent execution using Python's `threading` module alters this behavior. When a thread encounters an I/O wait state, Python switches context to begin or resume execution of another thread. Although standard Python threads do not execute code truly in parallel on multiple CPU cores at the exact same instant, concurrency creates the functional result of overlapped execution by eliminating idle waiting time.

### I/O-Bound vs. CPU-Bound Workloads

The decision to use threading depends entirely on whether a workload is I/O-bound or CPU-bound.

| Workload Type | Characteristics & Examples | Threading Suitability |
| :--- | :--- | :--- |
| **I/O-Bound** | Operations spending most execution time waiting for input/output responses (e.g., HTTP requests, reading/writing disk files, network socket calls, sleep delays). | **High Benefit**: Drastically reduces overall execution time by overlapping wait states across threads. |
| **CPU-Bound** | Operations requiring heavy mathematical computations and data crunching (e.g., image manipulation, matrix operations, machine learning processing). | **Low / Negative Benefit**: Can slow down execution due to the overhead of creating, context switching, and destroying threads. Requires multiprocessing for parallelism. |

---

## 2. Low-Level Thread Management with the `threading` Module

Python's built-in `threading` module provides explicit control over thread instantiation, execution, and synchronization.

### Manual Thread Creation and Execution

To execute a function inside a dedicated thread, instantiate a `threading.Thread` object by setting the `target` parameter to the uncalled function reference. Calling `.start()` kicks off thread execution in the background, while `.join()` blocks the main script execution until the target thread completes.

Consider a baseline function that simulates an I/O delay:

```python
import time
import threading

def do_something(seconds):
    print(f'Sleeping {seconds} second(s)...')
    time.sleep(seconds)
    print('Done sleeping.')
```

Executing this function twice manually with individual thread instances:

```python
start = time.perf_counter()

t1 = threading.Thread(target=do_something, args=[1.5])
t2 = threading.Thread(target=do_something, args=[1.5])

t1.start()
t2.start()

t1.join()
t2.join()

finish = time.perf_counter()
print(f'Finished in {round(finish - start, 2)} second(s)')
```

In this setup, `t1.start()` and `t2.start()` initiate both tasks almost simultaneously. The main program execution reaches `t1.join()` and `t2.join()`, waiting for both background operations to complete. Rather than taking 3.0 seconds sequentially, the combined execution completes in approximately 1.5 seconds.

### Scaling Manual Threads with Loops

To run multiple threads dynamically, threads should be stored in a collection. Placing `.join()` directly inside the creation loop causes execution to block sequentially on each thread, negating concurrency. Instead, split thread launching and thread joining into distinct iterations.

```python
import time
import threading

start = time.perf_counter()

threads = []

# Step 1: Instantiate and start threads
for _ in range(10):
    t = threading.Thread(target=do_something, args=[1.5])
    t.start()
    threads.append(t)

# Step 2: Ensure all threads complete before main script finishes
for thread in threads:
    thread.join()

finish = time.perf_counter()
print(f'Finished in {round(finish - start, 2)} second(s)')
```

While synchronous execution of 10 calls with a 1.5-second delay would require 15.0 seconds, the concurrent loop implementation completes in approximately 1.5 seconds.

---

## 3. Modern Concurrency with `concurrent.futures.ThreadPoolExecutor`

Introduced in Python 3.2, the `concurrent.futures` module offers high-level abstractions for managing thread execution via thread pools. The `ThreadPoolExecutor` handles thread lifecycle management automatically and works seamlessly with context managers.

### Submitting Individual Tasks with `submit()`

The `submit()` method schedules a callable for execution and immediately returns a `Future` object encapsulating the execution state. Calling `.result()` on a `Future` returns the target function's return value, blocking until execution finishes.

```python
import time
import concurrent.futures

def do_something(seconds):
    print(f'Sleeping {seconds} second(s)...')
    time.sleep(seconds)
    return f'Done sleeping...{seconds}'

with concurrent.futures.ThreadPoolExecutor() as executor:
    f1 = executor.submit(do_something, 1)
    f2 = executor.submit(do_something, 1)

    print(f1.result())
    print(f2.result())
```

### Collecting Results in Order of Completion via `as_completed()`

When running tasks with varying durations, `concurrent.futures.as_completed()` returns an iterator yielding futures as they finish, regardless of the order in which they were submitted.

```python
import time
import concurrent.futures

seconds_list = [5, 4, 3, 2, 1]

with concurrent.futures.ThreadPoolExecutor() as executor:
    results = [executor.submit(do_something, sec) for sec in seconds_list]

    for f in concurrent.futures.as_completed(results):
        print(f.result())
```

In this execution:
1. All five threads start almost simultaneously.
2. The 1-second thread completes first, yielding its result immediately.
3. Subsequent threads yield results in ascending order of completion (1, 2, 3, 4, 5 seconds).
4. Total execution runtime equals the longest single task (5.0 seconds), compared to 15.0 seconds synchronously.

### Bulk Mapping with `executor.map()`

When applying a single function across an iterable of inputs, `executor.map()` provides a concise syntax. Unlike `as_completed()`, `executor.map()` returns results strictly in the order tasks were started.

```python
import time
import concurrent.futures

seconds_list = [5, 4, 3, 2, 1]

with concurrent.futures.ThreadPoolExecutor() as executor:
    results = executor.map(do_something, seconds_list)

    for result in results:
        print(result)
```

Key characteristics of `executor.map()` and context managers include:
- **Result Ordering**: Iterating over `results` yields items matching the input iterable's sequence. If the first item (5 seconds) takes longer than subsequent items, iteration blocks until the first item finishes, after which subsequent ready results yield instantly.
- **Automatic Joining**: Exiting the `with` block automatically joins all worker threads, ensuring all background tasks conclude before execution proceeds past the context manager.
- **Exception Handling**: If a thread raises an exception during execution, the exception is deferred until its specific value is retrieved from the results iterator.

---

## 4. Real-World Case Study: Asynchronous Web Image Downloader

To measure real-world performance gains, consider downloading high-resolution images over a network connection—a classic I/O-bound operation.

### Synchronous Downloader Baseline

Downloading 15 high-resolution images sequentially using the `requests` library requires waiting for the full HTTP response of each image before requesting the next.

```python
import time
import requests

img_urls = [
    'photo_url_1',
    'photo_url_2',
    # ... list of 15 high-resolution photo URLs
]

start = time.perf_counter()

for img_url in img_urls:
    img_bytes = requests.get(img_url).content
    img_name = img_url.split('/')[3] + '.jpg'
    with open(img_name, 'wb') as img_file:
        img_file.write(img_bytes)
        print(f'{img_name} was downloaded...')

finish = time.perf_counter()
print(f'Finished in {round(finish - start, 2)} second(s)')
```

**Synchronous Benchmark Result**: Approximately **23 seconds** total runtime.

### Refactored Concurrent Downloader

By isolating the single-image retrieval logic into a worker function and mapping it across the URL list with `ThreadPoolExecutor`, requests are dispatched concurrently.

```python
import time
import requests
import concurrent.futures

img_urls = [
    'photo_url_1',
    'photo_url_2',
    # ... list of 15 high-resolution photo URLs
]

def download_image(img_url):
    img_bytes = requests.get(img_url).content
    img_name = img_url.split('/')[3] + '.jpg'
    with open(img_name, 'wb') as img_file:
        img_file.write(img_bytes)
        print(f'{img_name} was downloaded...')

start = time.perf_counter()

with concurrent.futures.ThreadPoolExecutor() as executor:
    executor.map(download_image, img_urls)

finish = time.perf_counter()
print(f'Finished in {round(finish - start, 2)} second(s)')
```

**Concurrent Benchmark Result**: Approximately **5 seconds** total runtime (over a 4.5x speedup).

---

## 5. Architectural Recommendations & Future Considerations

### Summary of Threading Best Practices

1. **Identify Task Boundaries**: Use threading strictly for I/O-bound operations (network requests, file system read/write operations, database queries).
2. **Prefer `ThreadPoolExecutor`**: Use `concurrent.futures.ThreadPoolExecutor` with context managers for cleaner code, automatic lifecycle joining, and unified exception handling.
3. **Select the Right Result Handler**:
   - Use `as_completed()` when task order does not matter and immediate handling of completed tasks is desired.
   - Use `executor.map()` when processing output sequentially matching the input order.

### Transitioning to Multiprocessing for CPU-Bound Workloads

If downloaded images require CPU-intensive post-processing (e.g., resizing, applying filters, format conversion), threading will fail to deliver performance gains due to thread management overhead. For CPU-bound workloads, Python's `multiprocessing` module should be used to achieve true CPU core parallelism.
