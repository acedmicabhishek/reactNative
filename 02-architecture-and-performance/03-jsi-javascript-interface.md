# JSI — JavaScript Interface: The Foundation of the New Architecture

## Overview
JSI (JavaScript Interface) is a **C++ API** that allows JavaScript code to directly interact with C++ objects without any serialization or bridge overhead. It is the foundational layer of the New Architecture, replacing the old JSON-based Bridge for native module communication.

---

## 1. What is JSI?

JSI is a thin C++ abstraction layer over a JavaScript engine. It provides a unified API to:
- **Execute JavaScript** code in an engine-agnostic way (works with Hermes, JSC, V8).
- **Expose C++ objects and functions** to the JavaScript runtime as **Host Objects**.
- **Call JS functions** from C++ without going through a queue.

**Key Concept: Host Objects**
A Host Object is a C++ class that implements the `jsi::HostObject` interface. When installed into the JS runtime, it appears as a regular JS object — but method calls on it invoke C++ code directly.

---

## 2. How JSI Replaces the Bridge

### Legacy Bridge (Before JSI)
```
JS: NativeModules.Camera.takePicture({ quality: 0.9 }, callback)
     → JSON serialize → Queue → JSON deserialize → Native method call
                                (async, batched, slow)
```

### JSI (After)
```
JS: globalThis.__cameraModule.takePicture({ quality: 0.9 })
     → Direct C++ function call (no serialization, no queue)
                                (synchronous or async, immediate)
```

The JS object `__cameraModule` is a C++ `jsi::HostObject`. When you call `.takePicture()`, it directly invokes a C++ function registered on that object.

---

## 3. Core JSI C++ API

```cpp
// Key types in the JSI API (from <jsi/jsi.h>):

// jsi::Runtime  — the JS engine runtime instance
// jsi::Value    — any JS value (undefined, bool, number, string, object, symbol, bigint)
// jsi::Object   — a JS object
// jsi::Function — a JS function
// jsi::HostObject — base class for C++ objects exposed to JS

// --- Example: Exposing a C++ object to JS ---

class MathModule : public jsi::HostObject {
public:
  // Called when JS accesses a property: obj.propName
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    auto methodName = name.utf8(rt);

    if (methodName == "add") {
      return jsi::Function::createFromHostFunction(
        rt,
        jsi::PropNameID::forAscii(rt, "add"),
        2,  // number of expected arguments
        [](jsi::Runtime& rt, const jsi::Value& thisVal, const jsi::Value* args, size_t count) {
          double a = args[0].asNumber();
          double b = args[1].asNumber();
          return jsi::Value(a + b);  // synchronous return!
        }
      );
    }
    return jsi::Value::undefined();
  }

  // Called when JS sets a property: obj.propName = value
  void set(jsi::Runtime& rt, const jsi::PropNameID& name, const jsi::Value& value) override {}
};

// Install into JS runtime
void installMathModule(jsi::Runtime& rt) {
  auto mathModule = std::make_shared<MathModule>();
  rt.global().setProperty(rt, "__mathModule",
    jsi::Object::createFromHostObject(rt, mathModule));
}
```

```js
// JS side — no imports needed, it's a global
const result = globalThis.__mathModule.add(3, 4); // = 7, synchronous!
```

---

## 4. How TurboModules Use JSI

TurboModules (the new native module system) use JSI under the hood:

1. A **TurboModule** is a C++ class that exposes methods via JSI.
2. The JS codegen tool auto-generates the `jsi::HostObject` wrapper from TypeScript types.
3. TurboModules are **lazily loaded** — the C++ object is only created when JS first accesses it (vs. the legacy architecture where all native modules were initialized on startup).

```typescript
// TurboModule spec (TypeScript) — codegen generates C++ bindings from this
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  takePicture(options: { quality: number }): Promise<string>;
  isAvailable(): boolean; // can be synchronous!
}

export default TurboModuleRegistry.getEnforcing<Spec>('NativeCameraModule');
```

The codegen generates:
- A C++ abstract class `NativeCameraModuleCxxSpec` with pure virtual methods.
- A JS JSI binding that calls those methods directly.

---

## 5. JSI and Shared Data

One of JSI's most powerful features is that **the same C++ object can be referenced from both JS and native simultaneously** — no copying.

```cpp
// A C++ HostObject wrapping a large data buffer
class ImageBuffer : public jsi::HostObject {
  std::vector<uint8_t> data;  // the actual pixel data — lives in C++ heap

public:
  ImageBuffer(std::vector<uint8_t>&& d) : data(std::move(d)) {}

  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    if (name.utf8(rt) == "byteLength") {
      return jsi::Value((int)data.size());
    }
    // Expose a method to read without copying
    if (name.utf8(rt) == "getPixel") {
      return jsi::Function::createFromHostFunction(rt, name, 1,
        [this](jsi::Runtime& rt, const jsi::Value&, const jsi::Value* args, size_t) {
          int index = (int)args[0].asNumber();
          return jsi::Value((int)data[index]);
        });
    }
    return jsi::Value::undefined();
  }
};
```

The pixel data `std::vector<uint8_t>` never leaves C++ memory. JS can read individual pixels without any copies.

---

## 6. JSI and Reanimated

React Native Reanimated uses JSI heavily:

```
Reanimated Worklet:
  JS code → serialized as a string → installed into the UI thread's separate JS runtime via JSI
  → runs directly on the UI thread (no bridge crossing per frame)

SharedValue:
  Backed by a C++ HostObject
  JS thread writes → directly updates C++ value
  UI thread worklet reads → directly reads C++ value
  (atomic access — no serialization)
```

This is why Reanimated achieves true 60/120fps animation: the C++ `SharedValue` is accessed directly from the UI thread's worklet without any JS-main-thread involvement per frame.

---

## 7. JSI vs. The Bridge: Mental Model

| Feature | Legacy Bridge | JSI |
|---------|--------------|-----|
| Communication | Async JSON queue | Direct C++ function calls |
| Synchronous calls | ❌ Not possible | ✅ Possible |
| Data passing | JSON copy | Direct C++ reference |
| Startup cost | All modules initialized | Lazy initialization |
| Engine coupling | Coupled to JSC | Engine-agnostic (Hermes, JSC, V8) |
| Memory sharing | ❌ No shared memory | ✅ Share C++ objects |

---

## 8. Building Your Own JSI Module

This is how you'd create a high-performance native module from scratch:

1. **Define the TypeScript spec** (for codegen).
2. **Write the C++ implementation** inheriting from the generated spec class.
3. **Register it** via the `ReactPackageTurboModuleManagerDelegate` (Android) or `RCTTurboModuleManagerDelegate` (iOS).
4. **Use it in JS** via `TurboModuleRegistry.get('YourModule')`.

This is the path to building **zero-overhead native integrations** — audio engines, ML inference, custom graphics renderers — that run at native speed while being callable from React Native JS.

---

## 9. Key Insight for Custom Engine Builders

JSI is the correct abstraction layer if you want to build a **custom rendering engine** or **performance-critical subsystem** for React Native:

- Write your core engine in C++ (OpenGL, Vulkan, Metal, audio DSP, ML).
- Expose a clean API as a `jsi::HostObject`.
- React Native JS drives your engine with zero serialization overhead.
- Your engine can run on its own dedicated thread, posting results back via `rt.queueMicrotask()` or a JS callback.

This is exactly how libraries like **VisionCamera**, **Reanimated**, and **MMKV** achieve their performance.
