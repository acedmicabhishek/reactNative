# State Management in React Native

## Overview
React Native inherits React's state system. Understanding how state updates trigger re-renders — and how to minimize unnecessary renders — is critical for performance. This guide covers hooks, Context API, and external state libraries (Redux, Zustand).

---

## 1. useState

Local component state. A state update schedules a re-render of that component and all its children.

```jsx
import { useState } from 'react';

const Counter = () => {
  const [count, setCount] = useState(0);

  return (
    <Pressable onPress={() => setCount(c => c + 1)}>  {/* functional update — always use prev state */}
      <Text>{count}</Text>
    </Pressable>
  );
};
```

**Functional updates:** Use `setCount(c => c + 1)` instead of `setCount(count + 1)` to avoid stale closure bugs in async handlers.

**Batching (React 18+):** React 18 batches multiple `setState` calls in the same event handler into a single re-render. In RN, this is enabled via the New Architecture.

---

## 2. useEffect

Handles side effects: data fetching, subscriptions, timers, syncing with native modules.

```jsx
import { useState, useEffect } from 'react';

const Profile = ({ userId }) => {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let cancelled = false;

    async function fetchUser() {
      const data = await api.getUser(userId);
      if (!cancelled) setUser(data);  // guard against setting state after unmount
    }

    fetchUser();

    return () => {
      cancelled = true;  // cleanup function — runs on unmount or before next effect
    };
  }, [userId]);  // dependency array — re-run when userId changes
};
```

**Dependency array rules:**
- `[]` — run once on mount only
- `[a, b]` — run when `a` or `b` changes
- No array — run after every render (usually a bug)

**Common mistakes:**
- Missing deps (stale closure): the effect uses a value but doesn't list it as a dep.
- Infinite loop: effect updates a state that is listed as a dep.
- Not cleaning up subscriptions/timers → memory leaks.

---

## 3. useRef

Persists a value across renders **without causing a re-render**. Also used to reference native component instances.

```jsx
const timerRef = useRef(null);
const inputRef = useRef(null);

// Access native component (focus a TextInput programmatically)
<TextInput ref={inputRef} />
inputRef.current?.focus();

// Mutable value that doesn't trigger re-render
useEffect(() => {
  timerRef.current = setTimeout(() => doSomething(), 1000);
  return () => clearTimeout(timerRef.current);
}, []);
```

---

## 4. useContext — Shared State Without Props Drilling

```jsx
// 1. Create Context
const ThemeContext = React.createContext({ dark: false, toggle: () => {} });

// 2. Provide it high in the tree
const App = () => {
  const [dark, setDark] = useState(false);
  return (
    <ThemeContext.Provider value={{ dark, toggle: () => setDark(d => !d) }}>
      <MainContent />
    </ThemeContext.Provider>
  );
};

// 3. Consume anywhere in the subtree
const Button = () => {
  const { dark, toggle } = useContext(ThemeContext);
  return (
    <Pressable onPress={toggle} style={{ backgroundColor: dark ? '#000' : '#fff' }}>
      <Text>Toggle Theme</Text>
    </Pressable>
  );
};
```

**⚠️ Context Performance Warning:**
Every component that consumes a context re-renders when **any value in that context changes**. Split contexts by concern (e.g., `ThemeContext`, `UserContext`, `CartContext`) to minimize unnecessary re-renders.

---

## 5. useMemo & useCallback

Memoize expensive computations and stable function references.

```jsx
const sortedList = useMemo(() => {
  return rawList.slice().sort((a, b) => a.score - b.score);
}, [rawList]); // only recomputes when rawList changes

// Stable function reference — prevents child re-renders
const handlePress = useCallback(() => {
  navigate('Detail', { id: item.id });
}, [item.id]);
```

**Rule of thumb:**
- Use `useMemo` for expensive calculations or object/array creation that is passed to memoized children.
- Use `useCallback` for functions passed as props to memoized children or listed in `useEffect` deps.
- Don't over-memoize — the hook itself has overhead.

---

## 6. React.memo

Prevents a component from re-rendering if its props haven't changed.

```jsx
const ItemCard = React.memo(({ item, onPress }) => {
  return (
    <Pressable onPress={onPress}>
      <Text>{item.title}</Text>
    </Pressable>
  );
});

// Custom comparison function for complex props
const ItemCard = React.memo(({ item }) => { ... }, (prevProps, nextProps) => {
  return prevProps.item.id === nextProps.item.id; // true = skip re-render
});
```

---

## 7. Zustand (Recommended for Most Apps)

Minimal, fast, and unopinionated global state. No boilerplate, no providers (unless needed).

```bash
npm install zustand
```

```jsx
import { create } from 'zustand';

const useAuthStore = create((set, get) => ({
  user: null,
  token: null,
  isLoading: false,

  login: async (credentials) => {
    set({ isLoading: true });
    const { user, token } = await api.login(credentials);
    set({ user, token, isLoading: false });
  },

  logout: () => set({ user: null, token: null }),

  // Access current state in actions
  isLoggedIn: () => get().token !== null,
}));

// Usage — component only re-renders if `user` changes
const Header = () => {
  const user = useAuthStore((state) => state.user);
  const logout = useAuthStore((state) => state.logout);
  return <Text onPress={logout}>{user?.name}</Text>;
};
```

**Zustand with Persistence (persisting to AsyncStorage):**
```jsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

const useSettingsStore = create(
  persist(
    (set) => ({
      theme: 'dark',
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);
```

---

## 8. Redux Toolkit (For Large / Complex Apps)

Industry standard for large-scale apps. More boilerplate than Zustand but provides predictability, DevTools, and a clear pattern for teams.

```bash
npm install @reduxjs/toolkit react-redux
```

```jsx
// store/slices/counterSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// Async action (thunk)
export const fetchUsers = createAsyncThunk('users/fetchAll', async () => {
  const response = await api.getUsers();
  return response.data;
});

export const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0, status: 'idle' },
  reducers: {
    increment: (state) => { state.value += 1; }, // Immer allows "mutations"
    decrement: (state) => { state.value -= 1; },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => { state.status = 'loading'; })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.users = action.payload;
      });
  },
});

export const { increment, decrement } = counterSlice.actions;
```

```jsx
// App.jsx — wrap with Provider
import { Provider } from 'react-redux';
import { store } from './store';
<Provider store={store}><App /></Provider>

// Component
import { useSelector, useDispatch } from 'react-redux';
const count = useSelector((state) => state.counter.value);
const dispatch = useDispatch();
dispatch(increment());
```

---

## 9. State Architecture Patterns

| Pattern | When to Use |
|---------|-------------|
| `useState` | Local, isolated component state |
| `useContext` | Lightweight shared state (theme, locale) |
| **Zustand** | App-wide state, simple API, most projects |
| **Redux Toolkit** | Large teams, complex async flows, time-travel debugging |
| React Query / SWR | Server state — caching, background sync, pagination |

**Server State is Different:**
Separate your server cache (API data) from your UI state. Use **TanStack Query (React Query)** for server state — it handles caching, deduplication, background refetching, and loading/error states automatically.

---

## 10. Performance Profiling

Use the **React DevTools Profiler** (in Metro or Flipper) to identify which components are re-rendering unnecessarily. Look for:
- Components that render with identical props (needs `React.memo`)
- A parent re-rendering all children because it holds a state that only one child needs (lift state down)
- Context consumers re-rendering on unrelated context changes (split the context)
