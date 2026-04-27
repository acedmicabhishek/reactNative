# JNI & C++ Bindings in React Native (Android)

## Overview
JNI (Java Native Interface) is the mechanism that allows Java/Kotlin code to call C++ code and vice versa on Android. React Native's entire C++ core (JSI, Yoga, Fabric, Hermes) runs in C++, and JNI is the bridge between that C++ world and the Android (Java/Kotlin) application layer.

---

## 1. Why JNI Exists

Android apps are primarily written in Java/Kotlin, which runs on the **ART (Android Runtime)** — a managed VM. The OS itself and most high-performance libraries are in C/C++ (native code).

JNI allows:
- Calling C++ functions from Java (downward call)
- Calling Java methods from C++ (upward call)
- Passing data between the two runtimes

React Native uses JNI extensively:
- Hermes (C++) ↔ Android runtime
- Yoga (C++) ↔ Android View system
- Fabric C++ renderer ↔ Android ViewManagers (Java)
- TurboModules C++ layer ↔ Java TurboModule implementations

---

## 2. How JNI Works

### Java → C++ (Downward)

```kotlin
// Kotlin: declare a native method
class ReactNativeHost {
  external fun nativeInit(instanceManager: Long): Boolean
  
  companion object {
    init {
      System.loadLibrary("reactnative")  // loads libreactnative.so
    }
  }
}
```

```cpp
// C++: implement the native method
// Naming convention: Java_packagename_classname_methodname
extern "C" JNIEXPORT jboolean JNICALL
Java_com_facebook_react_ReactNativeHost_nativeInit(
  JNIEnv* env,       // JNI environment — used for all JNI calls
  jobject thiz,      // 'this' Java object reference
  jlong instanceManager
) {
  // Cast instanceManager back to C++ pointer
  auto* manager = reinterpret_cast<ReactInstanceManager*>(instanceManager);
  return manager->init();
}
```

### C++ → Java (Upward)

```cpp
// C++: call a Java method
void callJavaMethod(JNIEnv* env, jobject javaObj) {
  // Get the Java class
  jclass cls = env->GetObjectClass(javaObj);
  
  // Get the method ID (expensive — cache this!)
  jmethodID mid = env->GetMethodID(cls, "onComplete", "(Ljava/lang/String;)V");
  
  // Call the method
  jstring result = env->NewStringUTF("success");
  env->CallVoidMethod(javaObj, mid, result);
  
  // Clean up local references (important!)
  env->DeleteLocalRef(result);
  env->DeleteLocalRef(cls);
}
```

---

## 3. JNI Data Type Mapping

| Java/Kotlin Type | JNI Type | C++ Type |
|-----------------|----------|----------|
| `boolean` | `jboolean` | `uint8_t` |
| `int` | `jint` | `int32_t` |
| `long` | `jlong` | `int64_t` |
| `double` | `jdouble` | `double` |
| `String` | `jstring` | (must convert via `GetStringUTFChars`) |
| `byte[]` | `jbyteArray` | (must lock/unlock via `GetByteArrayElements`) |
| Any object | `jobject` | (opaque reference) |

### String Conversion (Common Source of Bugs)

```cpp
// Java String → C++ std::string
std::string jstringToString(JNIEnv* env, jstring jstr) {
  const char* chars = env->GetStringUTFChars(jstr, nullptr);
  std::string result(chars);
  env->ReleaseStringUTFChars(jstr, chars);  // MUST release!
  return result;
}

// C++ std::string → Java String
jstring stringToJstring(JNIEnv* env, const std::string& str) {
  return env->NewStringUTF(str.c_str());
  // Caller must DeleteLocalRef when done
}
```

### Byte Array Conversion (For Binary Data)

```cpp
// Java byte[] → C++ data (zero-copy if you use GetByteArrayElements)
void processBytes(JNIEnv* env, jbyteArray jArray) {
  jsize length = env->GetArrayLength(jArray);
  jbyte* bytes = env->GetByteArrayElements(jArray, nullptr);  // pins the array
  
  // Process bytes[0..length-1] directly (no copy!)
  processBuffer(reinterpret_cast<uint8_t*>(bytes), length);
  
  env->ReleaseByteArrayElements(jArray, bytes, JNI_ABORT);  // unpin (no copy-back)
}
```

---

## 4. FBJNI — Facebook's JNI Wrapper

Facebook built **fbjni** (Facebook JNI) to make JNI in React Native safer and easier. It wraps raw JNI in C++ RAII types, preventing leaks.

```cpp
#include <fbjni/fbjni.h>
using namespace facebook::jni;

// C++ class wrapping a Java class (JavaClass pattern)
struct JReactCallback : JavaClass<JReactCallback> {
  static constexpr auto kJavaDescriptor = "Lcom/facebook/react/modules/core/ReactCallback;";
  
  void invoke(const std::string& event) {
    static const auto method = javaClassStatic()->getMethod<void(jstring)>("invoke");
    method(self(), make_jstring(event).get());
  }
};

// Safe string handling (no manual Release needed)
auto jStr = make_jstring("hello from C++");
// jStr auto-deletes local reference when it goes out of scope

// Safe exception handling
try {
  auto result = someJavaMethod->call(obj, args...);
} catch (const facebook::jni::JniException& e) {
  // Java exception translated to C++ exception
  ALOGW("JNI error: %s", e.what());
}
```

