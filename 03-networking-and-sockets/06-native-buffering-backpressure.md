# Native Buffering & Backpressure

When moving sockets to C++, you become responsible for memory management. If the server sends data at 100MB/s and your JS code can only process 10MB/s, your app will crash with an **OutOfMemory (OOM)** error unless you handle **Backpressure**.

## 1. The Producer-Consumer Problem
*   **Producer**: The network socket (fast).
*   **Consumer**: The JS thread/UI (slower).

In a JSI implementation, your C++ background thread is the producer. It pushes events into a queue for the JS thread to consume via `CallInvoker`.

## 2. High-Water Marks
You must set a "High-Water Mark" (a limit) on your internal C++ queue.

```cpp
// C++ logic
if (internalQueue.size() > MAX_QUEUE_SIZE) {
    // Stop reading from the TCP socket!
    // This tells the OS to stop acknowledging packets, 
    // which tells the server to slow down (TCP Window Scaling).
    pauseSocketReading();
}
```

## 3. Ring Buffers (Circular Buffers)
For high-performance streaming (like audio or video data), don't use `std::vector` which reallocates. Use a **Fixed-size Ring Buffer**.
*   Data is written to the end and read from the start.
*   If the write pointer catches the read pointer, the buffer is full.
*   This provides **constant-time (O(1))** performance and zero memory allocation overhead.

## 4. Zero-Copy via `jsi::ArrayBuffer`
One of the biggest performance wins in JSI is sharing memory.

1.  **C++**: Receives data into a pre-allocated buffer.
2.  **JSI**: Create a `jsi::ArrayBuffer` that points directly to that C++ memory address.
3.  **JS**: Accesses the data without a single byte being copied.

**Warning**: You must ensure the C++ memory stays valid as long as the JS `ArrayBuffer` exists. Use `std::shared_ptr` or a custom `jsi::Buffer` implementation.

## 5. Write Buffering (Nagle's Algorithm)
When sending data from JS -> C++ -> Server:
*   By default, TCP might wait to "batch" small writes (Nagle's Algorithm).
*   For real-time apps, you usually want to disable this:
    ```cpp
    int one = 1;
    setsockopt(socket_fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof(one));
    ```
*   This reduces latency at the cost of slightly more network overhead.

---

### Key Takeaway for JSI:
If you "flood" the JS thread with `CallInvoker->invokeAsync()` calls, the UI will lag. Implement a **batching mechanism** in C++: collect 10 small packets and send them as one "array of events" to JS in a single tick.
