# WebSocket Protocol Deep Dive (RFC 6455)

Once the handshake is complete, you are no longer sending HTTP strings. You are sending **Data Frames**. To build a C++ parser, you must understand exactly how these bytes are structured.

## 1. The Anatomy of a Frame
A WebSocket frame consists of a header followed by an optional payload.

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 +---------------------------------------------------------------+
```

### Breakdown:
*   **FIN (1 bit)**: Indicates if this is the final fragment of a message.
*   **Opcode (4 bits)**: What kind of data is this?
    *   `0x1`: Text
    *   `0x2`: Binary
    *   `0x8`: Connection Close
    *   `0x9`: Ping
    *   `0xA`: Pong
*   **Mask (1 bit)**: **CRITICAL.** All frames sent from Client to Server *must* be masked. Frames from Server to Client are *not* masked.
*   **Payload Length**:
    *   If 0-125: That's the length.
    *   If 126: Next 2 bytes are the length (16-bit uint).
    *   If 127: Next 8 bytes are the length (64-bit uint).

## 2. Masking (Client to Server)
The "Masking Key" is 4 random bytes. To send data, the client XORs each byte of the payload with the masking key.

**C++ Pseudocode for Unmasking:**
```cpp
void unmask(uint8_t* payload, size_t length, uint8_t* maskKey) {
    for (size_t i = 0; i < length; i++) {
        payload[i] = payload[i] ^ maskKey[i % 4];
    }
}
```
In your JSI C++ core, you will implement this in a background thread to keep the JS thread free for rendering.

## 3. Control Frames (Heartbeats)
WebSockets use **Ping** (`0x9`) and **Pong** (`0xA`) to keep the connection alive.
*   If a client receives a Ping, it *must* send a Pong back as soon as possible.
*   In a C++ implementation, your background loop can handle Pings automatically without notifying the JS layer, reducing noise in the JS environment.

## 4. Connection Lifecycle
1.  **CONNECTING**: Handshake in progress.
2.  **OPEN**: Data can flow.
3.  **CLOSING**: Closing handshake (Close frame sent).
4.  **CLOSED**: TCP connection terminated.

## 5. Security (WSS)
WSS is WebSockets over **TLS (Transport Layer Security)**.
*   In C++, you would link against a library like `OpenSSL` or `BoringSSL` to handle the encryption/decryption.
*   The raw TCP bytes go through the TLS engine before they reach your WebSocket frame parser.

---

### Implementation Tips for JSI:
*   **Zero-Copy Parsing**: Use `std::string_view` or raw buffers in C++ to avoid unnecessary copying of large binary payloads.
*   **Memory Management**: Since JSI can pass `ArrayBuffer` directly between JS and C++, you can share the underlying memory for binary data, making it extremely fast.
