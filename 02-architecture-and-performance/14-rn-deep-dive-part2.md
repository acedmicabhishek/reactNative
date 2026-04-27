# React Native Deep Dive — Part 2: Native Modules, TurboModules & JSI

---

## 5. Native Modules (Old Architecture)

A Native Module is a bridge between JS and platform-specific code you write in Kotlin/Java or Swift/ObjC. You need one when React Native has no built-in API — Bluetooth, flashlight, custom SDKs, device sensors with special access, etc.

### Android — Kotlin Native Module

```kotlin
// MyModule.kt
package com.myapp

import com.facebook.react.bridge.*

class MyModule(private val reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    // This is the name JS uses: NativeModules.MyModule
    override fun getName() = "MyModule"

    // Constants available synchronously in JS (no async call needed)
    override fun getConstants(): Map<String, Any> {
        return mapOf("VERSION" to "1.0.0")
    }

    // @ReactMethod marks a function as callable from JS
    // promises give you async await on the JS side
    @ReactMethod
    fun multiply(a: Double, b: Double, promise: Promise) {
        try {
            promise.resolve(a * b)
        } catch (e: Exception) {
            promise.reject("MULTIPLY_ERROR", e.message, e)
        }
    }

    // Callback style (older pattern, prefer promises)
    @ReactMethod
    fun greet(name: String, callback: Callback) {
        callback.invoke("Hello, $name")
    }

    // Events — push data to JS without JS calling you
    fun sendEvent(eventName: String, params: WritableMap) {
        reactContext
            .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
            .emit(eventName, params)
    }
}
```

```kotlin
// MyPackage.kt
package com.myapp

import com.facebook.react.ReactPackage
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.uimanager.ViewManager

class MyPackage : ReactPackage {
    override fun createNativeModules(ctx: ReactApplicationContext) =
        listOf(MyModule(ctx))

    override fun createViewManagers(ctx: ReactApplicationContext) =
        emptyList<ViewManager<*, *>>()
}
```

```kotlin
// MainApplication.kt — register your package
override fun getPackages(): List<ReactPackage> =
    PackageList(this).packages.apply {
        add(MyPackage())
    }
```

### iOS — ObjC Native Module

```objc
// MyModule.h
#import <React/RCTBridgeModule.h>

@interface MyModule : NSObject <RCTBridgeModule>
@end

// MyModule.m
#import "MyModule.h"

@implementation MyModule

RCT_EXPORT_MODULE(MyModule); // JS name

RCT_EXPORT_METHOD(multiply:(double)a
                  b:(double)b
                  resolve:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)
{
    resolve(@(a * b));
}

@end
```

### JS Side

```js
import { NativeModules } from 'react-native';

const { MyModule } = NativeModules;

async function run() {
  const result = await MyModule.multiply(6, 7);
  console.log(result); // 42
  console.log(MyModule.VERSION); // "1.0.0"
}
```

**Common mistakes:**
- Forgetting `@ReactMethod` — function won't be exposed to JS
- Calling UI operations from a non-UI thread — use `runOnUiThread { }` in Kotlin
- Events: JS must add a listener via `NativeEventEmitter` or events are silently dropped

---

## 6. TurboModules (New Architecture)

### TypeScript Spec File

The spec is the source of truth. Codegen reads it and generates all the C++ glue.

```ts
// NativeMyModule.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  multiply(a: number, b: number): Promise<number>;
  getVersion(): string; // synchronous return
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

Rules for spec files:
- Only use types Codegen understands: `string`, `number`, `boolean`, `Object`, `Array`, `Promise<T>`
- Synchronous methods return the value directly (no Promise)
- File must be named `Native<ModuleName>.ts`

### Kotlin TurboModule

```kotlin
// MyModule.kt
package com.myapp

import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.Promise
import com.myapp.NativeMyModuleSpec // Codegen generated this

class MyModule(ctx: ReactApplicationContext) : NativeMyModuleSpec(ctx) {