---

## 5. React Native's C++ ↔ Android Layer

### How Fabric Components Talk to Android

For a custom Fabric View Component:

```
React Component (JS)
  → Fabric C++ ShadowNode (describes props)
  → ViewManager (Java) implements createViewInstance()
    → Returns an actual Android View
  → ViewManager applies props via setters
    → view.setBackgroundColor(color)
    → view.setCustomProp(value)
```

```java
// Java: Custom ViewManager
public class MyCustomViewManager extends SimpleViewManager<MyCustomView> {
  
  @Override
  public String getName() { return "MyCustomView"; }

  @Override
  public MyCustomView createViewInstance(ThemedReactContext context) {
    return new MyCustomView(context);
  }

  // Called by Fabric when C++ side says prop changed
  @ReactProp(name = "color")
  public void setColor(MyCustomView view, String color) {
    view.setColor(Color.parseColor(color));
  }
}
```

```cpp
// C++: The Fabric side descriptor
class MyCustomViewComponentDescriptor 
    : public ConcreteComponentDescriptor<MyCustomViewShadowNode> {
  // Fabric calls createShadowNode when component is first rendered
  // It will eventually call Java's createViewInstance via JNI
};
```

---

## 6. JNI Performance Considerations

### Cache Method IDs (Critical)
`GetMethodID` and `GetFieldID` involve string lookups — they're expensive (~microseconds). Cache them in static variables:

```cpp
// BAD: Looks up method on every call
void onFrame(JNIEnv* env, jobject callback) {
  jmethodID mid = env->GetMethodID(env->GetObjectClass(callback), "onFrame", "()V");
  env->CallVoidMethod(callback, mid);
}

// GOOD: Cache in static (thread-safe after first call)
void onFrame(JNIEnv* env, jobject callback) {
  static jmethodID mid = nullptr;
  if (!mid) {
    mid = env->GetMethodID(env->GetObjectClass(callback), "onFrame", "()V");
  }
  env->CallVoidMethod(callback, mid);
}
```

### Local vs Global References
- **Local refs:** Valid only in the current JNI call stack frame. Auto-deleted when the native method returns. The JNI frame has a limit of 512 local refs.
- **Global refs:** Survive across JNI calls. Must be manually deleted with `DeleteGlobalRef`.
- **Weak global refs:** Like global refs but don't prevent GC.

```cpp
// Promoting a local ref to global (e.g., to store in a C++ struct)
jobject localRef = env->NewObject(cls, constructor);
jobject globalRef = env->NewGlobalRef(localRef);  // survive beyond this call
env->DeleteLocalRef(localRef);                     // clean up local immediately

// Later, in destructor:
env->DeleteGlobalRef(globalRef);
```

### Minimize JNI Crossings
Every Java ↔ C++ crossing has overhead (thread state checks, argument marshaling). Batch operations instead of one call per item:

```cpp
// BAD: 1000 JNI crossings
for (auto& item : items) {
  env->CallVoidMethod(callback, mid, item.value);
}

// GOOD: 1 JNI crossing with a batch
jintArray batch = env->NewIntArray(items.size());
jint* elems = env->GetIntArrayElements(batch, nullptr);
for (size_t i = 0; i < items.size(); ++i) elems[i] = items[i].value;
env->ReleaseIntArrayElements(batch, elems, 0);
env->CallVoidMethod(callback, batchMid, batch);
env->DeleteLocalRef(batch);
```

---

## 7. Writing a Custom JNI Native Module (Android)

Here's a minimal complete example of a custom C++ module accessible from React Native:

```kotlin
// android/src/main/java/.../NativeHasherModule.kt
class NativeHasherModule(reactContext: ReactApplicationContext) 
    : ReactContextBaseJavaModule(reactContext) {

  override fun getName() = "NativeHasher"

  @ReactMethod
  fun sha256(input: String, promise: Promise) {
    // Delegate to C++ for performance
    val result = nativeSha256(input)
    promise.resolve(result)
  }

  private external fun nativeSha256(input: String): String

  companion object {
    init { System.loadLibrary("native_hasher") }
  }
}
```

```cpp
// cpp/native_hasher.cpp
#include <jni.h>
#include <openssl/sha.h>
#include <string>
#include <sstream>
#include <iomanip>

extern "C" JNIEXPORT jstring JNICALL
Java_com_myapp_NativeHasherModule_nativeSha256(
    JNIEnv* env, jobject /* thiz */, jstring jinput) {
  
  const char* input = env->GetStringUTFChars(jinput, nullptr);
  
  unsigned char hash[SHA256_DIGEST_LENGTH];
  SHA256(reinterpret_cast<const unsigned char*>(input), strlen(input), hash);
  
  env->ReleaseStringUTFChars(jinput, input);
  
  std::ostringstream ss;
  for (int i = 0; i < SHA256_DIGEST_LENGTH; ++i)
    ss << std::hex << std::setw(2) << std::setfill('0') << (int)hash[i];
  
  return env->NewStringUTF(ss.str().c_str());
}
```

```cmake
# android/CMakeLists.txt
cmake_minimum_required(VERSION 3.13)
add_library(native_hasher SHARED cpp/native_hasher.cpp)
target_link_libraries(native_hasher log OpenSSL::Crypto)
```
