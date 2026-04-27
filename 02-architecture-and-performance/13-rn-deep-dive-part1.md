# React Native Deep Dive — Part 1: Metro, Babel & Architecture

---

## 1. Metro Bundler

### What is Metro?

Metro is the JavaScript bundler built specifically for React Native by Meta. It replaces Webpack for RN because Webpack was designed for the web — it doesn't understand native module resolution, doesn't handle fast refresh for device targets, and doesn't optimize for mobile bundle formats.

Metro is optimized for:
- **Speed** — incremental builds, aggressive caching
- **Mobile** — understands platform extensions (`.android.js`, `.ios.js`)
- **Hot reload** — only sends changed modules to the device, not the whole bundle

### The 3 Stages

```
Entry Point (index.js)
        │
        ▼
┌───────────────────┐
│   1. RESOLUTION   │  ← Find every file the entry point imports (recursively)
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ 2. TRANSFORMATION │  ← Run each file through Babel (JSX → JS, TS → JS, etc.)
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ 3. SERIALIZATION  │  ← Stitch all modules into one bundle file
└───────────────────┘
        │
        ▼
  bundle.js (served to device or written to disk)
```

**Resolution:** Metro starts at your entry file and follows every `import`/`require` recursively. It builds a dependency graph — a map of every module and what it depends on. It respects `package.json` `main` field, `node_modules`, and platform-specific extensions.

**Transformation:** Each file in the graph gets transformed. Metro calls Babel (or a custom transformer) on each file to convert JSX, TypeScript, and modern JS down to something the JS engine on the device understands. This happens in parallel workers.

**Serialization:** The transformed modules get combined into a single bundle. Each module is wrapped in a `__d()` (define) call with its module ID. The runtime uses this to resolve dependencies at runtime.

### Dependency Graph

```
index.js
  ├── App.js
  │     ├── HomeScreen.js
  │     │     └── Button.js
  │     └── utils/format.js
  └── react-native  (node_modules)
        └── ...
```

Metro keeps this graph in memory during dev mode. When a file changes, it only re-transforms that file and updates the graph — not the whole bundle.

### Hot Module Replacement (HMR)

In dev mode, Metro runs a local HTTP server (default port 8081). The device connects via WebSocket. When you save a file:

1. Metro detects the change via file watcher (watchman)
2. Re-transforms only the changed module
3. Sends a delta update over WebSocket to the device
4. The JS runtime swaps the old module for the new one in-place

The component state is preserved because only the module definition is replaced, not the entire JS context.

### Caching

Metro caches the transformation result of each file keyed by:
- File content hash
- Transformer config hash
- Babel config hash

Stored in a local cache (usually `node_modules/.cache/metro`). On rebuild, if nothing changed, Metro skips transformation entirely for that file. This is why second builds are much faster.

### Production Mode

```bash
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output android/app/src/main/assets/index.android.bundle \
  --assets-dest android/app/src/main/res
```

This runs all 3 stages and writes a static bundle file. No server, no HMR. Minification is applied. The bundle is shipped inside the APK.

### `metro.config.js`

```js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const config = {
  resolver: {
    // Add extra file extensions Metro should understand
    sourceExts: ['js', 'jsx', 'ts', 'tsx', 'json', 'svg'],
    // Map module names to specific files
    extraNodeModules: {
      'crypto': require.resolve('react-native-crypto'),
    },
    // Custom resolution for specific platforms
    platforms: ['android', 'ios', 'native', 'web'],
  },

  transformer: {
    // Use a custom transformer instead of default Babel one
    babelTransformerPath: require.resolve('./customTransformer.js'),
    // Inline requires — defer module execution until first use (perf optimization)
    inlineRequires: true,
  },

  serializer: {
    // Add a custom module to every bundle (polyfills etc.)
    getPolyfills: () => [require.resolve('./polyfills/myPolyfill.js')],
  },

  watchFolders: [
    // Tell Metro to watch files outside the project root
    '/home/user/shared-libs',
  ],
};

module.exports = mergeConfig(getDefaultConfig(__dirname), config);
```

