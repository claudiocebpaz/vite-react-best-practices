# Changelog

All notable changes to the `vite-react-best-practices` skill are documented in this file.

## 2.0.0 — November 2026

Full refresh of the rule set against the current React and Vite ecosystem.

### Changed

- All 11 rules refreshed for React 19.3, Vite 8 (Rolldown + Oxc), React Router 8, and TanStack Query v5.
- `vite-spa-rewrites`: expanded host coverage (Nginx, Netlify, Vercel, Firebase, Cloudflare Pages, AWS S3 + CloudFront, GitHub Pages, Azure Static Web Apps).
- `vite-caching-strategy`: content-hashed asset names, CDN cache-key requirements, `modulepreload` preservation, `vite:preloadError` recovery.
- `vite-build-validation`: Rolldown/Oxc build pipeline, separate `tsc --noEmit` step, `vite preview` limits, minimal CI workflow.
- `vite-env-vars`: build-time inlining, `.env` modes and priority, built-in flags, runtime configuration, `ImportMetaEnv` augmentation.
- `react-route-splitting`: granular `route.lazy` on `createBrowserRouter`, lazy route discovery, Transitions, `React.lazy` fallback.
- `react-server-state`: TanStack Query v5 `queryOptions`, options-object `useQuery`, `useSuspenseQuery`, `placeholderData`, `staleTime` / `gcTime`.
- `react-memoization`: React Compiler guidance and the `reactCompilerPreset` setup via `@rolldown/plugin-babel`.
- `react-image-optimization`: LCP `fetchPriority`, `<picture>` format chain, `srcSet`/`sizes`, CLS-safe dimensions.
- `react-colocation`: feature folders with thin React Router route entries.
- `anti-import-dist`: `exports` maps, subpath imports, ESM/CJS interop under Rolldown.
- `vite-troubleshooting`: CJS default imports, `optimizeDeps`, plugin migration after the Rolldown/Oxc switch, HMR cycles, React 19 hydration errors, missing CSS.
- `SKILL.md`, `metadata.json`, `marketplace.json`, and `README.md` aligned with the refreshed rules and the new stack.

### Added

- `impact` and `tags` frontmatter on every rule file.
- A References section on every rule file, linking to the primary sources.
- `CLAUDE.md` symlink to `AGENTS.md` for Claude Code.
- A priority table in `SKILL.md` mapping impacts to rules.
- This `CHANGELOG.md`.

### Deferred

- A build system that compiles `AGENTS.md` from `rules/` instead of maintaining it by hand.
- Automated validation that the compiled guide matches the rule files.
- Evals for measuring whether the skill changes agent output.
- CI to run the validation and evals on every push.

## 1.1.0 — January 2026

### Changed

- Added the marketplace listing in `marketplace.json`.
- Improved the `README.md` with installation and usage sections, plus links to skills.sh and SkillsMP.

## 1.0.0 — January 2026

### Added

- Initial release with 11 rules across 3 categories: Vite SPA Deployment, React Core Performance, and Architecture & Cleanup.
- `SKILL.md` with rule categories, `metadata.json`, and the compiled guide in `AGENTS.md`.
