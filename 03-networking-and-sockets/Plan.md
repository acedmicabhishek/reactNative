# NativeSocketIO

## `@matiks/nativesocketio`
### Key Architectural Advantages:
*   **Off-Thread Processing**: All networking and protocol logic runs on a background native thread.
*   **Zero-Copy Data Transfer**: Using JSI to share memory between C++ and JS, eliminating expensive data serialization/deserialization.
*   **Synchronous Execution**: JSI allows for faster, synchronous communication between the JS API and the native core.
*   **Full Compatibility**: The native core will strictly adhere to the Socket.IO 4.x protocol, ensuring it works out-of-the-box with our current Node.js servers.

---

## Implementation
note : we will not build tcp layer with scratch
### Native Infrastructure & Environment
*   Initialize an Expo Module with a C++ JSI backend.
*   Configure the Android NDK build system (CMake) to link against optimized C++ networking libraries (e.g., `libwebsockets` or `IXWebSocket`).

### Native Threading & Protocol Logic
*   **Dedicated Event Loop**: Implement a non-blocking background thread to handle raw TCP/TLS and WebSocket framing.
*   **Protocol Parser**: Implement the Engine.IO and Socket.IO state machines in C++ to handle handshakes, heartbeats, and packet decoding off-thread.

### JSI Integration (The Bridgehead)
*   Expose a `HostObject` to the JS environment.
*   Implement `CallInvoker` logic to safely dispatch network events from the background thread to JS callbacks without causing thread safety issues.

### API Layer
*   Develop a TypeScript wrapper that replicates the official `NativeSocketIO` API.
*   **Migration Path**: Developers can switch to the high-performance core by simply updating their import statements.