    override fun getName() = "MyModule"

    override fun multiply(a: Double, b: Double, promise: Promise) {
        promise.resolve(a * b)
    }

    override fun getVersion(): String = "2.0.0"
}
```

### Swift TurboModule

```swift
// MyModule.swift
import Foundation
import React

@objc(MyModule)
class MyModule: NSObject, NativeMyModuleSpec {

    func multiply(_ a: Double, b: Double, resolve: @escaping RCTPromiseResolveBlock,
                  reject: @escaping RCTPromiseRejectBlock) {
        resolve(a * b)
    }

    func getVersion() -> String {
        return "2.0.0"
    }

    static func moduleName() -> String! { "MyModule" }
    static func requiresMainQueueSetup() -> Bool { false }
}
```

### JS Usage (same as before — the API is identical)

```ts
import NativeMyModule from './NativeMyModule';

const result = await NativeMyModule.multiply(6, 7); // 42
const version = NativeMyModule.getVersion(); // synchronous
```

**Key differences from old NativeModules:**
- No `NativeModules.MyModule` — import the typed spec directly
- Module isn't loaded until first call (lazy)
- Type errors caught at Codegen time, not at runtime

---

## 7. C++ JSI Native Module (Advanced)

### Why C++ Instead of TurboModules?

Use direct JSI/C++ when you need:
- **Synchronous, zero-overhead calls** (TurboModules still have some JNI overhead on Android)
- **Loading `.so` files** — no Java/ObjC API for `dlopen`
- **Heavy computation** in native code with complex data passing
- **Shared C++ logic** between Android and iOS from a single `.cpp` file

### Key JSI Types

| Type | What it is |
|------|-----------|
| `jsi::Runtime&` | The JS engine — you need this for everything |
| `jsi::Value` | Any JS value (string, number, bool, object, undefined) |
| `jsi::String` | A JS string |
| `jsi::Object` | A JS object |
| `jsi::Function` | A callable JS function |
| `jsi::HostObject` | A C++ class exposed as a JS object |

### Minimal JSI Module

```cpp
// MyJSIModule.cpp
#include <jsi/jsi.h>
#include <android/log.h>

using namespace facebook::jsi;

// This function gets called once when the module installs itself
void installMyModule(Runtime& rt) {

    // Create a JS function from C++ lambda
    auto multiplyFn = Function::createFromHostFunction(
        rt,
        PropNameID::forAscii(rt, "multiply"), // JS name of the function
        2,                                      // expected argument count
        [](Runtime& rt, const Value& thisVal, const Value* args, size_t count) -> Value {
            // Always validate arguments — wrong type = crash
            if (count < 2 || !args[0].isNumber() || !args[1].isNumber()) {
                throw JSError(rt, "multiply requires two numbers");
            }

            double a = args[0].asNumber();
            double b = args[1].asNumber();

            return Value(a * b); // return a jsi::Value
        }
    );

    // Install the function as a global: global.nativeMultiply = multiplyFn
    rt.global().setProperty(rt, "nativeMultiply", std::move(multiplyFn));
}
```

### Host Object — Exposing a C++ Class to JS

```cpp
// MyHostObject.h
#include <jsi/jsi.h>
#include <string>

using namespace facebook::jsi;

class MyHostObject : public HostObject {
public:
    // Called when JS does: obj.someProperty
    Value get(Runtime& rt, const PropNameID& name) override {
        std::string propName = name.utf8(rt);

        if (propName == "version") {
            return String::createFromUtf8(rt, "1.0.0");
        }

        if (propName == "compute") {
            // Return a function as a property
            return Function::createFromHostFunction(
                rt,
                PropNameID::forAscii(rt, "compute"),
                1,
                [this](Runtime& rt, const Value&, const Value* args, size_t count) -> Value {
                    if (count < 1 || !args[0].isNumber()) {
                        throw JSError(rt, "compute needs a number");
                    }
                    return Value(args[0].asNumber() * 2.0);
                }
            );
        }

        return Value::undefined();
    }

