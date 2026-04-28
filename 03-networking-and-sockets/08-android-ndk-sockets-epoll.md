# Android NDK Sockets & epoll

To build a non-blocking C++ socket core for an APK, you need to use the Linux-based networking APIs provided by the Android NDK.

## 1. The Standard `socket` API
In the NDK, sockets are just file descriptors.

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <fcntl.h>

int fd = socket(AF_INET, SOCK_STREAM, 0);

// Make it non-blocking!
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

## 2. Why `epoll`?
If you have multiple sockets (e.g., main connection + image stream + logs), you don't want a thread for each. `epoll` (Event Poll) is the Linux kernel's most efficient way to monitor thousands of sockets simultaneously.

*   **`epoll_create1()`**: Create an epoll instance.
*   **`epoll_ctl()`**: Add your socket to the interest list.
*   **`epoll_wait()`**: This is your "Event Loop" heart. It sleeps efficiently and wakes up the instant data arrives.

## 3. The Android Network Permissions
Even though you are in C++, you are still inside an Android App.
*   **`AndroidManifest.xml`**: You MUST have `<uses-permission android:name="android.permission.INTERNET" />`.
*   **Network Security Config**: Android 9+ blocks cleartext (HTTP) traffic by default. If your WebSocket isn't `wss://`, you must explicitly allow it in your XML config.

## 4. `ADataLink` and Android Connectivity
For high-tier apps, you should know when the user switches from Wi-Fi to 4G.
*   In C++, you can't easily listen to system broadcasts.
*   **Strategy**: Use a small Kotlin class to listen for connectivity changes and call a `native void` function in your C++ core to trigger a reconnection or a "Transport Switch".

## 5. DNS Resolution
Native `getaddrinfo()` is blocking.
*   If you call it on the JS thread, the UI will freeze for 2 seconds if the DNS is slow.
*   **Native Solution**: Use a library like **c-ares** for non-blocking DNS resolution in C++, or perform the resolution on your C++ background thread before starting the connection.

## 6. Keeping the App Alive
Android aggressively kills background processes.
*   If your C++ socket is active, you might need a **Foreground Service** in Android to keep the system from killing your app's process and severing the connection.

---

### End-to-End Native Stack:
1.  **JSI Layer**: JS triggers connection.
2.  **Logic Layer**: C++ manages state machines (Connecting -> Open -> Closing).
3.  **Transport Layer**: C++ uses `epoll` + `socket` + `BoringSSL`.
4.  **Hardware**: Android Kernel handles the actual radio/NIC.

By mastering this stack, you are no longer a "React Native Developer"—you are a **Native Systems Engineer** using React for the UI.
