---
title: Strategic Memoization
impact: MEDIUM
tags: memoization, react-compiler, performance, rendering
---

# Strategic Memoization

React is fast by default. Over-memoizing (`useMemo`, `useCallback`, `memo`) can hurt performance more than it helps, because every memoized value costs a dependency comparison and a cache entry, and because the comparison itself is easy to get wrong.

## Why it matters

- **Too little:** Expensive calculations and wide re-render cascades block the main thread and make interactions feel slow.
- **Too much:** Code becomes harder to read (`useCallback` hell) and the overhead of memoization outweighs the re-render cost of simple components.

## React Compiler changes the default

With React Compiler enabled (stable, `babel-plugin-react-compiler` 1.x), manual `memo`, `useMemo`, and `useCallback` are usually unnecessary. The compiler analyzes components and hooks at build time and inserts memoization itself: it skips cascading re-renders of children and caches expensive calculations that run during render.

In a Vite project, the compiler runs as a Babel preset in front of the pipeline. `@vitejs/plugin-react` 6+ exposes `reactCompilerPreset`, which you pass to `@rolldown/plugin-babel`:

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react, { reactCompilerPreset } from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';

export default defineConfig({
  plugins: [react(), babel({ presets: [reactCompilerPreset()] })],
});
```

What the compiler does *not* let you skip:

- **Effect dependency stability.** The React docs still recommend `useMemo`/`useCallback` as the escape hatch when a memoized value feeds an effect dependency array and the effect must not re-fire.
- **Existing code.** Leave existing memoization in place unless you test the change; removing it can change the compiler's output.
- **Work outside React.** The compiler only memoizes components and hooks, and its memoization is not shared between them. A genuinely expensive pure function called from many components still deserves its own cache (or a `useMemo` per call site that was profiled as hot).
- **Opt-outs.** `"use no memo"` at the top of a component skips compilation for it while you debug.

Projects without the compiler keep the guidance below.

## Incorrect

Memoizing everything blindly.

```tsx
// Incorrect: Primitive values and simple objects don't need memoization
const value = useMemo(() => 42, []);
const style = useMemo(() => ({ color: 'red' }), []);

// Incorrect: Inline functions on lightweight HTML elements are fine
const handleClick = useCallback(() => {
  console.log('clicked');
}, []);

return <button onClick={handleClick}>Click me</button>;
```

## Correct

Memoize only where a measurement or a hard requirement justifies it.

### 1. Referential equality for dependencies

When a value or function is used in a `useEffect` dependency array, stability matters.

```tsx
const params = useMemo(() => ({ id: userId }), [userId]);

useEffect(() => {
  api.fetch(params);
}, [params]); // params must be stable to avoid re-running on every render
```

### 2. Expensive calculations

Heavy synchronous work that runs on every render, such as filtering a 5,000 item list.

```tsx
const sortedList = useMemo(() => {
  return largeArray.filter((item) => item.active).sort((a, b) => a.value - b.value);
}, [largeArray]);
```

### 3. Passing stable props to optimized children

If a child is wrapped in `memo`, a new function reference on every parent render breaks the optimization.

```tsx
import { memo, useCallback, useState } from 'react';

const HeavyComponent = memo(function HeavyComponent({
  onParamChange,
}: {
  onParamChange: (value: string) => void;
}) {
  return <input onChange={(event) => onParamChange(event.target.value)} />;
});

function Parent() {
  const [value, setValue] = useState('');

  const handleChange = useCallback((next: string) => {
    setValue(next);
  }, []);

  // memo() on HeavyComponent is only useful because handleChange is stable
  return <HeavyComponent onParamChange={handleChange} />;
}
```

Profile first. React DevTools' Profiler tells you whether a component re-renders often enough to matter, and the compiler's `Memo` badge tells you whether that component was already optimized automatically. Without either signal, plain code is the better default.

## References

- [React Compiler introduction](https://react.dev/learn/react-compiler/introduction)
- [React Compiler installation](https://react.dev/learn/react-compiler/installation)
- [`useMemo` reference](https://react.dev/reference/react/useMemo)
- [`memo` reference](https://react.dev/reference/react/memo)
