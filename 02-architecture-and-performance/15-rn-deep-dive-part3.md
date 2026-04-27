# React Native Deep Dive — Part 3: dlopen, CMake, E2E Flow & Pitfalls

---

## 8. Loading `.so` Files at Runtime (dlopen / dlsym)

### What `.so` Files Are

A `.so` (shared object) is a compiled native library — the Linux/Android equivalent of a Windows `.dll`. It contains machine code (ARM64, x86_64, etc.) that can be loaded and executed at runtime.

Android sandboxes each app under `/data/user/0/<package.name>/`. Your app has full read/write access to files inside it. You can download a `.so` into `files/` and load it dynamically.

### The Three Functions

```
dlopen(path, flags)  → void*  (handle to loaded library, NULL on failure)
dlsym(handle, name)  → void*  (pointer to function/symbol, NULL on failure)
dlclose(handle)      → int    (0 on success)
dlerror()            → char*  (last error string, NULL if no error)
```

### C++ Code — Full dlopen/dlsym Example

```cpp
// DynamicLoader.cpp
#include <jsi/jsi.h>
#include <dlfcn.h>       // dlopen, dlsym, dlclose, dlerror
#include <string>
#include <unordered_map>
#include <android/log.h>

#define TAG "DynamicLoader"
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, TAG, __VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,  TAG, __VA_ARGS__)

using namespace facebook::jsi;

// Track open handles so we can dlclose later
static std::unordered_map<std::string, void*> openHandles;

// ─── Install all functions into JS runtime ───────────────────────────────────
void installDynamicLoader(Runtime& rt) {

    // ── loadLibrary(path: string): string ─────────────────────────────────
    // Opens a .so from the given absolute path.
    // Returns "ok" on success, or an error string.
    auto loadLibrary = Function::createFromHostFunction(
        rt,
        PropNameID::forAscii(rt, "loadLibrary"),
        1,
        [](Runtime& rt, const Value&, const Value* args, size_t count) -> Value {

            if (count < 1 || !args[0].isString()) {
                throw JSError(rt, "loadLibrary(path: string)");
            }

            std::string path = args[0].asString(rt).utf8(rt);

            // Clear any previous error
            dlerror();

            // RTLD_NOW  — resolve all symbols immediately (find broken deps early)
            // RTLD_LOCAL — symbols don't bleed into other loaded libs
            void* handle = dlopen(path.c_str(), RTLD_NOW | RTLD_LOCAL);

            if (!handle) {
                // dlerror() returns the reason: wrong ABI, missing deps, bad path, etc.
                const char* err = dlerror();
                LOGE("dlopen failed: %s", err ? err : "unknown");
                std::string msg = "dlopen failed: ";
                msg += (err ? err : "unknown error");
                return String::createFromUtf8(rt, msg);
            }

            LOGI("Loaded: %s", path.c_str());
            openHandles[path] = handle; // save handle for later dlsym calls
            return String::createFromUtf8(rt, "ok");
        }
    );

    // ── callFunction(libPath: string, symbol: string, arg: number): number ──
    // Looks up a symbol in an already-loaded library and calls it.
    // This example assumes the function signature: double fn(double)
    auto callFunction = Function::createFromHostFunction(
        rt,
        PropNameID::forAscii(rt, "callFunction"),
        3,
        [](Runtime& rt, const Value&, const Value* args, size_t count) -> Value {

            if (count < 3 || !args[0].isString() || !args[1].isString() || !args[2].isNumber()) {
                throw JSError(rt, "callFunction(libPath, symbol, arg)");
            }

            std::string libPath = args[0].asString(rt).utf8(rt);
            std::string symbol  = args[1].asString(rt).utf8(rt);
            double arg          = args[2].asNumber();

            // Find the handle
            auto it = openHandles.find(libPath);
            if (it == openHandles.end()) {
                throw JSError(rt, "Library not loaded: " + libPath);
            }

            void* handle = it->second;

            // Clear error state before dlsym
            dlerror();

            // Look up the symbol — returns a raw function pointer
            void* sym = dlsym(handle, symbol.c_str());

            const char* err = dlerror();
            if (err) {
                LOGE("dlsym failed for %s: %s", symbol.c_str(), err);
                throw JSError(rt, std::string("dlsym failed: ") + err);
            }

            // Cast the void* to the expected function signature
            // YOU must know the function signature — no runtime type info in C
            using FnType = double(*)(double);
            FnType fn = reinterpret_cast<FnType>(sym);

            double result = fn(arg);
            LOGI("Called %s(%f) = %f", symbol.c_str(), arg, result);

            return Value(result);
        }
    );

    // ── closeLibrary(libPath: string): string ─────────────────────────────
    auto closeLibrary = Function::createFromHostFunction(
        rt,
        PropNameID::forAscii(rt, "closeLibrary"),
        1,
        [](Runtime& rt, const Value&, const Value* args, size_t count) -> Value {

            if (count < 1 || !args[0].isString()) {
                throw JSError(rt, "closeLibrary(path: string)");
            }

            std::string path = args[0].asString(rt).utf8(rt);
            auto it = openHandles.find(path);

            if (it == openHandles.end()) {
                return String::createFromUtf8(rt, "not loaded");
            }

            dlclose(it->second);
            openHandles.erase(it);
            return String::createFromUtf8(rt, "closed");
        }
    );

    // Install all three on a single namespace object: global.DynLoader
    auto nsObj = Object(rt);
    nsObj.setProperty(rt, "loadLibrary",  std::move(loadLibrary));
    nsObj.setProperty(rt, "callFunction", std::move(callFunction));
    nsObj.setProperty(rt, "closeLibrary", std::move(closeLibrary));
    rt.global().setProperty(rt, "DynLoader", std::move(nsObj));
}
```

