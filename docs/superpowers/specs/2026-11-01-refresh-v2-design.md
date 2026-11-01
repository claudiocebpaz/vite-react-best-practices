# Vite React Best Practices — v2.0.0 Content Refresh (Design)

Date: 2026-11-01
Status: Approved for implementation

## Context

This repository is an AI coding-agent skill (`vite-react-best-practices`) that ships
React + Vite SPA best practices. It was last refreshed in January 2026 (v1.1.0).

As of November 2026 the ecosystem has moved substantially:

| Piece | Skill baseline (Jan 2026) | Current (verified) |
|---|---|---|
| React | 19.x generic | 19.3.1 (19.3: View Transitions, Fragment Refs, browser(), Trusted Types) |
| Vite | 5/6 era | 8.3.5 (Vite 8 since Mar 2026, Rolldown-based) |
| React Router | v7 implicit | 8.5.1 (lazy route discovery, granular `route.lazy`) |
| TanStack Query | v5 | v5.105.1 (no v6 for React exists; verified on npm) |

A comparison with `vercel-labs/agent-skills` (`react-best-practices`) showed the two
skills are complementary: Vercel covers React/Next.js performance (RSC, server actions);
this skill covers Vite SPA deployment (rewrites, caching, env vars) which Vercel does not.
Vercel's conventions worth adopting: per-rule `impact`/`tags` frontmatter, per-rule
references, and a `CLAUDE.md` symlink to `AGENTS.md`.

## Decisions

- Scope: content refresh of the existing 11 rules. No new rule files, no structural change.
- Language: English (matches repo and public distribution).
- Versioning: `2.0.0` + new `CHANGELOG.md` (major content refresh, semver for skills).
- Adopt: rule frontmatter (`title`, `impact`, `tags`), `References` section per rule,
  `CLAUDE.md` → `AGENTS.md` symlink.
- Defer to phase 2 (out of scope): build system that compiles `AGENTS.md` from `rules/`,
  validation scripts, eval test cases, CI.

## Baseline (shared by every subagent)

- Target stack: React 19.3 + Vite 8.3 (Rolldown) + React Router 8.5 + TanStack Query v5 +
  strict TypeScript.
- Preserve each file's H1 and overall format: short intro, "Why it matters",
  Incorrect / Correct code examples in fenced blocks with language tags, English, no emojis.
- Add frontmatter to every file under `rules/`:
  ```yaml
  ---
  title: <Rule Title>
  impact: CRITICAL | HIGH | MEDIUM | LOW
  tags: tag1, tag2, tag3
  ---
  ```
- Add a `## References` section at the end of every rule with official documentation links
  (react.dev, vite.dev, reactrouter.com, tanstack.com/query, host docs).
- Facts must be verified against official documentation at edit time. If a claim cannot be
  verified, it is not asserted.
- Each agent edits only its assigned files. It does not rename files, does not touch
  `SKILL.md`, `AGENTS.md`, `README.md`, `metadata.json`, `marketplace.json`.

## Wave 1 — three parallel subagents

### Agent 1: Vite SPA Deployment (`vite-*`)

Files: `vite-spa-rewrites.md`, `vite-caching-strategy.md`, `vite-build-validation.md`,
`vite-env-vars.md`.

- **vite-spa-rewrites**: keep Nginx/Netlify/Vercel/Firebase; add Cloudflare Pages
  (`_redirects`), AWS S3 + CloudFront (`CustomErrorResponses` → `/index.html` with 200),
  GitHub Pages (`404.html` fallback), Azure Static Web Apps (`staticwebapp.config.json`
  navigation fallback).
- **vite-caching-strategy**: keep entry-point vs hashed-asset split; verify Vite 8 asset
  naming still content-hashed; add CDN cache-key note and `modulepreload` reminder.
- **vite-build-validation**: Vite 8 / Rolldown context; typecheck first
  (`tsc --noEmit`), build, `vite preview`; add a minimal CI example.
- **vite-env-vars**: keep `VITE_` prefix and security warning; add `.env.[mode]` files,
  `import.meta.env.MODE/DEV/PROD`, and note that runtime config requires an external
  mechanism (e.g. `/config.json` fetched at boot) because values are baked at build time.

### Agent 2: React Core Performance (`react-*`)

Files: `react-route-splitting.md`, `react-server-state.md`, `react-memoization.md`,
`react-image-optimization.md`.

- **react-route-splitting**: React Router 8 `route.lazy` (object-based granular lazy),
  lazy route discovery; keep `React.lazy` + `Suspense` for non-framework usage.
- **react-server-state**: TanStack Query v5 (`queryOptions`, `useSuspenseQuery`,
  `placeholderData` replacing `keepPreviousData`, `gcTime`); demote SWR to a brief mention.
- **react-memoization**: add React Compiler (v1+) auto-memoization; explain when manual
  `useMemo`/`useCallback` still matters (compiler not enabled, external consumers, effect
  dependencies); keep the "don't memoize primitives" guidance.
- **react-image-optimization**: LCP with `fetchpriority="high"`, responsive `srcset`/`sizes`,
  AVIF/WebP, explicit `width`/`height`, native lazy loading default; keep CLS guidance.

### Agent 3: Architecture & Cleanup

Files: `react-colocation.md`, `anti-import-dist.md`, `vite-troubleshooting.md`.

- **react-colocation**: keep feature-based structure; align examples with React Router 8
  route modules; shared UI stays in `components/`.
- **anti-import-dist**: add `exports` map / subpath exports guidance and ESM vs CJS notes;
  keep the "never import from dist" core.
- **vite-troubleshooting**: update for Vite 8/Rolldown (CJS interop, `optimizeDeps`,
  plugin compatibility), React 19 hydration mismatch errors, and the 404-on-refresh entry.

## Wave 2 — integration subagent (sequential, depends on Wave 1)

- Regenerate `AGENTS.md` mirroring all 11 updated rules (same sections, expanded content).
- `SKILL.md`: version `2.0.0`, description mentions React 19 / Vite 8 / React Router 8 /
  TanStack Query v5, add priority table (CRITICAL/HIGH/MEDIUM) and a "How to Use" section.
- `metadata.json`: version `2.0.0`, date `November 2026`, updated abstract and references.
- `marketplace.json`: updated description and tags.
- `README.md`: updated category list, stack versions, contribution note.
- `CHANGELOG.md`: new file with `2.0.0` entry (Changed / Added / Deferred) plus `1.1.0`
  and `1.0.0` history.
- Create `CLAUDE.md` symlink to `AGENTS.md`.

## Wave 3 — review and release

- Review full diff; verify AGENTS.md ↔ rules consistency, relative links, version strings,
  rule count (11) and category count (3).
- Verify no file was renamed and no Next.js/RSC content was introduced.
- Commit with conventional message and tag `v2.0.0`.

## Verification checklist

- [ ] `rules/` has 11 files, each with valid frontmatter and a References section
- [ ] All relative links in SKILL.md and README.md resolve
- [ ] AGENTS.md mirrors the rules and mentions current versions
- [ ] Version `2.0.0` consistent in SKILL.md and metadata.json
- [ ] CHANGELOG.md documents 2.0.0
- [ ] `CLAUDE.md` symlink resolves to AGENTS.md
- [ ] `git status` clean after commit; tag `v2.0.0` created

## Risks

- **Agent inconsistency** (style, versions): mitigated by the shared baseline above.
- **Factual drift** (stale versions, deprecated APIs): mitigated by mandatory verification
  against official docs at edit time.
- **AGENTS.md desync** (it is hand-maintained): mitigated by regenerating it from the rules
  in Wave 2 and checking consistency in Wave 3.
