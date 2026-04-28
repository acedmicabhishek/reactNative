# High-Performance Serialization (Beyond JSON)

In a high-tier C++ JSI socket, **JSON is your enemy**. `JSON.parse` in JS and `json::parse` in C++ are CPU-intensive and create thousands of small objects. For 60fps data, use binary serialization.

## 1. Why JSON Fails
*   **Text-based**: Numbers like `123.456` take 7 bytes as text but only 4 or 8 bytes as binary.
*   **Parsing overhead**: The computer has to scan strings, handle escapes, and build a tree.
*   **Memory Fragmentation**: Each key/value in a JSON object is a separate allocation.

## 2. MessagePack (The "Binary JSON")
MessagePack is a drop-in replacement for JSON that is binary.
*   **Pros**: Smaller than JSON, very fast C++ libraries (`msgpack-c`).
*   **Cons**: Still requires "parsing" into an object structure.

## 3. Protocol Buffers (Protobuf)
Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data.
*   **Pros**: Extremely small, very fast, strongly typed.
*   **Cons**: Requires a `.proto` schema file and a compilation step.
*   **JSI Strategy**: Generate C++ code for your messages. C++ parses the binary and only exposes the specific fields JS needs via JSI getters.

## 4. FlatBuffers (The "Zero-Copy" King)
Used by Facebook and for high-end mobile games.
*   **The Magic**: You can access data *without parsing or unpacking*.
*   **How**: It stores data in a format that is memory-aligned and ready to be read.
*   **JSI Flow**:
    1.  C++ receives a binary blob.
    2.  C++ passes the blob to JS as an `ArrayBuffer`.
    3.  JS uses a tiny wrapper to read specific offsets. **Zero CPU time spent on parsing.**

## 5. Implementation Comparison

| Format | Parsing Speed | Size | Ease of Use |
|--------|--------------|------|-------------|
| **JSON** | Slow | Large | Easiest |
| **MessagePack** | Fast | Small | Easy |
| **Protobuf** | Very Fast | Tiny | Moderate |
| **FlatBuffers** | Instant | Medium | Hard |

## 6. Recommendation for your JSI Core
If you are building the "Ultimate" Socket.IO replacement:
*   Use **MessagePack** for general events (replaces standard Socket.IO JSON).
*   Use **FlatBuffers** for heavy data streams (position updates, telemetry, image metadata).

In C++, you can detect the first byte of a packet to decide which parser to use.

---

### Pro Tip:
Socket.IO natively supports binary attachments. When you emit `socket.emit('event', arrayBuffer)`, Socket.IO doesn't base64 it; it sends a separate binary frame. In your C++ core, make sure you handle these "Placeholder" packets to reconstruct the data for JS.