### JS Side Usage

```ts
// dl.ts — typed wrapper around the C++ globals
const PACKAGE = 'com.myapp';
const LIB_DIR = `/data/user/0/${PACKAGE}/files/libs`;

export async function loadAndCall(libName: string, fnName: string, arg: number) {
  const path = `${LIB_DIR}/${libName}`;

  // loadLibrary is synchronous via JSI
  const loadResult = (global as any).DynLoader.loadLibrary(path);

  if (loadResult !== 'ok') {
    throw new Error(`Failed to load ${libName}: ${loadResult}`);
  }

  // callFunction is also synchronous
  const result = (global as any).DynLoader.callFunction(path, fnName, arg);
  return result;
}

// Usage
const output = await loadAndCall('libmymath.so', 'computeSquare', 9);
console.log(output); // 81.0
```

### ABI Compatibility

Android devices can run multiple ABIs. You must ship the `.so` for the right one:

| ABI | Device type |
|-----|------------|
| `arm64-v8a` | Modern 64-bit Android (most phones) |
| `armeabi-v7a` | Older 32-bit ARM |
| `x86_64` | Emulator (64-bit) |
| `x86` | Old emulator (32-bit) |

Get the device ABI at runtime:

```kotlin
import android.os.Build
val abi = Build.SUPPORTED_ABIS[0] // e.g. "arm64-v8a"
```

Then download/store the correct `.so` for that ABI.

**dlopen error meanings:**

| Error message | Likely cause |
|--------------|-------------|
| `No such file or directory` | Wrong path or file not downloaded yet |
| `Permission denied` | SELinux policy blocking execution |
| `ELF file ABI mismatch` | Wrong ABI — arm64 lib on x86 device |
| `cannot locate symbol` in `dlsym` | Symbol not exported or name mangled (C++ needs `extern "C"`) |

---

## 9. CMake Setup for React Native C++

### Full `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.13)
project(DynamicLoader CXX)

set(CMAKE_CXX_STANDARD 17)

# Path to react-native in node_modules (relative to this CMakeLists.txt)
set(RN_DIR ${CMAKE_SOURCE_DIR}/../../../../../node_modules/react-native)

# ── Define our shared library ──────────────────────────────────────────────
add_library(
    DynamicLoader    # produces libDynamicLoader.so
    SHARED
    DynamicLoader.cpp
    OnLoad.cpp
)

