# Memory Management in React Native

## Overview
Memory management in React Native spans two worlds: the **JS heap** (managed by the JS engine's garbage collector) and **native memory** (managed by the OS and native components). Leaks or excessive allocation in either world causes the app to slow down, crash, or be killed by the OS.

---

## 1. The Two Memory Worlds

```
┌─────────────────────────────────────────┐
│  JS Heap (Hermes/JSC GC managed)        │
│  - React component trees                │
│  - JS objects, arrays, strings          │
│  - Closures, callbacks                  │
│  - Reanimated shared values (JS side)   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Native Memory (OS managed)             │
│  - Native Views (UIView/android.view)   │
│  - Image bitmaps (decoded pixels)       │
│  - Video frames                         │
│  - Audio buffers                        │
│  - Custom C++ objects (JSI HostObjects) │
│  - JNI object references               │
└─────────────────────────────────────────┘
```

---

## 2. JS Heap Memory

### How JS Memory is Allocated

```javascript
// Each of these allocates on the JS heap:
const obj = { name: 'John', age: 30 };           // ~100 bytes
const arr = new Array(10000).fill(0);              // ~80KB
const str = 'Hello'.repeat(100000);                // ~500KB
const closure = () => capturedVariable;            // captures outer scope
const imageData = new Uint8Array(1920 * 1080 * 4); // 7.9MB — but in native memory!
```

**Typed Arrays** (`Uint8Array`, `Float32Array`, etc.) are backed by `ArrayBuffer`, which in Hermes is allocated in **native/C++ memory**, not the JS heap. This is important: large data buffers won't pressure the JS GC.

### Measuring JS Heap Usage

```bash
# Android: Connect device, open Chrome DevTools
# Navigate to: chrome://inspect
# Click "inspect" next to your app's Hermes VM

# In DevTools > Memory tab:
# - Take heap snapshot
# - Compare snapshots to find leaks
# - Use Allocation Timeline to see allocations over time

# Or via Flipper:
# Install Flipper → Open app → Memory tab → JS Heap section
```

### Common JS Memory Leaks

#### 1. Closures Capturing Large Objects
```js
// BAD: This closure prevents the entire `data` array from being GC'd
// even after the component unmounts
const handler = someEventEmitter.addListener('event', () => {
  processData(data); // `data` is captured in closure
});
// If you forget to remove the listener, `data` lives forever
```

#### 2. Unremoved Event Listeners / Subscriptions
```jsx
useEffect(() => {
  const subscription = DeviceEventEmitter.addListener('appStateChange', handler);
  return () => subscription.remove();  // ← critical! Don't forget this
}, []);
```

#### 3. Intervals and Timers
```jsx
useEffect(() => {
  const id = setInterval(() => updateState(), 1000);
  return () => clearInterval(id);  // ← clean up or the interval fires after unmount
}, []);
```

#### 4. Large State Arrays
```jsx
// Accumulating data in state without cleanup
const [logs, setLogs] = useState([]);
// If you push to logs on every render without truncating, it grows unbounded
setLogs(prev => [...prev, newLog].slice(-100)); // keep only last 100
```

---

## 3. Native Memory

### Image Memory — The Silent Killer

A single high-resolution image decoded to a bitmap can consume enormous memory:
```
1920×1080 RGBA image = 1920 × 1080 × 4 bytes = 7.9MB per image
3840×2160 4K image   = 3840 × 2160 × 4 bytes = 31.6MB per image

For a list of 100 images: 790MB of decoded bitmaps!
```

**Solutions:**
- **Image resizing:** Use `Image` component with explicit `width` and `height` — the platform decodes to the display size, not the file size.
- **Caching with eviction:** Use `react-native-fast-image` (uses SDWebImage / Glide) which implements proper LRU cache with memory limits.
- **Lazy decoding:** Only decode images when they're about to enter the viewport.

```jsx
// react-native-fast-image — proper memory management
import FastImage from 'react-native-fast-image';

<FastImage
  source={{ uri: imageUrl, priority: FastImage.priority.normal }}
  style={{ width: 150, height: 150 }}
  resizeMode={FastImage.resizeMode.cover}
/>
```

### Native View Memory

Every `<View>`, `<Text>`, `<Image>` creates a native object in memory. For lists:
- A FlatList with 1000 items that are all rendered = 1000 native Views in memory.
- Virtualization (`FlatList` with `windowSize`) keeps only ~10-30 native Views at a time.
- `removeClippedSubviews={true}` — detaches (but doesn't destroy) off-screen views, freeing their backing GPU memory.

---

## 4. Android Memory Management

### Memory Limits
Android allocates each app a **heap size limit** defined by the device:
```bash
adb shell getprop dalvik.vm.heapsize    # e.g., 512m
adb shell getprop dalvik.vm.heapgrowthlimit # e.g., 256m (initial limit)
```

Exceeding the limit → `OutOfMemoryError` → app crash.

### `largeHeap` Mode
```xml
<!-- AndroidManifest.xml -->
<application android:largeHeap="true" ...>
```
Increases the heap limit (2x on most devices). Use as a last resort — fix leaks instead.

### GC on Android
Android uses ART's garbage collector. The Java/Kotlin native layer has its own GC separate from Hermes. Heavy native allocations (Bitmaps, Buffers) go through ART.

```bash
# Monitor Android memory (native + Java heap + GPU)
adb shell dumpsys meminfo com.yourapp.package

# Output includes:
# Dalvik Heap: JS/Java heap
# Native Heap: C++ allocations
# Graphics: GPU memory for textures, buffers
# Code: Loaded libraries and code segments
```

---

## 5. Detecting Memory Leaks

### Android: LeakCanary
```groovy
// build.gradle (debug only)
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.x'
// No code changes needed — LeakCanary auto-hooks into Activity/Fragment lifecycle
```

### The `useEffect` Cleanup Pattern
The most common source of React Native memory leaks: not cleaning up effects.

```jsx
const MyComponent = ({ userId }) => {
  useEffect(() => {
    // Everything allocated in here must be cleaned up in the return
    const ws = new WebSocket(`wss://api.example.com/user/${userId}`);
    const subscription = store.subscribe(handleStoreChange);
    const timer = setInterval(pollData, 5000);

    return () => {  // cleanup function
      ws.close();
      subscription();  // unsubscribe
      clearInterval(timer);
    };
  }, [userId]);
};
```

### Profiling with Android Studio Memory Profiler

```
Android Studio → Run → Profile → Memory
→ Record allocations for 30 seconds of usage
→ Look for:
  - Growing heap that never decreases (leak)
  - Large Bitmap allocations
  - Many instances of the same object (leaked listeners)
