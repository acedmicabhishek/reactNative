# FlatList, SectionList & Virtualization in React Native

## Overview
Rendering long lists is one of the most common performance bottlenecks in React Native. The virtualization system (VirtualizedList/FlatList) ensures that only the items **visible on screen** (plus a small buffer) are rendered as native views — off-screen items are unmounted and their memory is reclaimed.

---

## 1. The Problem: ScrollView for Lists

`ScrollView` renders ALL children eagerly. For 1000 items this means:
- 1000 native Views created immediately
- All held in memory simultaneously
- Massive startup time and memory footprint

**Never use ScrollView for lists with more than ~20-30 items.**

---

## 2. FlatList

`FlatList` is the primary virtualized list component. Built on top of `VirtualizedList`.

### Basic Usage

```jsx
import { FlatList, View, Text } from 'react-native';

const DATA = Array.from({ length: 1000 }, (_, i) => ({ id: String(i), title: `Item ${i}` }));

const renderItem = ({ item, index }) => (
  <View style={{ padding: 16, borderBottomWidth: 1, borderColor: '#222' }}>
    <Text style={{ color: '#fff' }}>{item.title}</Text>
  </View>
);

<FlatList
  data={DATA}
  renderItem={renderItem}
  keyExtractor={(item) => item.id}   // MUST be unique per item
/>
```

### Critical Props (Performance)

```jsx
<FlatList
  data={DATA}
  renderItem={renderItem}
  keyExtractor={(item) => item.id}

  // --- PERFORMANCE ---
  initialNumToRender={10}        // items rendered on first paint (keep low)
  maxToRenderPerBatch={10}       // items rendered per batch during scroll
  windowSize={5}                 // render window = (windowSize * screen height) centered on viewport
  removeClippedSubviews={true}   // detach off-screen views (saves memory, can cause visual glitches)
  updateCellsBatchingPeriod={50} // ms between render batches
  getItemLayout={(data, index) => ({  // CRUCIAL for fixed-height items — skips measurement
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}

  // --- UX ---
  ListHeaderComponent={<Header />}
  ListFooterComponent={<Footer />}
  ListEmptyComponent={<EmptyState />}
  ItemSeparatorComponent={() => <View style={{ height: 1, backgroundColor: '#333' }} />}
  onEndReached={fetchMoreData}       // called when user scrolls near end
  onEndReachedThreshold={0.3}        // trigger when 30% from end
  refreshControl={                   // Pull-to-refresh
    <RefreshControl refreshing={isRefreshing} onRefresh={onRefresh} />
  }
  horizontal={false}                 // set true for horizontal lists
/>
```

### `getItemLayout` — The Most Important Optimization

If all your items have the **same fixed height**, implement `getItemLayout`. It tells FlatList the exact size and offset of each item without measuring them, enabling:
- Instant scroll-to-index
- Faster render batching

```jsx
const ITEM_HEIGHT = 72; // must be exact

getItemLayout={(data, index) => ({
  length: ITEM_HEIGHT,
  offset: ITEM_HEIGHT * index,
  index,
})}
```

For variable-height items, you cannot use `getItemLayout` — consider using `FlashList` instead.

---

## 3. SectionList

Like FlatList but with grouped data and sticky section headers.

```jsx
import { SectionList } from 'react-native';

const SECTIONS = [
  { title: 'React Native', data: ['View', 'Text', 'Image'] },
  { title: 'Expo', data: ['Camera', 'Location', 'FileSystem'] },
];

<SectionList
  sections={SECTIONS}
  keyExtractor={(item, index) => item + index}
  renderItem={({ item }) => <Text style={{ padding: 12 }}>{item}</Text>}
  renderSectionHeader={({ section: { title } }) => (
    <Text style={{ backgroundColor: '#111', padding: 8, fontWeight: 'bold', color: '#fff' }}>
      {title}
    </Text>
  )}
  stickySectionHeadersEnabled={true}  // section headers stick as you scroll past them
/>
```

---

## 4. FlashList (Recommended for Production)

[FlashList](https://shopify.github.io/flash-list/) by Shopify is a drop-in, faster replacement for FlatList. It recycles native views (like RecyclerView on Android) instead of creating and destroying them.

```bash
npx expo install @shopify/flash-list
```

```jsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={DATA}
  renderItem={({ item }) => <Text>{item.title}</Text>}
  estimatedItemSize={72}  // avg height estimate — unlike getItemLayout, approximate is fine
/>
```

**Why it's faster:**
- Recycles existing native `View` objects instead of unmounting/remounting
- Reduces GC pressure from creating/destroying JS and native objects
- Better initial render time and scroll FPS

---

## 5. VirtualizedList (Base Class)

Both FlatList and SectionList are built on `VirtualizedList`. Use it directly when your data structure is custom (not a simple array).

```jsx
import { VirtualizedList } from 'react-native';

<VirtualizedList
  data={myCustomDataSource}
  getItem={(data, index) => data.getItemAt(index)}  // custom item accessor
  getItemCount={(data) => data.totalCount}            // total item count
  renderItem={renderItem}
  keyExtractor={(item) => item.id}
/>
```

---

## 6. Common Pitfalls

### Anonymous Functions in `renderItem`
```jsx
// BAD: creates a new function on every render, breaking PureComponent/memo
<FlatList renderItem={({ item }) => <ItemComponent item={item} />} />

// GOOD: define outside or use useCallback
const renderItem = useCallback(({ item }) => <ItemComponent item={item} />, []);
<FlatList renderItem={renderItem} />
```

### Not Memoizing Row Components
```jsx
// BAD: re-renders every row when any state changes
const ItemComponent = ({ item }) => <View><Text>{item.title}</Text></View>;

// GOOD: only re-renders when item prop changes
const ItemComponent = React.memo(({ item }) => <View><Text>{item.title}</Text></View>);
```

### Using `key` instead of `keyExtractor`
`key` prop works but `keyExtractor` is the correct API — it's more performant and avoids reconciliation issues.

---

## 7. Profiling List Performance

Use **Flipper** or the **React DevTools Profiler** to detect:
- **Blank cells during fast scroll** → increase `windowSize` or `maxToRenderPerBatch`
- **Slow initial render** → decrease `initialNumToRender`, implement `getItemLayout`
- **Memory growth** → check if images in cells are being cached; use `FastImage`

```jsx
// FlatList debug mode — visualizes render windows in yellow/green
<FlatList debug={true} ... />
```

---

## 8. Under the Hood: How Virtualization Works

1. **JS Thread:** Computes which items fall within the `windowSize` render window.
2. **Items outside window:** Set to `display: none` (if `removeClippedSubviews: false`) or actually unmounted (if `true`).
3. **As user scrolls:** New items are enqueued for rendering in `maxToRenderPerBatch` chunks.
4. **Each batch:** Rendered asynchronously over `updateCellsBatchingPeriod` ms to avoid blocking the JS thread.

**The limitation:** All this orchestration runs on the JS thread. If your JS thread is doing heavy work, scroll blanking will occur. This is why FlashList's native recycling approach is superior.
