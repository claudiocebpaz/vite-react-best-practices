---
name: vite-react-best-practices
description: Comprehensive React 19 and Vite 8 SPA performance, architecture, and deployment guidelines. Use this skill when building, reviewing, or refactoring React applications built with Vite (SPA). Covers Vite 8 build and static hosting requirements, React Router 8 route-level code splitting, TanStack Query v5 server state, React Compiler memoization, and core React performance patterns.
license: MIT
metadata:
  author: ant-gravity
  version: "2.0.0"
---

# Vite React Best Practices

A senior-level guide for building production-ready React Single Page Applications (SPAs) with Vite.

## When to Apply

Reference these guidelines when:
- Setting up a new Vite + React project
- Configuring build pipelines and CI/CD for SPAs
- Troubleshooting production build or caching issues
- Refactoring React components for performance

## Priority Table

| Impact | Rules |
| --- | --- |
| CRITICAL | [Static Rewrites](rules/vite-spa-rewrites.md), [Caching Strategy](rules/vite-caching-strategy.md) |
| HIGH | [Build Validation](rules/vite-build-validation.md), [Environment Variables](rules/vite-env-vars.md), [Route Splitting](rules/react-route-splitting.md), [Server State](rules/react-server-state.md) |
| MEDIUM | [Memoization](rules/react-memoization.md), [Image Optimization](rules/react-image-optimization.md), [Colocation](rules/react-colocation.md), [Anti-Patterns: Import from Dist](rules/anti-import-dist.md), [Troubleshooting](rules/vite-troubleshooting.md) |

## Rule Categories

### 1. Vite SPA Deployment (CRITICAL)

- [Static Rewrites](rules/vite-spa-rewrites.md) - **Mandatory** for client-side routing.
- [Caching Strategy](rules/vite-caching-strategy.md) - Immutable assets, no-cache index.html.
- [Build Validation](rules/vite-build-validation.md) - Preview before push.
- [Environment Variables](rules/vite-env-vars.md) - `VITE_` prefix and security.

### 2. React Core Performance

- [Route Splitting](rules/react-route-splitting.md) - Lazy load pages with React Router 8 `route.lazy`.
- [Server State](rules/react-server-state.md) - TanStack Query v5 for cached server data.
- [Memoization](rules/react-memoization.md) - When to use useMemo/useCallback, and what React Compiler replaces.
- [Image Optimization](rules/react-image-optimization.md) - LCP and CLS prevention.

### 3. Architecture & Cleanup

- [Colocation](rules/react-colocation.md) - Feature-based structure.
- [Anti-Patterns: Import from Dist](rules/anti-import-dist.md) - Avoid bundling twice.
- [Troubleshooting](rules/vite-troubleshooting.md) - Common Vite 8 fixes.

## How to Use

1. Pick the rule that matches the task from the categories above, or from the priority table if you need to know what must not be skipped.
2. Open the rule file in `rules/`. Each file is self-contained: frontmatter with `impact` and `tags`, the rationale, an incorrect example, a correct example, and a References section.
3. Load only the rules relevant to the task. CRITICAL rules apply to every Vite SPA; HIGH rules apply to most production apps; MEDIUM rules apply when the area is being changed.
4. When a rule references another rule, follow the relative link inside `rules/`.

| Stack | Version |
| --- | --- |
| React | 19.3.1 |
| Vite | 8.3.5 (Rolldown + Oxc) |
| React Router | 8.5.1 |
| TanStack Query | 5.105.1 |
| React Compiler | `babel-plugin-react-compiler` 1.x |
| Node (Vite 8) | `^20.19.0 || >=22.12.0` |

## Full Compiled Document

For the complete guide with all rules expanded: `AGENTS.md` (also available as `CLAUDE.md`).
