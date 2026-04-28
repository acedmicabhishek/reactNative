# JSI C++ Native Socket Strategy

This is the blueprint for building a high-performance Socket.IO-like library that lives in C++ and interacts with React Native via JSI. The goal is zero JS-thread blocking.

## 1. Threading Architecture
This is the most important part. You need two main threads:

1.  **Networking Thread (C++)**:
    *   Runs an event loop (`uv_loop_t` if using libuv, or a custom `epoll` loop).
    *   Handles TCP/TLS/WebSocket framing.
    *   Manages the Socket.IO packet parser.
    *   Queues incoming events.
2.  **JS Thread (The React Native loop)**:
    *   Only receives a "ping" when a full packet is ready.
    *   Executes the JS callback.

## 2. JSI HostObject Design
Expose a single object to JS that represents the socket.

**C++ Interface (Simplified):**
```cpp
class SocketJSI : public jsi::HostObject {
public:
    // JS: socket.emit('event', data)
    void emit(jsi::Runtime& rt, const std::string& event, const jsi::Value& data);

    // JS: socket.on('event', callback)
    void on(const std::string& event, jsi::Function callback);

private:
    std::thread networkThread_;
    std::map<std::string, std::vector<jsi::Function>> listeners_;
    // CallInvoker is needed to execute JS functions from the background thread
    std::shared_ptr<react::CallInvoker> jsInvoker_;
};
```

## 3. The `CallInvoker` Pattern
C++ cannot just call a `jsi::Function` from a background thread. It will crash. You must use `CallInvoker` to schedule the function to run on the JS thread.

**Dispatching an event to JS:**
```cpp
void onMessageReceived(std::string event, std::string payload) {
    jsInvoker_->invokeAsync([this, event, payload]() {
        // Now we are on the JS thread!
        jsi::Runtime& rt = getRuntime();
        jsi::Value data = jsi::Value::createFromJsonUtf8(rt, (const uint8_t*)payload.c_str(), payload.length());
        
        for (auto& callback : listeners_[event]) {
            callback.call(rt, data);
        }
    });
}
```

## 4. Native Library Recommendations
Building a WebSocket parser from scratch is hard. Use these:

*   **[uWebSockets](https://github.com/uNetworking/uWebSockets)**: The fastest WebSocket library in existence.
*   **[libwebsockets](https://libwebsockets.org/)**: Very stable, used by many embedded systems.
*   **[nlohmann/json](https://github.com/nlohmann/json)**: For parsing Socket.IO JSON packets in C++.
*   **[simdjson](https://simdjson.org/)**: If you have massive JSON payloads and need extreme speed.

## 5. End-to-End Flow for Your Implementation

1.  **Initialize**: On app start, JSI installs the `NativeSocket` HostObject.
2.  **Connect**: JS calls `NativeSocket.connect("url")`. C++ starts the background thread.
3.  **Handshake**: C++ thread performs TCP sync, TLS handshake, and HTTP Upgrade.
4.  **Receive**:
    *   Data arrives on TCP socket.
    *   C++ thread unmasks the WS frame.
    *   C++ thread parses the Engine.IO / Socket.IO headers.
    *   C++ thread parses the JSON body into a `HostObject` or passes it as a string.
5.  **Dispatch**: C++ uses `CallInvoker` to trigger the JS listener.

## 6. Pro-Tip: Zero-Copy Binary
For images or sensor data, use `jsi::ArrayBuffer`.
1.  C++ receives raw binary from socket.
2.  C++ wraps the buffer in a `jsi::ArrayBuffer`.
3.  JS receives the buffer directly. No base64 encoding, no string conversion.

---

### Why this is "High Tier":
By following this, your networking layer will be faster than anything built into the standard React Native framework. You are effectively building a custom browser engine's networking stack tailored for your app.
