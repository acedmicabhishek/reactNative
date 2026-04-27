# Metro Bundler: Deep Dive

## Overview
Metro is the **JavaScript bundler** used by React Native. It takes your JS/TS source files (and all their dependencies) and produces a single JavaScript bundle file that the app loads at runtime. Understanding Metro's internals lets you optimize build times, bundle size, and startup performance.

---

## 1. What Metro Does

```
Source Files (.js, .ts, .jsx, .tsx, images, fonts)
         │
         ▼
┌─────────────────────────────────────────────┐
│                   METRO                      │
│                                             │
│  1. Resolution    — find all files          │
│  2. Transformation — transpile / compile    │
│  3. Serialization  — bundle into one file   │
└─────────────────────────────────────────────┘
         │
         ▼
Single JS bundle (index.android.bundle / index.ios.bundle)
+ Source Maps (.map)
+ Asset registry (images, fonts, etc.)
```

---

## 2. The Three Phases in Detail

### Phase 1: Resolution

Metro starts from your entry point (`index.js`) and recursively resolves all `require()` / `import` statements.

```
index.js
  → import App from './App'
       → import Screen from './screens/HomeScreen'
             → import { Button } from 'react-native'
                   → react-native/Libraries/Components/Button/Button.js
                         → ...
```

**Resolution order for an import `./Button`:**
1. `./Button.js`
2. `./Button.jsx`
3. `./Button.ts`
4. `./Button.tsx`
5. `./Button/index.js`
6. Platform-specific: `./Button.android.js` (on Android)

**`metro.config.js` — customizing resolution:**
```js
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);

// Add custom file extensions
config.resolver.sourceExts.push('mjs', 'cjs');

// Custom asset extensions
config.resolver.assetExts.push('bin', 'pb');  // e.g., TFLite models

// Custom module resolution (monorepo support)
config.resolver.extraNodeModules = {
  '@shared': path.resolve(__dirname, '../shared/src'),
};

// Block certain modules from being bundled
config.resolver.blacklistRE = /node_modules\/some-server-only-lib\/.*/;

module.exports = config;
```

### Phase 2: Transformation

Each resolved file is **transformed** (transpiled) into a format the JS engine can run.

```
TypeScript (.tsx)
  → Babel / SWC transform
  → JSX → React.createElement() calls
  → TypeScript types stripped
  → Modern ES2022 syntax → ES5-compatible
  → Hermes-specific transforms (optional)
  → CommonJS module format (require/exports)
```

**Transformation is cached.** Metro caches each file's transform result keyed by the file content hash. Only changed files are re-transformed.

**Babel config for React Native:**
```js
// babel.config.js
module.exports = {
  presets: ['babel-preset-expo'],
  plugins: [
    // Inline environment variables at build time
    ['transform-inline-environment-variables', { include: ['NODE_ENV', 'EXPO_PUBLIC_*'] }],
    // Remove console.log in production
    ['transform-remove-console', { exclude: ['error', 'warn'] }],
    // Reanimated plugin (MUST be last)
    'react-native-reanimated/plugin',
  ],
};
```

**Using SWC instead of Babel:**
SWC is a Rust-based compiler that's 20-70x faster than Babel. Expo has experimental SWC support:
```js
// app.json
{
  "expo": {
    "experiments": { "turboSources": true }  // enables SWC/Turbopack-based transforms
  }
}
```

### Phase 3: Serialization

All transformed modules are combined into a single bundle file.

**Module format:**
```js
// Each module is wrapped in a factory function
__d(function(global, _$$_REQUIRE, _$$_IMPORT_DEFAULT, _$$_IMPORT_ALL, module, exports, _dependencyMap) {
  // Module code here
  var _react = _$$_REQUIRE(_dependencyMap[0]); // require('react')
}, [/* module ID */, [/* dependency IDs */]], /* module name */);

// Entry point invocation
__r(/* entry module ID */);
```

**Development vs Production bundles:**
- **Dev:** `__DEV__ = true`, no minification, inline source maps.
- **Production:** `__DEV__ = false`, minified (Terser), external source maps, Hermes bytecode compiled by Metro.

---

## 3. The Metro Dev Server

In development, Metro runs a local HTTP server:

```
Metro Dev Server: http://localhost:8081

GET /index.bundle?platform=android&dev=true
→ Metro builds the bundle on demand and streams it

GET /index.bundle?platform=android&dev=true&minify=false
→ Unminified bundle

GET /assets/./assets/logo.png?platform=android
→ Serves asset files

Websocket (ws://localhost:8081)
→ Hot Module Replacement (HMR) updates
→ Fast Refresh
```