→ "Force GC" button → if heap doesn't drop, something is preventing GC
```

---

## 6. Memory Optimization Strategies

### 1. Reduce Image Cache Size
```jsx
// FastImage cache configuration
import FastImage from 'react-native-fast-image';
FastImage.clearMemoryCache();  // call when app goes to background
FastImage.clearDiskCache();    // call on logout / cache invalidation
```

### 2. Limit List Window Size
```jsx
// FlatList: only keep 5x screen height of content in memory
<FlatList windowSize={5} removeClippedSubviews={true} ... />
```

### 3. Release Resources on App Background
```jsx
import { AppState } from 'react-native';

useEffect(() => {
  const subscription = AppState.addEventListener('change', (state) => {
    if (state === 'background') {
      // Release large caches, pause video players, disconnect websockets
      videoPlayer.current?.pause();
      imageCache.trim();
    }
  });
  return () => subscription.remove();
}, []);
```

### 4. Use Native Storage for Large Data
Don't store large buffers in JS state. Use:
- **MMKV** — C++ key-value store, very fast, data lives in native memory.
- **SQLite (op-sqlite)** — SQL database with C++ core.
- **File system** — for binary data (images, audio).

### 5. Avoid Inline JSX Object Creation

```jsx
// BAD: creates a new object on every render
<View style={{ flex: 1, backgroundColor: theme.colors.background }}>

// GOOD: stable reference
const styles = StyleSheet.create({ container: { flex: 1 } });
<View style={[styles.container, { backgroundColor: theme.colors.background }]}>
```

---

## 7. Native Memory for Custom C++ Modules

When building JSI-based modules, you own your C++ memory:

```cpp
class LargeDataModule : public jsi::HostObject {
  std::vector<uint8_t> buffer_;  // C++ heap
  
public:
  explicit LargeDataModule(size_t size) : buffer_(size) {}
  
  // When all JS references to this HostObject are GC'd,
  // the shared_ptr goes to zero → destructor runs → buffer_ is freed
  ~LargeDataModule() {
    // buffer_ is automatically freed here (RAII)
  }
};

// Install — the JS GC keeps this alive as long as JS holds a reference
auto module = std::make_shared<LargeDataModule>(10 * 1024 * 1024); // 10MB
rt.global().setProperty(rt, "largeData", 
  jsi::Object::createFromHostObject(rt, module));
```

The lifetime of your C++ objects is tied to the JS GC via `shared_ptr`. When JS GC collects the HostObject reference, the `shared_ptr` reference count drops to zero, and your destructor frees native memory. This is RAII — no manual `free()` needed.

**Warning:** If you store a `jsi::Value` or `jsi::Object` inside a C++ HostObject, you're creating a reference cycle — JS GC won't collect the HostObject because it holds a JS reference, and the JS reference won't be released because it's in the HostObject. Use `jsi::WeakObject` for back-references to JS.
