---
title: Correct Caching Strategy for Vite Assets
impact: CRITICAL
tags: vite, caching, cdn, performance, deployment
---

# Correct Caching Strategy for Vite Assets

Vite 8 emits content-hashed file names by default: `assets/[name]-[hash].js` for entry chunks and `assets/[name]-[hash].[ext]` for other assets. The hash is derived from the file contents, so a deployment that changes a file also changes its URL. That property is what makes an aggressive cache policy safe.

## Why it matters
Incorrect caching produces two opposite failures:
1. **Users see old code** after a deployment, because `index.html` was cached and still points to the previous build.
2. **Slow loading**, because hashed assets that can never be reused under the same URL are revalidated on every visit.

A single policy applied to both file types cannot solve both problems.

## Incorrect
Applying the same cache policy to every file.

```http
# Incorrect: caching everything for a week
Cache-Control: public, max-age=604800
```

## Correct
Split the files into two categories with different policies.

### 1. The entry point (`index.html`)
This file is the pointer to the current build. It must not be cached, or must be revalidated on every request.

```http
Cache-Control: no-cache, no-store, must-revalidate
```

At most, use `Cache-Control: public, max-age=0, must-revalidate`. Vite's build guide makes the same point for the missing-chunk case below: without `Cache-Control: no-cache` on the HTML, clients keep referencing deleted assets.

### 2. Hashed assets (`/assets/*.js`, `/assets/*.css`, images, fonts)
The file name changes whenever the content changes, so these can be cached for a year and marked immutable.

```http
Cache-Control: public, max-age=31536000, immutable
```

If you change `build.assetsDir` (default `assets`), update the host and CDN rules to match the new directory.

### Make the CDN cache key include the hashed file name
The policy above only works if the cache key distinguishes one hash from another. Keep the full request path in the cache key. A rule that normalizes `/assets/*` to one key, strips the file name, or versions assets only through a query string will serve a previous build's JavaScript under the current HTML, which fails at runtime with a chunk that no longer exists.

Do not rewrite hashed file names on upload either. The HTML references the exact names produced by the build.

### Keep the modulepreload tags for the entry chunk
Vite injects `<link rel="modulepreload">` tags for the entry chunk and its static imports, and adds the module preload polyfill by default (`build.modulePreload`). If you post-process `index.html` through an HTML minifier or a CDN transform, do not strip those tags: they start fetching the entry chunk before the parser reaches the module script. The preloaded URLs are hashed files and need the same immutable header as any other asset.

### Recover from chunks removed by a new deployment
Even with correct headers, a user with an open tab can request a chunk that the latest deployment deleted. Vite emits a `vite:preloadError` event for this case.

```ts
window.addEventListener('vite:preloadError', (event) => {
  event.preventDefault()
  window.location.reload()
})
```

## References
- [Vite: Building for Production](https://vite.dev/guide/build)
- [Vite: Build Options](https://vite.dev/config/build-options)
- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
