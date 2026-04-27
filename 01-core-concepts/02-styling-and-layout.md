# Styling & Layout in React Native

## Overview
React Native uses a **subset of CSS** implemented in JavaScript via `StyleSheet`. There is no CSS file system — all styles are JavaScript objects. Layout is handled by the **Yoga** engine (a C++ port of Flexbox), which runs on a separate thread before any native views are created.

---

## 1. StyleSheet API

Always use `StyleSheet.create()` rather than inline objects. It validates styles in development and enables optimizations in production (styles are frozen and IDs are sent over the bridge instead of full objects on each render).

```jsx
import { StyleSheet, View, Text } from 'react-native';

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#0f0f0f',
    padding: 16,
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    color: '#ffffff',
    letterSpacing: 0.5,
  },
});

// Usage
<View style={styles.container}>
  <Text style={styles.title}>Hello</Text>
</View>
```

**Combining Styles:**
```jsx
// Array syntax — last style wins on conflict
<View style={[styles.container, styles.darkMode, { marginTop: 10 }]} />
```

**StyleSheet.flatten():** Merges an array of styles into a single object.

---

## 2. Flexbox in React Native

React Native uses Flexbox for **all** layout — there are no floats, grids (by default), or block/inline elements. The Yoga engine computes all Flexbox layouts in C++.

### Key Differences from Web CSS Flexbox

| Property | Web Default | React Native Default |
|----------|-------------|----------------------|
| `flexDirection` | `row` | **`column`** |
| `alignContent` | `stretch` | `flex-start` |
| `flexShrink` | `1` | **`0`** |
| Units | `px`, `%`, `em`, `rem`... | **Density-independent pixels (dp) only** |

### Core Flexbox Properties

```jsx
<View style={{
  flex: 1,                     // Take all available space on the main axis
  flexDirection: 'column',     // 'row', 'column', 'row-reverse', 'column-reverse'
  justifyContent: 'center',    // Align along MAIN axis: 'flex-start', 'flex-end', 'center', 'space-between', 'space-around', 'space-evenly'
  alignItems: 'stretch',       // Align along CROSS axis: 'flex-start', 'flex-end', 'center', 'stretch', 'baseline'
  alignSelf: 'auto',           // Override alignItems for THIS child
  flexWrap: 'nowrap',          // 'wrap', 'nowrap', 'wrap-reverse'
  gap: 12,                     // Space between children (supported in RN 0.71+)
}}>
```

### `flex` vs `flexGrow` / `flexShrink` / `flexBasis`

```jsx
// flex: 1 is shorthand for flexGrow: 1, flexShrink: 1, flexBasis: 0
// Use explicit properties for precise control:
<View style={{ flexGrow: 1, flexShrink: 0, flexBasis: 200 }} />
```

### Absolute Positioning

```jsx
<View style={{ position: 'absolute', top: 0, right: 0, bottom: 0, left: 0 }}>
  {/* Overlays parent completely */}
</View>

// Shorthand (RN 0.71+)
<View style={{ ...StyleSheet.absoluteFillObject }}>
```

---

## 3. Dimensions & Units

All numeric values are **density-independent pixels (dp)** — they scale automatically with screen density. There are NO `px`, `rem`, or `em` units.

```jsx
import { Dimensions, PixelRatio } from 'react-native';

const { width, height } = Dimensions.get('window');  // screen dimensions in dp
const { width: screenWidth } = Dimensions.get('screen'); // includes nav bar areas

// Listen for dimension changes (rotation, split screen)
Dimensions.addEventListener('change', ({ window }) => {
  console.log(window.width, window.height);
});

// Convert dp to physical pixels
const physicalPixels = PixelRatio.getPixelSizeForLayoutSize(100);
// On a 3x density screen, 100dp = 300 physical pixels

// Get the screen's pixel density ratio
const ratio = PixelRatio.get(); // e.g., 3 on iPhone 12 Pro
```

**Percentage values** are supported for dimensions and positions:
```jsx
<View style={{ width: '50%', height: '100%' }} />
```

---

## 4. Platform-Specific Styling

```jsx
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  shadow: {
    // iOS uses shadow properties
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 6,
    // Android uses elevation
    elevation: 8,
  },
  text: {
    // Conditional style based on platform
    ...Platform.select({
      ios: { fontFamily: 'San Francisco' },
      android: { fontFamily: 'Roboto' },
    }),
  },
});
```

---

## 5. Fonts & Typography

```jsx
// 1. Register font in your expo config (app.json) or link via react-native.config.js
// 2. Load font (in Expo):
import { useFonts } from 'expo-font';

const [fontsLoaded] = useFonts({
  'Inter-Bold': require('./assets/fonts/Inter-Bold.ttf'),
});

// 3. Use the font
<Text style={{ fontFamily: 'Inter-Bold', fontSize: 24 }}>Styled Text</Text>
```

**Font Weights on Android:** Android matches font weight using separate font files per weight. Unlike web, `fontWeight: '700'` alone won't bold a custom font — you must provide the bold variant and reference it by family name.

---

## 6. Transforms & Animations (Static)

```jsx
<View style={{
  transform: [
    { translateX: 50 },
    { translateY: -20 },
    { rotate: '45deg' },
    { scale: 1.2 },
    { scaleX: 0.8 },
    { skewX: '10deg' },
  ]
}} />
```

Transforms are computed in native and **do NOT trigger a layout pass** — they're cheap to animate via `Animated.Value` or Reanimated.

---

## 7. Performance Notes

- **Avoid inline style objects:** `style={{ color: 'red' }}` creates a new object on every render. Use `StyleSheet.create()`.
- **`StyleSheet.hairlineWidth`:** A pixel-perfect 1dp line on all screen densities. Better than hardcoding `0.5`.
- **Avoid deep nesting of Views:** Every extra View is a native object. The renderer flattens unnecessary Views, but deep nesting still costs layout computation time.
- **`overflow: 'hidden'` on Android** disables hardware layer rendering for that subtree — be aware of the tradeoff with clipping.

---

## Under the Hood: How Layout Works

1. Your `StyleSheet` styles are shipped to the **Shadow Thread** (C++).
2. The **Yoga engine** (C++) computes the exact size and position of every node in the component tree.
3. The resulting layout is sent to the **Main/UI Thread** which applies these values to native Views.

This is why layout is efficient — the heavy computation is done in C++ off the main thread.
