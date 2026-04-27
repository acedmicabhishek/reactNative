# Basic Components in React Native

## Overview
React Native provides a set of **core primitive components** that map directly to native UI elements on Android (Android Views) and iOS (UIKit Views). Unlike web development, there is no DOM — React Native's components compile to 100% native platform widgets.

---

## 1. View
The most fundamental building block. Equivalent to `<div>` in web, `UIView` on iOS, and `android.view.View` on Android.

```jsx
import { View } from 'react-native';

// Acts as a container, supports Flexbox layout, touch handling, and styling
<View style={{ flex: 1, backgroundColor: '#fff', padding: 16 }}>
  {/* children go here */}
</View>
```

**Key Props:**
- `style` — Layout and visual styling via StyleSheet
- `onLayout` — Called when layout changes (useful for measuring dimensions)
- `accessible` / `accessibilityLabel` — Accessibility support
- `collapsable` — Android optimization. Set to `false` to prevent the OS from collapsing Views that serve only layout purposes.
- `pointerEvents` — Control how the View reacts to touch events: `'auto'`, `'none'`, `'box-only'`, `'box-none'`

**Under the Hood:**
When React Native renders a `<View>`, it sends instructions over the bridge (or via JSI in the New Architecture) to create a native `android.view.ViewGroup` or `UIView`. The layout is computed in C++ by the **Yoga** engine before the native view is mounted.

---

## 2. Text
Renders text. All text MUST be wrapped in a `<Text>` component (unlike web where raw strings inside `<div>` are valid).

```jsx
import { Text } from 'react-native';

<Text
  style={{ fontSize: 18, fontWeight: 'bold', color: '#1a1a1a' }}
  numberOfLines={2}          // truncate after 2 lines
  ellipsizeMode="tail"       // '...', where truncation happens
  selectable={true}          // allow user to long-press to copy
  onPress={() => alert('tapped')}
>
  Hello, React Native!
</Text>
```

**Key Props:**
- `numberOfLines` — Clamp text to N lines
- `ellipsizeMode` — `'head'`, `'middle'`, `'tail'`, `'clip'`
- `selectable` — Allow the user to copy text
- `allowFontScaling` — Respect system font size accessibility setting (default: `true`)

**Nesting Text:**
`<Text>` can be nested. Nested `<Text>` does NOT create a new layout box; it inherits styles from its parent and continues inline rendering.

```jsx
<Text style={{ color: 'black' }}>
  Normal text <Text style={{ fontWeight: 'bold' }}>BOLD</Text> normal again.
</Text>
```

---

## 3. Image
Renders a single image. Supports local (bundled) assets and remote URIs.

```jsx
import { Image } from 'react-native';

// Remote image
<Image
  source={{ uri: 'https://example.com/photo.jpg' }}
  style={{ width: 200, height: 200, borderRadius: 100 }}
  resizeMode="cover"
  onLoad={() => console.log('loaded')}
  onError={(e) => console.error(e.nativeEvent.error)}
/>

// Local bundled image (handled by Metro at build time)
<Image source={require('./assets/logo.png')} />
```

**`resizeMode` options:**
| Value | Behavior |
|-------|----------|
| `cover` | Scale uniformly to fill, cropping if needed |
| `contain` | Scale uniformly to fit without cropping |
| `stretch` | Stretch to fill exactly (distorts aspect ratio) |
| `repeat` | Tile the image (iOS only) |
| `center` | Center without scaling |

**Performance Tips:**
- Use `FastImage` (react-native-fast-image) for agressive caching and better performance on lists.
- Always specify explicit `width` and `height` to avoid layout thrashing.
- For remote images, include `Cache-Control` headers on your server.

---

## 4. ScrollView
A scrollable container. Renders **all** child components at once (not virtualized).

```jsx
import { ScrollView } from 'react-native';

<ScrollView
  horizontal={false}          // vertical by default
  showsVerticalScrollIndicator={false}
  contentContainerStyle={{ padding: 16 }}
  onScroll={(e) => {
    const { contentOffset } = e.nativeEvent;
    console.log(contentOffset.y); // scroll position in px
  }}
  scrollEventThrottle={16}    // fire onScroll at most every 16ms (~60fps)
  keyboardShouldPersistTaps="handled"  // tapping dismisses keyboard (unless child handles)
>
  {/* All children rendered immediately */}
</ScrollView>
```

**⚠️ Critical Warning:** ScrollView renders ALL children at once. If you have 1000 items, all 1000 are in memory. **Use FlatList for long lists.**

**Scroll Interaction with Keyboard:**
- `keyboardShouldPersistTaps="handled"` — Best for forms (buttons inside scroll still work when keyboard is open).
- Combine with `KeyboardAvoidingView` to shift the view up when the keyboard appears.

---

## 5. TextInput
A text input field. The native equivalent of `<input>` on web.

```jsx
import { TextInput, useState } from 'react';
import { TextInput as RNTextInput } from 'react-native';

const [text, setText] = useState('');

<RNTextInput
  value={text}
  onChangeText={setText}
  placeholder="Type here..."
  placeholderTextColor="#aaa"
  style={{ borderWidth: 1, borderColor: '#ccc', padding: 10, borderRadius: 8 }}
  keyboardType="email-address"   // 'default', 'numeric', 'email-address', 'phone-pad'
  returnKeyType="done"           // label on the keyboard's return key
  secureTextEntry={false}        // for passwords
  autoCapitalize="none"          // 'none', 'sentences', 'words', 'characters'
  multiline={false}              // allow multiple lines
  maxLength={100}                // max character count
  onSubmitEditing={() => { /* called when return key pressed */ }}
  onBlur={() => { /* input lost focus */ }}
  onFocus={() => { /* input gained focus */ }}
/>
```

**Under the Hood:**
`TextInput` renders a native `EditText` on Android and `UITextField`/`UITextView` on iOS. The text is kept in sync between JS and the native view. In the legacy architecture this happens asynchronously; in the new architecture (with Fabric) this sync is more direct.

**Controlled vs Uncontrolled:**
- **Controlled** (recommended): Pass `value` and `onChangeText` — React owns the state.
- **Uncontrolled**: Pass `defaultValue` only — native manages the value.

---

## 6. Other Essential Primitives
| Component | Description |
|-----------|-------------|
| `SafeAreaView` | Renders content within safe screen boundaries (notch, status bar, home indicator) |
| `StatusBar` | Controls the appearance of the top status bar |
| `Modal` | Renders content in a native modal overlay |
| `ActivityIndicator` | Native loading spinner |
| `Switch` | A native toggle switch |
| `KeyboardAvoidingView` | Shifts content up when keyboard appears |
