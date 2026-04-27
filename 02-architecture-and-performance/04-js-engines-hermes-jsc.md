# JS Engines in React Native: Hermes vs JSC

## Overview
A JavaScript engine is the runtime that executes your JS code. React Native has supported multiple engines: **JavaScriptCore (JSC)**, **Hermes**, and experimentally **V8**. Choosing and understanding your engine is critical for startup time, runtime performance, memory usage, and GC pauses.

---

## 1. JavaScriptCore (JSC) — The Original Engine

JSC is the JS engine that powers **Safari** and **WebKit**. It was React Native's original engine from 2015–2021.

### Architecture
- **Interpreter:** LLInt (Low-Level Interpreter) — executes bytecode directly.
- **Baseline JIT:** DFG (Data Flow Graph) — first tier of JIT compilation.
- **Optimizing JIT:** FTL (Fourth Tier LLVM) — full optimization using LLVM backend.

### Why it was replaced as the default
1. **Large binary size:** JSC ships as a full `libjsc.so` (~3-5MB) on Android. iOS uses the system-provided JSC (no extra size).
2. **Slow startup:** JSC needs to parse and compile JS at startup — no ahead-of-time (AOT) compilation.
3. **Memory:** JSC's JIT tiers consume significant memory.
4. **Not optimized for mobile:** Designed for desktop/server workloads (Safari).

### When to still consider JSC
- Apps requiring `eval()` or `new Function()` at runtime (Hermes disables these for security).
- When using `react-native-v8` for V8 instead.
- Legacy codebases not yet migrated to Hermes.

---

## 2. Hermes — The React Native Engine

Hermes is a **JavaScript engine built by Meta specifically for React Native on mobile**. Open-sourced in 2019, it became the default in Expo SDK 42 / RN 0.64.

### Key Design Decisions

#### a) Ahead-of-Time (AOT) Bytecode Compilation

The biggest performance win. Metro Bundler compiles your JS source code into **Hermes bytecode** (`.hbc`) at **build time**, not at app startup.

```
Build time (Metro):
  Your JS code → parse → AST → Hermes bytecode (.hbc) → packed into app bundle

Runtime (app launch):
  App starts → loads pre-compiled bytecode → executes immediately
  (NO parsing, NO JIT warm-up delay)
```

**Result:** App startup time dramatically reduced (often 30–50% faster TTI).

#### b) Optimized for Low Memory
- No JIT compiler at runtime (in standard mode) → predictable, low memory footprint.
- Hermes uses a **compact GC** designed for the 2–4GB RAM constraints of mobile devices.
- Smaller heap sizes compared to JSC.

#### c) Lazy Evaluation
Hermes can be configured to load and execute only the modules needed for initial render, deferring the rest. Combined with Metro's bundle splitting, this enables fast startup.

---

## 3. Hermes vs JSC: Direct Comparison

| Metric | JSC | Hermes |
|--------|-----|--------|
| Startup time (TTI) | Slower (JIT warm-up) | **Faster** (pre-compiled bytecode) |
| Runtime speed (steady-state) | Faster (JIT-optimized) | Slightly slower (no runtime JIT) |
| Memory usage | Higher | **Lower** |
| Binary size (Android) | +3-5MB | +1-2MB (or system Hermes) |
| GC pauses | JIT adds unpredictability | More predictable |
| `eval()` support | ✅ | ❌ (security decision) |
| Source maps | ✅ | ✅ (`.map` files) |
| Debugging | Chrome DevTools | Hermes Inspector (Chrome DevTools protocol) |

---

## 4. Garbage Collection in Hermes

Understanding GC is critical for diagnosing **jank spikes** that aren't caused by CPU overload.

### Hermes GC: GenGC / HadesGC

Hermes has two GC implementations:
- **GenGC:** Generational, stop-the-world GC. Simple but causes JS thread pauses.
- **HadesGC (default in newer Hermes):** Concurrent GC — most collection work happens on background threads concurrently with JS execution. Stop-the-world pauses are very short.

