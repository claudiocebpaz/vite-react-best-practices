---
title: Image Optimization for CLS and Performance
impact: MEDIUM
tags: images, lcp, cls, responsive-images, performance
---

# Image Optimization for CLS and Performance

Images are usually the largest assets on a page. Handled carelessly they shift the layout when they arrive, they delay the Largest Contentful Paint because the browser discovers them late or at low priority, and they consume bandwidth that interactions need.

## Why it matters

- **CLS (Cumulative Layout Shift):** Without reserved space the page "jumps" when an image arrives, and users lose their place or click the wrong control.
- **LCP (Largest Contentful Paint):** The hero image is the LCP element on most content pages. Late discovery, lazy loading, or an oversized file pushes LCP past the 2.5s target.
- **INP (Interaction to Next Paint):** Every byte spent on images that are too large competes with the work an interaction needs in order to paint its update, especially on slow connections.

## Incorrect

An `img` with no dimensions, no priority hint, and no format or size alternatives. Lazy loading the hero image makes it worse.

```tsx
// Incorrect: the browser cannot reserve space, and the LCP image
// is discovered late and fetched at whatever priority it guesses
<img src="/hero.jpg" alt="Hero" loading="lazy" />
```

## Correct

### The LCP image: high priority, sized, modern formats

```tsx
function Hero() {
  return (
    <picture>
      <source
        type="image/avif"
        srcSet="/hero-640.avif 640w, /hero-1280.avif 1280w, /hero-1920.avif 1920w"
        sizes="100vw"
      />
      <source
        type="image/webp"
        srcSet="/hero-640.webp 640w, /hero-1280.webp 1280w, /hero-1920.webp 1920w"
        sizes="100vw"
      />
      <img
        src="/hero-1280.jpg"
        srcSet="/hero-640.jpg 640w, /hero-1280.jpg 1280w, /hero-1920.jpg 1920w"
        sizes="100vw"
        alt="Team reviewing designs in the studio"
        width={1920}
        height={1080}
        fetchPriority="high"
        decoding="async"
        className="h-auto w-full"
      />
    </picture>
  );
}
```

- `fetchPriority="high"` (React's camelCase prop for `fetchpriority`) tells the browser that this image is the one that matters. Set it on one or two images at most; if everything is high priority, nothing is.
- Never put `loading="lazy"` on the LCP image: the browser only starts fetching it after layout decides it is near the viewport, which adds avoidable delay.
- `width` and `height` give the browser the intrinsic aspect ratio so it reserves space before the bytes arrive, which is what prevents CLS. `h-auto w-full` lets CSS keep that ratio while scaling.
- `<picture>` with AVIF and WebP sources is a fallback chain: modern browsers take the smallest format they support, and everyone else gets the JPEG in `<img>`. Keep the `<img>` `srcSet`/`sizes` so the fallback is also responsive.
- `sizes` must describe the real rendered width. A wrong `sizes` value defeats the whole `srcSet` negotiation and downloads the largest candidate.

React does not generate an automatic preload hint for an `<img>` inside `<picture>`, so if that hero is the LCP element, add the hint to the HTML shell yourself:

```html
<link
  rel="preload"
  as="image"
  href="/hero-1280.avif"
  type="image/avif"
  fetchpriority="high"
/>
```

### Below-the-fold images: lazy, but still sized

```tsx
function ProductCard({ product }: { product: Product }) {
  return (
    <img
      src={product.image}
      srcSet={`${product.imageSmall} 640w, ${product.imageLarge} 1280w`}
      sizes="(min-width: 768px) 320px, 100vw"
      width={640}
      height={480}
      loading="lazy"
      decoding="async"
      alt={product.name}
      className="h-auto w-full"
    />
  );
}
```

`loading="lazy"` defers the request until the image approaches the viewport, which saves bandwidth and keeps early requests free for the critical path. It only pays off when the element has explicit dimensions: without them the browser treats the image as zero-sized, can believe everything fits on screen, and loads all of it anyway.

## References

- [Optimize Largest Contentful Paint](https://web.dev/articles/optimize-lcp)
- [Browser-level image lazy loading](https://web.dev/articles/browser-level-image-lazy-loading)
- [Serve responsive images](https://web.dev/articles/serve-responsive-images)
- [React `<img>` reference](https://react.dev/reference/react-dom/components/img)