    // Called when JS does: obj.someProperty = value
    void set(Runtime& rt, const PropNameID& name, const Value& value) override {
        // Handle property writes if needed
    }
};
```

```cpp
// Installing the host object
void installHostObject(Runtime& rt) {
    auto obj = std::make_shared<MyHostObject>();
    // Object::createFromHostObject wraps our C++ class as a JS object
    auto jsObj = Object::createFromHostObject(rt, obj);
    rt.global().setProperty(rt, "myNativeObj", std::move(jsObj));
}
```

**JS usage:**
```js
console.log(myNativeObj.version);    // "1.0.0"
console.log(myNativeObj.compute(21)); // 42
```

### `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.13)
project(MyJSIModule)

# Find the JSI and ReactNative headers from the RN node_modules
set(RN_DIR ${CMAKE_SOURCE_DIR}/../../../node_modules/react-native)

add_library(
    MyJSIModule   # output .so name
    SHARED        # shared library
    MyJSIModule.cpp
    MyHostObject.cpp
)

target_include_directories(MyJSIModule PRIVATE
    ${RN_DIR}/ReactCommon/jsi
    ${RN_DIR}/ReactCommon/react/nativemodule/core
    ${RN_DIR}/ReactAndroid/src/main/jni/react/turbomodule
)

target_link_libraries(MyJSIModule
    android        # Android system lib
    log            # for __android_log_print
    jsi            # JSI interface (provided by RN's prefab)
)
```

### `build.gradle` Changes

```gradle
android {
    defaultConfig {
        ndk {
            // Only build for these ABIs — must match what your loaded .so supports
            abiFilters "arm64-v8a", "x86_64"
        }
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++17"
                arguments "-DANDROID_STL=c++_shared"
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
```

### Installing from JNI (`OnLoad.cpp`)

```cpp
// OnLoad.cpp — entry point when .so is loaded by the JVM
#include <jni.h>
#include <jsi/jsi.h>
#include "MyJSIModule.h"

extern "C" {

// Called by Java when the library loads: System.loadLibrary("MyJSIModule")
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void*) {
    return JNI_VERSION_1_6;
}

// Called explicitly from a ReactPackage or TurboModule to install into runtime
JNIEXPORT void JNICALL
Java_com_myapp_MyJSIModule_nativeInstall(JNIEnv* env, jobject thiz, jlong runtimePtr) {
    // runtimePtr is the address of the jsi::Runtime — cast it back
    auto& runtime = *reinterpret_cast<facebook::jsi::Runtime*>(runtimePtr);
    installMyModule(runtime);
    installHostObject(runtime);
}

} // extern "C"
```

```kotlin
// MyJSIModule.kt — Kotlin side that calls into C++
package com.myapp

import com.facebook.react.bridge.*
import com.facebook.react.turbomodule.core.CallInvokerHolderImpl

class MyJSIModule(ctx: ReactApplicationContext) : ReactContextBaseJavaModule(ctx) {

    override fun getName() = "MyJSIModule"

    @ReactMethod(isBlockingSynchronousMethod = true)
    fun install(): Boolean {
        try {
            System.loadLibrary("MyJSIModule") // loads libMyJSIModule.so
            val instance = ctx.javaScriptContextHolder?.get() ?: return false
            nativeInstall(instance) // pass runtime pointer
            return true
        } catch (e: Exception) {
            return false
        }
    }

    private external fun nativeInstall(jsiRuntimeRef: Long)
}
```

**Common mistakes:**
- Not checking `isString()` / `isNumber()` before calling `asString()` / `asNumber()` — immediate crash
- Storing `jsi::Value` across calls without copying — values are tied to the current call scope
- Forgetting `-std=c++17` in cmake flags — JSI uses modern C++ features
- ABI mismatch: your module `.so` compiled for `arm64-v8a` won't load on an `x86` emulator
