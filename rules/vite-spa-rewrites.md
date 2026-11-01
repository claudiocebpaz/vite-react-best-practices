---
title: Configure Static Host Rewrites for SPA Routing
impact: CRITICAL
tags: vite, spa, routing, deployment, static-hosting
---

# Configure Static Host Rewrites for SPA Routing

A Vite SPA is a folder of static files. Every route below `/` (for example `/dashboard`) exists only in the browser, handled by React Router. When a user opens or refreshes that URL, the browser asks the host for that path, and the host looks for a file that does not exist.

## Why it matters
Failing to configure rewrites is the most common cause of "404 Not Found" (or "403 Forbidden" on S3) after a page refresh. It breaks deep linking, shared URLs, bookmarks, and any crawler that follows an internal link. The application code is fine; the host simply never hands `index.html` to the browser.

## Incorrect
Deploying the `dist/` output to a static host with its default behavior: serve the requested path, return an error when no file matches.

```nginx
# Nginx default (Incorrect for SPA)
location / {
  root /var/www/html;
  index index.html;
  # Returns 404 if the requested file does not exist
}
```

## Correct
Configure the host to serve `index.html` with a `200` status for any path that does not map to a real file. Real files (JS, CSS, images, fonts) must still be served directly, otherwise the app cannot boot.

### Nginx
`try_files` checks the filesystem first and falls back to the entry point.

```nginx
location / {
  root /var/www/html;
  try_files $uri $uri/ /index.html;
}
```

### Netlify (`netlify.toml`)
A rule with `status = 200` is a rewrite, not a redirect: the URL in the address bar does not change.

```toml
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

### Vercel (`vercel.json`)
```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

### Firebase (`firebase.json`)
```json
{
  "hosting": {
    "public": "dist",
    "rewrites": [{ "source": "**", "destination": "/index.html" }]
  }
}
```

### Cloudflare Pages (`_redirects`)
Create `public/_redirects` so Vite copies it to the root of `dist/`. The trailing `200` turns the rule into a proxy/rewrite.

```bash
# public/_redirects
/* /index.html 200
```

### AWS S3 + CloudFront
Upload the contents of `dist/` to the bucket, set the distribution's default root object to `index.html`, and add **custom error responses** for both `403` and `404` that return `/index.html` with an HTTP response code of `200`. CloudFront caches error responses for the error caching minimum TTL (10 seconds by default), so keep that value low while iterating.

```yaml
# CloudFormation fragment: AWS::CloudFront::Distribution DistributionConfig
CustomErrorResponses:
  - ErrorCode: 403
    ResponseCode: 200
    ResponsePagePath: /index.html
    ErrorCachingMinTTL: 10
  - ErrorCode: 404
    ResponseCode: 200
    ResponsePagePath: /index.html
    ErrorCachingMinTTL: 10
```

Mapping both codes matters: depending on the origin type and bucket permissions, S3 can answer a missing key with `403` instead of `404`.

### GitHub Pages
GitHub Pages serves [`404.html`](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-custom-404-page-for-your-github-pages-site) for unknown paths, so publish a copy of the built entry point under that name. Add the copy step after the build:

```bash
vite build
cp dist/index.html dist/404.html
```

Also set `base` to `/<REPO>/` when the site is served from a project subpath such as `https://<USERNAME>.github.io/<REPO>/`. The response still carries a `404` status code, which is acceptable for client-rendered apps but not for crawlers that require a `200`.

### Azure Static Web Apps (`staticwebapp.config.json`)
`navigationFallback` returns the entry point for requests that do not match a deployed file. Route rules are not applied to requests handled by the fallback.

```json
{
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/assets/*"]
  }
}
```

## References
- [Vite: Deploying a Static Site](https://vite.dev/guide/static-deploy)
- [Cloudflare Pages: Redirects](https://developers.cloudflare.com/pages/configuration/redirects/)
- [Amazon CloudFront: Change response codes returned by CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/custom-error-pages-response-code.html)
- [Azure Static Web Apps: Configuration](https://learn.microsoft.com/en-us/azure/static-web-apps/configuration)
