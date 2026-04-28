# TLS/SSL Handshake & Native Encryption

To support `wss://` (secure WebSockets), you must understand the TLS layer. In a C++ JSI implementation, you won't use the high-level browser API; you'll likely interface with **BoringSSL** (Android's default) or **OpenSSL**.

## 1. The Secure Handshake (The "Extra" 4 Steps)
After the TCP 3-way handshake, but *before* the HTTP Upgrade, a TLS handshake occurs:

1.  **Client Hello**: Client sends supported cipher suites and a random number.
2.  **Server Hello**: Server chooses a cipher, sends its Certificate (Public Key), and another random number.
3.  **Key Exchange**: Client verifies the certificate. It then encrypts a "Pre-master secret" with the server's public key and sends it back.
4.  **Finished**: Both sides derive "Session Keys" for symmetric encryption. All future data is now encrypted.

## 2. Impact on Performance
*   **Latency**: The handshake adds 2-3 round trips *before* the first WebSocket byte is sent.
*   **CPU**: Sockets using TLS require significant CPU for encryption/decryption. Doing this on the JS thread is a disaster; doing it in a C++ background thread is essential.

## 3. Implementing in C++ (OpenSSL/BoringSSL)
You treat the TLS engine as a **Bio-Filter**.

```cpp
// Pseudocode for TLS Buffer flow
SSL* ssl = SSL_new(ctx);
SSL_set_fd(ssl, socket_fd);

// Reading secure data:
// Raw Socket -> SSL_read() -> Decrypted Buffer -> WebSocket Parser
char buffer[4096];
int bytes = SSL_read(ssl, buffer, sizeof(buffer));

// Writing secure data:
// WebSocket Frame -> SSL_write() -> Encrypted Buffer -> Raw Socket
SSL_write(ssl, frame_data, frame_length);
```

## 4. Certificate Pinning (Native Advantage)
When building a custom C++ JSI socket, you can implement **Certificate Pinning** much more securely than in standard JS.
*   You hardcode the server's public key hash in your C++ code.
*   During the TLS handshake, if the server's cert doesn't match the hash exactly, you kill the connection.
*   This prevents **Man-in-the-Middle (MITM)** attacks even if the user has a compromised Root CA.

## 5. ALPN (Application-Layer Protocol Negotiation)
Modern servers use ALPN to decide whether to speak HTTP/1.1 or HTTP/2. For WebSockets, you ensure your C++ TLS config includes `http/1.1` to ensure the upgrade succeeds.

---

### Implementation Note for Android:
Android ships with `BoringSSL`. You can link against it in your `CMakeLists.txt` using the NDK's `libcrypto.so` and `libssl.so`. This keeps your APK size small since you aren't bundling a separate copy of OpenSSL.
