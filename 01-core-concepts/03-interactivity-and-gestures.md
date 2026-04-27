# Interactivity, Touch & Gestures in React Native

## Overview
React Native has its own touch and gesture system that sits below JavaScript and interfaces directly with the native platform's touch responder system. Understanding this system is key to building smooth, responsive interactions that feel native — and to diagnosing dropped frames during gestures.

---

## 1. Basic Touch Components

### TouchableOpacity
The most commonly used touchable. Fades the wrapped content to a specified opacity on press. Implemented **in native** (not JS), so the visual feedback is immediate.

```jsx
import { TouchableOpacity, Text } from 'react-native';

<TouchableOpacity
  onPress={() => console.log('pressed')}
  onLongPress={() => console.log('long press')}
  onPressIn={() => console.log('finger down')}  // fires immediately on touch
  onPressOut={() => console.log('finger up')}
  activeOpacity={0.7}          // opacity when pressed (0.0 - 1.0)
  disabled={false}
  delayLongPress={500}         // ms before onLongPress fires
>
  <Text>Press Me</Text>
</TouchableOpacity>
```

### Pressable (Modern)
The recommended replacement for all Touchable* components. More flexible — you can conditionally style based on press state.

```jsx
import { Pressable } from 'react-native';

<Pressable
  onPress={handlePress}
  style={({ pressed }) => ({
    backgroundColor: pressed ? '#555' : '#333',
    padding: 16,
    borderRadius: 8,
    transform: [{ scale: pressed ? 0.97 : 1 }],
  })}
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}  // expand touch target
  android_ripple={{ color: '#ffffff30', borderless: false }} // Android ripple effect
>
  {({ pressed }) => (
    <Text style={{ color: pressed ? '#aaa' : '#fff' }}>Press Me</Text>
  )}
</Pressable>
```

**`hitSlop`:** Expands the touchable area beyond the visual bounds. Use this to ensure your touch targets are at least 44x44 points (Apple's HIG recommendation).

---

## 2. The Responder System (How Native Touches Work)

React Native has a **Gesture Responder System** — the native touch event pipeline that determines which component "owns" a touch.

### How It Works:
1. **Touch starts** → RN asks components from bottom to top: "do you want to be the responder?"
2. A component claims ownership by returning `true` from `onStartShouldSetResponder` or `onMoveShouldSetResponder`.
3. Only ONE component can be the active responder at a time.
4. The responder receives all subsequent touch events.

```jsx
<View
  onStartShouldSetResponder={(e) => true}  // claim on touch start
  onMoveShouldSetResponder={(e) => true}   // claim on touch move (if not already claimed)
  onResponderGrant={(e) => { /* we are the responder */ }}
  onResponderMove={(e) => {
    const { locationX, locationY } = e.nativeEvent;
  }}
  onResponderRelease={(e) => { /* touch ended */ }}
  onResponderTerminate={(e) => { /* stolen by another responder (e.g. ScrollView) */ }}
/>
```

This is the low-level API. In practice, use **React Native Gesture Handler** instead.

---

## 3. React Native Gesture Handler (RNGH)

The standard library for advanced gesture recognition. Unlike the built-in Responder System, RNGH runs **entirely in native code** on the UI thread, so gestures are silky smooth even when the JS thread is busy.

```bash
# Installation
npx expo install react-native-gesture-handler
```

```jsx
import { GestureDetector, Gesture } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';

const Pan = () => {
  const offsetX = useSharedValue(0);
  const offsetY = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((e) => {
      offsetX.value = e.translationX;
      offsetY.value = e.translationY;
    })
    .onEnd(() => {
      offsetX.value = withSpring(0);  // snap back
      offsetY.value = withSpring(0);
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: offsetX.value },
      { translateY: offsetY.value },
    ],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: 'coral' }, animatedStyle]} />
    </GestureDetector>
  );
};
```

### Gesture Types
| Gesture | Use Case |
|---------|----------|
| `Gesture.Tap()` | Single, double, N-tap |
| `Gesture.LongPress()` | Long press with configurable delay |
| `Gesture.Pan()` | Drag and drop |
| `Gesture.Pinch()` | Zoom in/out |
| `Gesture.Rotation()` | Two-finger rotation |
| `Gesture.Fling()` | Quick swipe |

### Composing Gestures

```jsx
// Run simultaneously (e.g., pan + pinch)
const composed = Gesture.Simultaneous(panGesture, pinchGesture);

// One blocks the other
const exclusive = Gesture.Exclusive(tapGesture, panGesture);

// First one to activate wins
const race = Gesture.Race(tapGesture, panGesture);
```

---

## 4. Reanimated (Worklets)

React Native Reanimated allows animations to run entirely on the **UI thread** via "worklets" — small JS functions that are compiled and transferred to the native UI thread.

```jsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
  withRepeat,
  Easing,
  runOnJS,  // call a JS function from a worklet
} from 'react-native-reanimated';

const opacity = useSharedValue(1);

// Shared values: like useRef, but readable on both JS and UI threads
// Changing them directly on the UI thread (in a gesture handler/worklet)
// does NOT require a JS-thread roundtrip — this is why it's 60/120fps.

const style = useAnimatedStyle(() => ({
  opacity: opacity.value,
  transform: [{ scale: withSpring(opacity.value) }],
}));

// Trigger from JS thread:
opacity.value = withTiming(0, { duration: 300, easing: Easing.out(Easing.exp) });
```

**Why this is fast:** The `useAnimatedStyle` worklet is serialized and runs directly on the UI thread. It does not need the JS thread to compute the next frame value — each frame is computed natively.

---

## 5. The ScrollView Touch Conflict

A common problem: a horizontal gesture inside a vertical `ScrollView`. The scroll view and the gesture handler both compete for the same touch.

**Solution: `simultaneousHandlers` and `waitFor`:**
```jsx
const scrollRef = useAnimatedRef(); // Reanimated's ref to ScrollView

const panGesture = Gesture.Pan()
  .activeOffsetX([-20, 20])  // only activate if moved horizontally 20px
  .failOffsetY([-5, 5])      // fail if moved vertically 5px
  .simultaneousWithExternalGesture(scrollRef);
```

---

## 6. Performance Tips

- **Always use RNGH + Reanimated** for gesture-driven animations. The JS thread is too slow for per-frame animation callbacks.
- **`useNativeDriver: true`** in the legacy `Animated` API — offloads `opacity` and `transform` animations to the UI thread. Does NOT support `backgroundColor` or `height`.
- **Avoid JS functions in `onResponderMove`** for anything that needs to update UI — it's called on the JS thread and will drop frames if your JS is busy.
- **`InteractionManager.runAfterInteractions()`** — defers expensive operations until after animations and interactions have completed.