```
GenGC (older):
  ┌──────────────────────────────────────┐
  │ JS Thread ████████ GC PAUSE ████████ │   ← jank!
  └──────────────────────────────────────┘

HadesGC (modern):
  ┌──────────────────────────────────────┐
  │ JS Thread ████████ ░░░░░ ████████   │   ← short pause
  │ GC Thread         ██████████████    │   ← runs concurrently
  └──────────────────────────────────────┘
```

### Diagnosing GC-Induced Jank

```bash
# Enable GC logging in Hermes
# Add to your JS entry point:
if (global.HermesInternal) {
  global.HermesInternal.enableSamplingProfiler?.();
}

# After triggering GC jank, collect profile:
global.HermesInternal.dumpSampledTraceToFile?.('/sdcard/trace.json');
# Open in chrome://tracing
```

Or use **Android Profiler** in Android Studio to see GC events correlated with frame drops.

### Reducing GC Pressure

GC pauses happen when too many JS objects are allocated and need collection. To reduce pressure:
- Avoid creating large temporary arrays in tight loops.
- Reuse objects instead of creating new ones.
- Use typed arrays (`Uint8Array`, `Float32Array`) for numeric data — they're backed by native memory and don't pressure the JS heap.
- Use `MMKV` (C++ storage) instead of `AsyncStorage` (JS objects) for large data.

---

## 5. Hermes Bytecode Deep Dive

```bash
# Compile JS to Hermes bytecode manually
npx hermes -emit-binary -out output.hbc input.js

# Inspect bytecode
npx hermes -dump-bytecode output.hbc

# Check if your production bundle uses Hermes bytecode
# (Metro does this automatically in production builds)
file android/app/src/main/assets/index.android.bundle
# → "Hermes JavaScript bytecode, version ..."
```

**Bytecode Format:**
- Binary format (not human-readable JS)
- Contains the compiled IR (Intermediate Representation) of all functions
- Functions are lazy — their body is not parsed until they're first called (even in bytecode form)

---

## 6. V8 for React Native

V8 is Google's JS engine (powers Chrome and Node.js). Available as an experimental replacement via [react-native-v8](https://github.com/Kudo/react-native-v8).

### V8 Advantages
- Full JIT pipeline (Ignition → Sparkplug → Maglev → Turbofan) — potentially faster steady-state performance.
- Better support for modern JS features.
- Familiar tooling (Chrome DevTools native support).

### V8 Disadvantages
- Large binary (+6-8MB on Android).
- Not officially supported by Meta.
- No AOT bytecode (starts parsing JS at runtime).
- Higher memory usage.

**Verdict:** V8 is better for compute-intensive apps (games, ML). Hermes is better for typical mobile apps (fast startup matters more than peak throughput).

---

## 7. Engine Internals: How JS Executes

```
Source Code (.js)
   ↓ Parsing
Abstract Syntax Tree (AST)
   ↓ Compilation
Bytecode / IR
   ↓ Interpreter (executes immediately)
Profiling data (which functions are hot?)
   ↓ JIT Compilation (for hot functions — JSC/V8 only, not Hermes by default)
Native machine code (x86_64 / ARM64)
   ↓ Execution
Result
```

Hermes skips the last two steps at runtime (JIT is done at build time instead).

---

## 8. Enabling Hermes in Your App

**Expo (default since SDK 48):**
```json
// app.json
{
  "expo": {
    "jsEngine": "hermes"  // default, can be "jsc"
  }
}
```

**React Native CLI:**
```java
// android/app/build.gradle
project.ext.react = [
    enableHermes: true,
]
```

```ruby
# ios/Podfile
use_react_native!(
  :hermes_enabled => true
)
```

---

## 9. Building a Custom JS Engine Integration

If you want to integrate a completely different JS engine (e.g., QuickJS for size, or a custom WASM-based engine):

1. Implement the **`jsi::Runtime`** interface in C++.
2. React Native's JSI layer sits on top of `jsi::Runtime` — your engine just needs to implement this API.
3. Register your runtime factory via `ReactInstanceManagerBuilder.setJavaScriptExecutorFactory()` (Android).

This is the level at which you can swap the entire JS execution environment of a React Native app.
