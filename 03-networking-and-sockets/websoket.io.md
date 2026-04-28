# Socket.IO Client API Implementation Checklist

To build `@matiks/nativesocketio` as a drop-in replacement for the standard `socket.io-client`, you need to expose a specific set of JavaScript APIs that map to your underlying C++ JSI core.

Here is the master list of functions and properties you need to implement to achieve API parity.

---

## 1. Initialization & Connection

The entry point for the client.

- [ ] `io(url, options)`
  - **Purpose**: Creates and returns a new Socket instance.
  - **JSI Map**: Instantiates your `NativeSocketHostObject` in C++.
  - **Options to handle**: `transports` (force `['websocket']`), `query`, `auth`, `reconnection`, `reconnectionAttempts`, `reconnectionDelay`.

- [ ] `socket.connect()` (or `socket.open()`)
  - **Purpose**: Manually opens the socket (if `autoConnect` was false).
  - **JSI Map**: Tells C++ thread to initiate TCP/TLS handshake.

- [ ] `socket.disconnect()` (or `socket.close()`)
  - **Purpose**: Manually disconnects the socket.
  - **JSI Map**: Tells C++ to send a Close frame and tear down the socket loop.

---

## 2. Event Listening (Receiving Data)

How the JS thread registers callbacks for events coming from the server.

- [ ] `socket.on(eventName, callback)`
  - **Purpose**: Listens for a specific event.
  - **JSI Map**: Adds the `jsi::Function` to a map in C++ associated with the `eventName`.

- [ ] `socket.once(eventName, callback)`
  - **Purpose**: Listens for an event, but automatically removes the listener after it fires once.
  - **JSI Map**: Can be implemented purely in JS wrapper logic, wrapping the callback to call `socket.off()` after execution.

- [ ] `socket.off(eventName, callback)` (or `socket.removeListener`)
  - **Purpose**: Removes a specific listener.
  - **JSI Map**: Removes the specific `jsi::Function` reference in C++.

- [ ] `socket.removeAllListeners([eventName])`
  - **Purpose**: Removes all listeners for a specific event, or ALL events if no name is provided.
  - **JSI Map**: Clears the corresponding lists in the C++ map.

---

## 3. Event Emission (Sending Data)

How the JS thread sends data to the server.

- [ ] `socket.emit(eventName, ...args)`
  - **Purpose**: Sends an event to the server. Can include multiple arguments.
  - **JSI Map**: C++ serializes the arguments into `42["eventName", arg1, arg2]` and sends it over the wire.

- [ ] `socket.emit(eventName, ...args, ackCallback)` *(Acknowledgements)*
  - **Purpose**: Sends an event and waits for the server to call the acknowledgment callback.
  - **JSI Map**: C++ generates a unique `PacketID`, sends `42[PacketID, "eventName", ...]`, and stores the `jsi::Function` waiting for an Engine.IO Ack response with that ID.

---

## 4. Built-in System Events

Your C++ core must automatically trigger these standard Socket.IO system events so the JS code knows the socket state.

- [ ] `"connect"`
  - Fired upon a successful connection and successful Socket.IO handshake.
- [ ] `"disconnect"`
  - Fired upon a disconnection (includes the `reason` string).
- [ ] `"connect_error"`
  - Fired when a connection fails (includes the Error object/string).
- [ ] `"ping"` / `"pong"` (Optional)
  - Exposed if the user wants to measure latency. (Engine.IO layer handles this).

---

## 5. Properties & State

Read-only properties that JS can check at any time.

- [ ] `socket.id`
  - **Purpose**: A unique identifier for the socket session (assigned by the server during the handshake).
  - **JSI Map**: Read from a C++ string variable.

- [ ] `socket.connected`
  - **Purpose**: Boolean indicating if the socket is currently connected.
  - **JSI Map**: Read from a C++ boolean state.

- [ ] `socket.disconnected`
  - **Purpose**: The inverse of `connected`.

---

## 6. Advanced Features (Optional for V1)

If you want 100% full compatibility, you will eventually need these.

- [ ] `socket.volatile.emit(...)`
  - **Purpose**: Sends a packet that will *not* be buffered or retried if the connection is currently down.
- [ ] `socket.compress(false).emit(...)`
  - **Purpose**: Explicitly disables per-message compression for a specific emit.
- [ ] **Binary Support (ArrayBuffers)**
  - **Purpose**: Allowing `socket.emit("file", new ArrayBuffer(100))` without stringifying.
  - **JSI Map**: Detect `jsi::ArrayBuffer` in the arguments, extract the raw pointer, and construct a binary Socket.IO frame.

---

## Example API Wrapper (TypeScript)

This is how your JS facade will wrap the raw JSI calls:

```typescript
// Inside your NPM package:
const NativeSocketIO = global.NativeSocketIO; // The C++ JSI Object

export class Socket {
  public id: string | undefined;
  public connected: boolean = false;

  constructor(url: string, options: any) {
    this.native = NativeSocketIO.createConnection(url, options);
    
    // Wire up internal state listeners
    this.native.on('connect', (id) => {
      this.connected = true;
      this.id = id;
    });
    this.native.on('disconnect', () => {
      this.connected = false;
    });
  }

  on(event: string, callback: Function) {
    this.native.on(event, callback);
  }

  emit(event: string, ...args: any[]) {
    this.native.emit(event, args);
  }
}
```