**Custom resolver example:**

```js
// metro.config.js
resolver: {
  resolveRequest: (context, moduleName, platform) => {
    // Redirect all imports of 'lodash' to 'lodash-es'
    if (moduleName === 'lodash') {
      return {
        filePath: require.resolve('lodash-es'),
        type: 'sourceFile',
      };
    }
    // Fall through to default resolution
    return context.resolveRequest(context, moduleName, platform);
  },
},
```

**Custom transformer example:**

```js
// customTransformer.js
const { transform } = require('@react-native/metro-babel-transformer');

module.exports.transform = async ({ filename, src, options }) => {
  // Pre-process the source before Babel touches it
  const modifiedSrc = src.replace(/__DEV_ONLY__\(.*?\)/gs, '');
  return transform({ filename, src: modifiedSrc, options });
};
```

**Common mistakes:**
- Forgetting `mergeConfig` and overwriting default config entirely — breaks SVG support, flow, etc.
- Not restarting Metro after changing `metro.config.js` — config is loaded once at startup
- Using `require.resolve` for paths in config vs raw strings — always use `require.resolve` so paths are absolute

---

## 2. Babel in React Native

### What Babel Does

Babel is a JavaScript compiler. It takes JS source code, converts it to an AST, applies transformations, then outputs JS source code again. It does NOT run at runtime — it runs at build/bundle time (called by Metro's transformation stage).

The 3 stages:

```
Source Code (string)
      │
      ▼
┌──────────────┐
│   1. PARSE   │  ← @babel/parser converts source string → AST
└──────────────┘
      │
      ▼
┌──────────────┐
│ 2. TRANSFORM │  ← Plugins traverse AST nodes and mutate them
└──────────────┘
      │
      ▼
┌──────────────┐
│ 3. GENERATE  │  ← @babel/generator converts mutated AST → source string
└──────────────┘
      │
      ▼
Output Code (string) → Metro puts this in the bundle
```

### What an AST Is

Given `const x = 1 + 2`:

```
VariableDeclaration (kind: "const")
  └── VariableDeclarator
        ├── id: Identifier (name: "x")
        └── init: BinaryExpression (operator: "+")
              ├── left: NumericLiteral (value: 1)
              └── right: NumericLiteral (value: 2)
```

Every piece of code is a node with a `type` and child properties. Plugins walk this tree.

### Plugins vs Presets

- **Plugin** — handles one specific transformation (e.g., convert arrow functions to regular functions)
- **Preset** — a curated collection of plugins for a common use case

Plugins run before presets. Within presets, they run last-to-first (reversed order).

### Key Presets in React Native

| Preset | What it does |
|--------|-------------|
| `babel-preset-react-native` | The main RN preset — includes JSX transform, Flow stripping, module system |
| `@babel/preset-react` | JSX → `React.createElement()` calls |
| `@babel/preset-typescript` | Strips TypeScript types (no type checking — that's `tsc`) |

### `babel.config.js` for React Native

```js
module.exports = {
  presets: ['module:@react-native/babel-preset'],

  plugins: [
    // Required for React Native Reanimated (must be last!)
    'react-native-reanimated/plugin',

    // Module resolver — aliases for cleaner imports
    [
      'module-resolver',
      {
        root: ['./src'],
        alias: {
          '@components': './src/components',
          '@utils': './src/utils',
          '@hooks': './src/hooks',
        },
      },
    ],
  ],

  env: {
    production: {
      plugins: ['transform-remove-console'], // Strip console.log in prod
    },
  },
};
```

### JSX Before and After Babel

**Before (what you write):**
```jsx
function Button({ label, onPress }) {
  return (
    <TouchableOpacity onPress={onPress} style={styles.btn}>
      <Text>{label}</Text>
    </TouchableOpacity>
  );
}
```

**After (what Metro puts in the bundle):**
```js
function Button({ label, onPress }) {
  return React.createElement(
    TouchableOpacity,
    { onPress: onPress, style: styles.btn },
    React.createElement(Text, null, label)
  );
}
```

JSX is purely syntax sugar. The engine never sees `<` tags — only `React.createElement` calls.

### Custom Babel Plugin

This plugin replaces all calls to `__log(x)` with `console.log('[DEBUG]', x)`:

```js
// babel-plugin-debug-log.js
module.exports = function ({ types: t }) {
  return {
    visitor: {
      // Called for every CallExpression node in the AST
      CallExpression(path) {
        const callee = path.node.callee;

        // Check if it's a call to __log
        if (t.isIdentifier(callee, { name: '__log' })) {
          // Replace with console.log('[DEBUG]', ...originalArgs)
          path.replaceWith(
            t.callExpression(
              t.memberExpression(
                t.identifier('console'),
                t.identifier('log')
              ),
              [
                t.stringLiteral('[DEBUG]'),
                ...path.node.arguments, // spread original args
              ]
            )
          );
        }
      },
    },
  };
};
```

Usage in `babel.config.js`:
```js
plugins: ['./babel-plugin-debug-log']
```

**Common mistakes:**
- Putting `react-native-reanimated/plugin` anywhere but last — it must be the final plugin
- Confusing Babel with TypeScript type checking — Babel just strips types, it doesn't validate them
- Forgetting `module:` prefix on the RN preset — `'@react-native/babel-preset'` vs `'module:@react-native/babel-preset'` are different things

---

## 3. React Native Architecture

### 3a. Old Architecture (Bridge-based)

The old architecture had two completely separate worlds:

```
┌─────────────────────────────────┐
│         JS Thread               │
│  React reconciler               │
│  Your component code            │
│  Business logic                 │
└──────────────┬──────────────────┘
               │
        JSON serialization
        (async queue)
               │
        ┌──────▼──────┐
        │   BRIDGE    │  ← The bottleneck
        └──────┬──────┘
               │
        JSON deserialization
               │
┌──────────────▼──────────────────┐
│         Native Thread           │
│  UIManager (layout/views)       │
│  NativeModules (APIs)           │
│  Platform code (Java/ObjC)      │
└─────────────────────────────────┘
```

**Problems:**

1. **Serialization overhead:** Every call across the bridge serializes to JSON on one side, sends a message, then deserializes on the other. Doing this for 60fps animations is expensive.

2. **Async only:** You can't wait for a synchronous result. JS fires a message and hopes. This makes things like reading a view's current position impossible without a callback round-trip.

3. **Batching:** Messages were batched and sent in chunks. This introduced latency between a JS event and the native response.

4. **UIManager:** JS sent layout instructions as JSON to UIManager on the native side. UIManager then created/updated native views. The shadow tree (layout calculations) lived in Java/ObjC, separate from C++.

### 3b. New Architecture

The new architecture (stable from RN 0.76) eliminates the Bridge entirely.

#### JSI (JavaScript Interface)

JSI is a lightweight C++ layer that replaces the Bridge for the core communication mechanism.

```
JS Engine (Hermes)
      │
      │  (JSI API — direct C++ calls)
      │
C++ Layer (jsi::Runtime)
      │
      ├── Fabric (UI)
      ├── TurboModules (APIs)
      └── Your custom modules
```

Key properties:
- **Synchronous:** JS can call C++ and get a return value immediately — no async queue
- **No JSON:** C++ objects are exposed directly as JS values — no serialization
- **Direct references:** JS can hold a reference to a C++ object and call methods on it

`jsi::Runtime` is the core object. It represents the JS engine. You use it to:
- Evaluate JS code
- Get/set global variables
- Create JS objects, functions, and arrays from C++
- Call JS functions from C++

#### Fabric (New Renderer)

Fabric replaces UIManager. Key changes:

| Old (UIManager) | New (Fabric) |
|----------------|--------------|
| Shadow tree in Java/ObjC | Shadow tree in C++ (shared) |
| Async layout | Synchronous layout via JSI |
| Separate render passes | Integrated with React reconciler via JSI |

Fabric uses the React reconciler's output (a tree of React elements) and maps it to native views synchronously. Because the shadow tree is now in C++, it can be accessed directly from JS via JSI without any bridge crossing.

#### TurboModules

TurboModules replace the old NativeModules system.

| Old NativeModules | TurboModules |
|------------------|--------------|
| All modules initialized on startup | Lazily initialized (first use) |
| No type safety at bridge boundary | Strongly typed via Codegen |
| Async only | Synchronous capable via JSI |
| Manual bridging code | Auto-generated bindings |

#### Codegen

Codegen takes a TypeScript spec file as input:

```ts
// NativeMyModule.ts (the spec)
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  multiply(a: number, b: number): number;
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

And generates:
- C++ header files with the exact function signatures
- Java/ObjC boilerplate that implements the C++ interface
- Type-safe JS glue

Generated files live in `android/app/build/generated/source/codegen/`.

**Full end-to-end flow (New Architecture):**

```
Your Component Code
        │
        ▼
React Reconciler (JS)
        │  produces a tree of changes
        ▼
Fabric Renderer (C++ via JSI)
        │  maps React tree → native view tree
        ▼
Native UI (Android Views / iOS UIKit)

        ┌──────────────────────┐
        │  During this flow,   │
        │  TurboModules handle │
        │  API calls via JSI:  │
        │                      │
        │  JS → C++ → Java/   │
        │  Swift (sync)        │
        └──────────────────────┘
```

**Common mistakes:**
- Enabling new arch without updating all native deps — many libraries still need bridge compatibility mode
- Thinking Codegen replaces your native implementation — it only generates the glue, you still write the native logic
- Expecting TurboModules to be faster for all cases — they're faster mainly for synchronous, frequent calls

---

## 4. Hermes Engine

### What Hermes Is

Hermes is a JavaScript engine built by Meta specifically for React Native. It replaces JavaScriptCore (JSC) which is the engine used by Safari.

The key difference: **Hermes compiles JS to bytecode at build time**, not at runtime.

### JSC vs Hermes

```
With JSC (old):
  App starts → Load bundle.js → Parse JS → Compile to bytecode → Execute
                                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                 This happens on the user's device
                                 (slow, especially on low-end devices)

With Hermes (new):
  Build time → bundle.js → Hermes compiler → bundle.hbc (bytecode)
  App starts → Load bundle.hbc → Execute
                                  ^^^^^^^
                                  Already compiled, skip parse+compile
```

### Benefits

- **Faster startup:** No parsing or JIT compilation on device
- **Lower memory:** Bytecode is more compact than source JS
- **Smaller APK:** The `.hbc` file is often smaller than the JS source + runtime overhead
- **Deterministic:** Same bytecode every run, no JIT surprises

### Integration with JSI

Hermes implements the JSI interface. That's the key — JSI is an abstraction, and any engine that implements it (Hermes, JSC, V8) can be used. Hermes is the default in modern RN because it implements JSI and is the most optimized for mobile.

### Enabling/Verifying Hermes

In `android/app/build.gradle`:
```gradle
android {
    defaultConfig {
        // Hermes is ON by default in RN 0.70+
    }
}

dependencies {
    // This pulls in Hermes
    implementation("com.facebook.react:react-android")
    implementation("com.facebook.react:hermes-android")
}
```

In `android/gradle.properties`:
```
hermesEnabled=true
```

**Verify at runtime (JS):**
```js
const isHermes = () => !!global.HermesInternal;
console.log('Running on Hermes:', isHermes());
```

**Common mistakes:**
- Assuming Hermes supports all ES2022+ features — it lags slightly behind V8; check the compatibility table
- Forgetting to enable it in gradle.properties when upgrading older projects
- Debugging: Hermes has its own debugger protocol, not Chrome DevTools directly. Use Flipper or the new React Native DevTools