# ── Headers ───────────────────────────────────────────────────────────────
target_include_directories(DynamicLoader PRIVATE
    ${RN_DIR}/ReactCommon/jsi
    ${RN_DIR}/ReactCommon/callinvoker
    ${RN_DIR}/ReactAndroid/src/main/jni/react/turbomodule
    ${CMAKE_SOURCE_DIR}
)

# ── Link libraries ────────────────────────────────────────────────────────
target_link_libraries(DynamicLoader
    android          # Core Android JNI support
    log              # Logging (__android_log_print)
    dl               # dlopen, dlsym, dlclose — THIS IS REQUIRED
)

# Note: jsi is usually provided via RN's prefab package.
# In newer RN versions, add this before add_library:
# find_package(ReactAndroid REQUIRED CONFIG)
# then link: ReactAndroid::jsi
```

### `build.gradle` (Module-level)

```gradle
android {
    compileSdkVersion 34

    defaultConfig {
        minSdkVersion 24
        targetSdkVersion 34

        ndk {
            // Only build ABIs you actually need
            abiFilters "arm64-v8a", "x86_64"
        }

        externalNativeBuild {
            cmake {
                cppFlags "-std=c++17 -fexceptions -frtti"
                arguments "-DANDROID_STL=c++_shared",
                          "-DANDROID_PLATFORM=android-24"
            }
        }
    }

    externalNativeBuild {
        cmake {
            path "src/main/cpp/CMakeLists.txt"
            version "3.22.1"
        }
    }
}

dependencies {
    implementation "com.facebook.react:react-android"
    implementation "com.facebook.react:hermes-android"
}
```

### Directory Structure

```
android/app/src/main/
├── cpp/
│   ├── CMakeLists.txt
│   ├── DynamicLoader.cpp     ← JSI functions + dlopen logic
│   ├── DynamicLoader.h
│   └── OnLoad.cpp            ← JNI_OnLoad + install entry point
├── java/com/myapp/
│   ├── MainApplication.kt
│   ├── DynamicLoaderModule.kt  ← Kotlin wrapper that calls nativeInstall
│   └── DynamicLoaderPackage.kt
└── jniLibs/
    └── arm64-v8a/
        └── (third-party .so files that must ship with the APK)
```

---

## 10. Full End-to-End Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Write C++ (DynamicLoader.cpp)                           │
│   • dlopen/dlsym logic wrapped in jsi::Function lambdas         │
│   • installDynamicLoader(Runtime&) sets up global.DynLoader     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│ Step 2: CMakeLists.txt                                          │
│   • Links DynamicLoader.cpp against android + log + dl          │
│   • Outputs libDynamicLoader.so                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│ Step 3: build.gradle                                            │
│   • externalNativeBuild points to CMakeLists.txt                │
│   • abiFilters limits which ABIs get compiled                   │
│   • Gradle runs CMake during build → .so ends up in APK         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│ Step 4: App Startup                                             │
│   DynamicLoaderModule.install() is called from JS (or auto):   │
│     System.loadLibrary("DynamicLoader")                         │
│     nativeInstall(jsiRuntimeRef) ← JNI call                     │
│     → installDynamicLoader(runtime) runs                        │
│     → global.DynLoader is now available in JS                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│ Step 5: JS calls DynLoader.loadLibrary(path)                    │
│   • Synchronous JSI call → C++ lambda runs                      │
│   • dlopen('/data/user/0/.../mylib.so', RTLD_NOW | RTLD_LOCAL)  │
│   • Handle saved in openHandles map                             │
│   • Returns "ok" or error string to JS                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│ Step 6: JS calls DynLoader.callFunction(path, 'myFn', 9.0)      │
│   • C++ looks up handle from openHandles                        │
│   • dlsym(handle, "myFn") → raw function pointer               │
│   • Cast to double(*)(double) and call it                       │
│   • Result returned synchronously as jsi::Value to JS           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. Common Pitfalls and Debugging

### ABI Mismatch

```
dlopen failed: "/data/.../mylib.so": ELF file ABI mismatch
```

You built your `.so` for `arm64-v8a` but the device or emulator is `x86_64`. Always check `Build.SUPPORTED_ABIS[0]` and download/use the matching library.

### dlopen Returns NULL

```cpp
void* handle = dlopen(path, RTLD_NOW | RTLD_LOCAL);
if (!handle) {
    const char* err = dlerror();
    // Log err — it's the ONLY way to know why
}
```

Common reasons:
- Wrong path (use absolute paths from `context.filesDir`)
- File not yet downloaded
- ABI mismatch
- Missing transitive dependencies (the `.so` itself links against something not present)
- SELinux denial — check `adb logcat | grep avc`

### JSI Type Crashes

```cpp
// WRONG — crashes if arg is not a string
std::string s = args[0].asString(rt).utf8(rt);

