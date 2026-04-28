# Networking Fundamentals for WebSockets

To build a high-performance, non-blocking C++ JSI socket library, you must understand the underlying layers. WebSockets don't exist in a vacuum; they are an evolution of the traditional networking stack.

## 1. The OSI Model vs. TCP/IP Stack

WebSockets operate at the **Application Layer**, but their performance is dictated by the **Transport Layer**.

### The Layers (Simplified for Sockets)
1.  **Physical/Data Link**: Ethernet/Wi-Fi (the hardware).
2.  **Network (IP)**: Handles routing. It knows how to get a packet from IP `A` to IP `B`. It doesn't care about reliability.
3.  **Transport (TCP/UDP)**:
    *   **TCP (Transmission Control Protocol)**: What WebSockets use. It provides a reliable, ordered, and error-checked stream of octets.
    *   **UDP (User Datagram Protocol)**: Faster, but unreliable. Used for games/VOIP. Not used for standard WebSockets.
4.  **Application (HTTP/WS)**: The protocol that defines the format of the data being sent.

## 2. TCP: The Foundation
WebSockets rely on TCP's **Full-Duplex** nature. Once a TCP connection is established:
*   The client can send data to the server.
*   The server can send data to the client.
*   They can do this simultaneously.

### The 3-Way Handshake (TCP Level)
Before a WebSocket can start, a TCP connection must be born:
1.  **SYN**: Client sends "Let's sync."
2.  **SYN-ACK**: Server says "I hear you, let's sync back."
3.  **ACK**: Client says "Got it, we are connected."

## 3. HTTP vs. WebSockets: The "Handshake"

WebSockets start as an HTTP request. This is critical for compatibility with firewalls and proxies that expect traffic on port 80/443.

### The Upgrade Request
The client sends a standard HTTP GET request with special headers:

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

### The Server Response (101 Switching Protocols)
If the server supports WebSockets, it responds with:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**What just happened?**
The TCP socket is still open, but the protocol has changed from HTTP (stateless, request-response) to WebSocket (stateful, continuous stream). From this point forward, the data follows the **RFC 6455 binary framing format**.

## 4. Why Native C++ for this?
In a standard React Native environment, the "Bridge" or even standard JS `WebSocket` API can be a bottleneck for high-frequency data (e.g., 60fps game state or high-speed telemetry).

*   **JS Thread Congestion**: Parsing complex JSON or binary packets in JS can freeze the UI.
*   **Context Switching**: Moving data from the native network stack -> bridge -> JS context adds latency.
*   **Thread Safety**: By moving the socket to a C++ background thread, you can process packets, handle encryption (TLS), and manage buffers without touching the JS event loop until a fully formed "event" is ready to be dispatched.

---

### Key Concepts for JSI Implementation:
*   **Socket Descriptors**: In C++, a socket is just an integer (on Linux/Android). You use `select()`, `poll()`, or `epoll()` to wait for data without blocking the whole program.
*   **Non-Blocking I/O**: You set the socket to `O_NONBLOCK` so that `read()` returns immediately if there's no data, allowing your C++ loop to keep running.