### Fast Refresh

Metro's Fast Refresh sends incremental module updates over the WebSocket connection:

```
You edit App.tsx → file watcher detects change
  → Metro re-transforms only App.tsx (cached)
  → Sends diff over WebSocket to device
  → React Native replaces the module in-place
  → Components that use the module re-render
  → State is PRESERVED (if module exports only React components)
```

State is preserved across Fast Refresh updates — unless you change hooks, their order, or non-component exports.

---

## 4. Bundle Splitting (RAM Bundles / Hermes Lazy Loading)

For large apps, loading all JS at startup is slow. Metro supports splitting bundles:

### RAM Bundles (Legacy)
```js
// metro.config.js
config.serializer.createModuleIdFactory = () => {
  // Custom module ID strategy for RAM bundles
};
```

RAM Bundles (Random Access Memory Bundles) store each module separately so they can be loaded on-demand rather than all at once.

### Lazy Requires
```js
// Instead of:
import HeavyComponent from './HeavyComponent';

// Use lazy require (Metro can defer this module load):
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));
```

### Expo Router's Bundle Splitting
Expo Router (v3+) with Metro's new `bundleSplitting` option splits bundles per route:
```json
{
  "expo": {
    "experiments": { "typedRoutes": true }
  }
}
```
Each screen's code is loaded only when the user navigates to it.

---

## 5. Source Maps

Source maps map compiled/bundled code back to original TypeScript source lines. Essential for production crash debugging.

```bash
# Generate source map alongside bundle
metro bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output android/app/src/main/assets/index.android.bundle \
  --sourcemap-output android/app/src/main/assets/index.android.bundle.map
```

**Using source maps with Sentry:**
```bash
npx @sentry/wizard@latest -i reactNative
# Automatically uploads source maps to Sentry on each build
```

**Symbolicating a crash stack trace manually:**
```bash
npx react-native symbolicatestack --sourcemap index.android.bundle.map crashstack.txt
```

---

## 6. Asset Handling

Metro treats non-JS files (images, fonts, audio) as **assets**:

```js
// In JS:
const logo = require('./assets/logo.png');
// Metro resolves this to an asset object: { uri: 'asset://logo.png', width: 100, height: 100 }

// Metro generates multiple resolutions automatically:
// logo.png → default
// logo@2x.png → 2x density screens
// logo@3x.png → 3x density screens
```

At build time, Metro copies assets to the correct platform location and generates an asset registry.

---

## 7. Custom Metro Transformers

You can plug in custom transformers for non-standard file types:

```js
// metro.config.js — handle .glsl shader files
config.transformer.babelTransformerPath = require.resolve('./glslTransformer');
config.resolver.sourceExts.push('glsl', 'vert', 'frag');
```

```js
// glslTransformer.js
module.exports.transform = function({ filename, src }) {
  if (filename.endsWith('.glsl')) {
    // Convert GLSL shader to a JS string export
    return {
      code: `module.exports = ${JSON.stringify(src)};`,
      map: null,
    };
  }
  // Fall through to default Babel transform
  return require('metro-transform-plugins').defaultTransformer({ filename, src });
};
```

---

## 8. Build Performance Optimization

### Cache Warming
```bash
# Pre-warm Metro's transform cache
npx react-native start --reset-cache  # clear first, then warm on next run
```

### Disable Expensive Plugins in Dev
```js
// babel.config.js
module.exports = {
  plugins: [
    // Only run source map transforms in production
    ...(process.env.NODE_ENV === 'production' ? [
      ['transform-remove-console'],
    ] : []),
  ],
};
```

### Profile Metro Build Time
```bash
# Start Metro with profiling
METRO_BUNDLER_PROFILE=true npx expo start

# Or use --max-workers to limit parallelism on memory-constrained machines
npx expo start --max-workers 2
```

---

## 9. Metro vs. Other Bundlers

| Feature | Metro | Webpack | esbuild | Vite |
|---------|-------|---------|---------|------|
| React Native support | ✅ Native | Plugin | Plugin | Plugin |
| Fast Refresh | ✅ Built-in | Plugin | Limited | ✅ HMR |
| Hermes bytecode | ✅ | ❌ | ❌ | ❌ |
| Build speed | Moderate | Slow | Fast | Fast |
| Code splitting | Limited | Advanced | Basic | Advanced |
| Tree shaking | Limited | ✅ | ✅ | ✅ |
| RN asset handling | ✅ Native | Manual | Manual | Manual |

Metro is purpose-built for React Native — its asset handling, platform-specific resolution, and Hermes integration are unique to it.