// RIGHT — always check first
if (!args[0].isString()) {
    throw JSError(rt, "expected string");
}
std::string s = args[0].asString(rt).utf8(rt);
```

### Memory Ownership

`jsi::Value` is value-type — copy it if you need it beyond the current scope:

```cpp
// WRONG — args is a pointer to a stack array, invalid after the lambda returns
Value* saved = args; // dangling pointer

// RIGHT
Value copy = Value(rt, args[0]); // deep copy
```

### C++ Symbol Mangling

If your loaded `.so` has C++ functions, they have mangled names. Use `extern "C"` in the library to keep names plain:

```cpp
// In your loadable .so
extern "C" {
    double computeSquare(double x) { return x * x; }
}
```

Without `extern "C"`, `dlsym(handle, "computeSquare")` returns NULL because the actual symbol is `_Z13computeSquared`.

### Debugging Native Crashes

```bash
# Stream native logs and filter for crashes
adb logcat | grep -iE "fatal|jni|dlopen|signal|crash|SIGSEGV"

# Get a tombstone after a crash (more detail)
adb shell ls /data/tombstones/
adb pull /data/tombstones/tombstone_00
```

In Android Studio: **Run → Debug** with a **Native** debug configuration. Add breakpoints in `.cpp` files. LLDB will attach and show the C++ call stack on crash.

---

## 12. Glossary

| Term | Definition |
|------|-----------|
| **AST** | Abstract Syntax Tree — tree representation of source code structure, used by Babel to analyze and transform code |
| **JSI** | JavaScript Interface — C++ layer in React Native that lets JS call native C++ functions synchronously without JSON serialization |
| **TurboModule** | New-arch native module that is lazily loaded, strongly typed via Codegen, and communicates via JSI |
| **Fabric** | New React Native renderer — manages the UI shadow tree in C++ and syncs to native views via JSI |
| **Codegen** | Tool that reads TypeScript spec files and auto-generates C++/Java/ObjC bridge boilerplate |
| **Bridge** | Old RN mechanism for JS↔native communication — async, JSON-serialized message queue (replaced by JSI) |
| **.so file** | Shared Object — compiled native library for Linux/Android, loaded at runtime like a DLL on Windows |
| **dlopen** | C function that loads a `.so` file from a filesystem path into the process memory at runtime |
| **ABI** | Application Binary Interface — defines calling conventions, data layout, and instruction set; must match between caller and callee |
| **CMake** | Cross-platform build system that generates native build files (Makefiles, Ninja) for C/C++ code |
| **Hermes** | Meta's JavaScript engine for React Native — compiles JS to bytecode at build time for faster startup |
| **Metro** | React Native's JavaScript bundler — resolves, transforms, and serializes JS modules into a single bundle |
| **Babel preset** | A named collection of Babel plugins for a common transformation goal (e.g., all transforms needed for React Native) |
| **Native Module** | Old-arch mechanism for calling platform-specific Java/ObjC code from JS via the Bridge |
| **Host Object (JSI)** | A C++ class that implements `jsi::HostObject` and appears as a plain JS object — property access calls C++ |
| **jsi::Runtime** | Core JSI object representing the JS engine — used to create values, access globals, and evaluate code |
