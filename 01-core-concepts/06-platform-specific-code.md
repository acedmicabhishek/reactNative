# Platform-Specific Code in React Native

## Overview
React Native targets multiple platforms (Android, iOS, and increasingly Web, Windows, macOS) from a single codebase. This guide covers how to write code that adapts or diverges per-platform when needed.

---

## 1. The `Platform` Module

```jsx
import { Platform } from 'react-native';

// Check the platform
console.log(Platform.OS); // 'android', 'ios', 'web', 'windows', 'macos'

// Boolean shortcuts
const isAndroid = Platform.OS === 'android';
const isIOS = Platform.OS === 'ios';

// Platform version
console.log(Platform.Version);
// Android: integer API level (e.g., 33 for Android 13)
// iOS: string (e.g., '16.1')

// Check if running on Android 12+
if (Platform.OS === 'android' && Platform.Version >= 31) {
  // Use Material You APIs
}
```

### `Platform.select()`
Returns the value from the key matching the current platform.

```jsx
const CONTAINER_STYLE = Platform.select({
  ios: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 4,
  },
  android: {
    elevation: 6,
  },
  default: {  // web, windows, macos
    boxShadow: '0px 2px 4px rgba(0,0,0,0.25)',
  },
});

// Also works for components
const InputComponent = Platform.select({
  ios: require('./IOSInput').default,
  android: require('./AndroidInput').default,
});
```

---

## 2. Platform-Specific File Extensions

React Native's Metro bundler resolves files with platform-specific extensions automatically. This is the cleanest way to have entirely different implementations per platform.

**File Naming Convention:**
```
Button.ios.js        ← loaded on iOS
Button.android.js    ← loaded on Android
Button.js            ← fallback for other platforms
```

**Usage (no changes to import):**
```jsx
// This import automatically picks the right file
import Button from './Button';
```

**Native-specific extensions:**
```
MyComponent.native.js   ← loaded on both iOS and Android (native), but NOT web
MyComponent.js          ← loaded on web
```

This is particularly useful when you have `react-native-web` and need different implementations for web vs. mobile.

---

## 3. Common Platform Divergences

### StatusBar
```jsx
import { StatusBar, Platform } from 'react-native';

// iOS — overlaps content by default (must account for it with SafeAreaView)
// Android — default height is 24dp, can be transparent

<StatusBar
  barStyle="light-content"   // 'default', 'dark-content', 'light-content'
  backgroundColor="#000000"  // Android only — status bar background color
  translucent={true}         // Android — content goes under status bar (like iOS)
/>
```

**Safe Area:**
```jsx
import { SafeAreaView } from 'react-native-safe-area-context';

// The recommended safe area library — works consistently across both platforms
// and accounts for notches, Dynamic Island, navigation bar, etc.
<SafeAreaView edges={['top', 'bottom']} style={{ flex: 1 }}>
  <Content />
</SafeAreaView>
```

### Shadows
```jsx
const styles = StyleSheet.create({
  card: {
    backgroundColor: '#1a1a1a',
    borderRadius: 12,
    // iOS shadow (4 separate properties)
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    // Android shadow (single number — maps to Material elevation)
    elevation: 8,
  },
});
```

### Fonts
Android and iOS ship with different system fonts:
- **iOS:** San Francisco (SF Pro) — accessed via `fontFamily: undefined` or `fontFamily: 'System'`
- **Android:** Roboto — accessed via `fontFamily: 'sans-serif'`

```jsx
const styles = StyleSheet.create({
  body: {
    fontFamily: Platform.select({
      ios: 'San Francisco',
      android: 'Roboto',
      default: 'system-ui',
    }),
  },
});
```

### Back Button (Android)
Android devices have a hardware/software back button. Handle it explicitly with the `BackHandler` API.

```jsx
import { BackHandler } from 'react-native';
import { useEffect } from 'react';

useEffect(() => {
  const subscription = BackHandler.addEventListener('hardwareBackPress', () => {
    if (canGoBack) {
      navigation.goBack();
      return true;  // true = we handled it, don't bubble to OS
    }
    return false;   // false = let OS handle it (exit app)
  });
  return () => subscription.remove();  // cleanup
}, [canGoBack]);
```

### Permissions
```jsx
import { PermissionsAndroid, Platform } from 'react-native';

// On iOS, permissions are declared in Info.plist and requested at runtime via a native prompt.
// On Android, permissions must be declared in AndroidManifest.xml AND requested at runtime (Android 6+).

const requestCameraPermission = async () => {
  if (Platform.OS === 'android') {
    const granted = await PermissionsAndroid.request(
      PermissionsAndroid.PERMISSIONS.CAMERA,
      { title: 'Camera Permission', message: 'App needs camera access', buttonPositive: 'OK' }
    );
    return granted === PermissionsAndroid.RESULTS.GRANTED;
  }
  // On iOS, this is handled by native permission dialogs automatically
  return true;
};
```

---

## 4. Android-Specific APIs

```jsx
// Haptic feedback (Android)
import { Vibration } from 'react-native';
Vibration.vibrate(100); // vibrate for 100ms

// Toast messages
import { ToastAndroid } from 'react-native'; // Android only!
ToastAndroid.show('Saved!', ToastAndroid.SHORT);

// Android ripple effect on Pressable
<Pressable android_ripple={{ color: 'rgba(255,255,255,0.2)', borderless: true }}>
```

---

## 5. iOS-Specific APIs

```jsx
// ActionSheetIOS — native iOS action sheet
import { ActionSheetIOS } from 'react-native'; // iOS only!

ActionSheetIOS.showActionSheetWithOptions(
  {
    options: ['Cancel', 'Remove', 'Share'],
    destructiveButtonIndex: 1,
    cancelButtonIndex: 0,
    title: 'Options',
  },
  (buttonIndex) => {
    if (buttonIndex === 1) removeItem();
    if (buttonIndex === 2) shareItem();
  }
);

// Haptic feedback (iOS — more nuanced than Android Vibration)
import * as Haptics from 'expo-haptics';
Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
```

---

## 6. `isAvailable` Pattern

For APIs that only exist on certain platforms, always guard with a check:

```jsx
// Pattern 1 — inline guard
if (Platform.OS === 'android') {
  ToastAndroid.show('Message', ToastAndroid.SHORT);
}

// Pattern 2 — cross-platform abstraction
const showToast = (message) => {
  if (Platform.OS === 'android') {
    ToastAndroid.show(message, ToastAndroid.SHORT);
  } else {
    // Use a cross-platform library like react-native-toast-message
    Toast.show({ type: 'success', text1: message });
  }
};
```

---

## 7. Key Takeaway

Platform divergence in React Native is **deliberate** — the goal is not just pixel-identical cross-platform code, but a UI that feels at home on each platform. iOS users expect bounce scrolling and swipe-back navigation; Android users expect Material ripples and the back button. The best React Native apps respect both.
