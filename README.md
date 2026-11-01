# Vite React Best Practices Skill

This repository contains the `vite-react-best-practices` skill, a comprehensive guide for building production-ready React Single Page Applications (SPAs) with Vite. This skill is designed to be used by AI coding assistants to provide guidance on performance, architecture, and deployment best practices.

The rules target React 19.3, Vite 8.3 (Rolldown + Oxc), React Router 8.5, TanStack Query 5, and React Compiler 1.x. Vite 8 requires Node `^20.19.0 || >=22.12.0`.

## What's Inside?

This skill provides 11 rules organized into the following categories:

### 1. Vite SPA Deployment (CRITICAL)
- **Static Rewrites** (`CRITICAL`): Mandatory for client-side routing. Nginx, Netlify, Vercel, Firebase, Cloudflare Pages, S3 + CloudFront, GitHub Pages, Azure Static Web Apps.
- **Caching Strategy** (`CRITICAL`): Immutable hashed assets, no-cache `index.html`, CDN cache keys, `vite:preloadError` recovery.
- **Build Validation** (`HIGH`): Typecheck, build, and preview before push; minimal CI workflow.
- **Environment Variables** (`HIGH`): `VITE_` prefix, modes, runtime config, `ImportMetaEnv` typing.

### 2. React Core Performance
- **Route Splitting** (`HIGH`): React Router 8 granular `route.lazy`, lazy route discovery, `React.lazy` + `Suspense`.
- **Server State** (`HIGH`): TanStack Query v5 `queryOptions`, `useQuery`, `useSuspenseQuery`, `staleTime` / `gcTime`.
- **Memoization** (`MEDIUM`): When `useMemo`/`useCallback` still pay off, and what React Compiler replaces.
- **Image Optimization** (`MEDIUM`): LCP priority hints, `srcSet`/`sizes`, CLS prevention.

### 3. Architecture & Cleanup
- **Colocation** (`MEDIUM`): Feature-based structure with thin route entries.
- **Anti-Patterns: Import from Dist** (`MEDIUM`): `exports` maps and ESM/CJS interop.
- **Troubleshooting** (`MEDIUM`): Common Vite 8 fixes, HMR, hydration errors, missing styles.

Each rule file in `rules/` is self-contained: frontmatter with `impact` and `tags`, rationale, an incorrect example, a correct example, and references.

For a complete, compiled version of all rules, see the `AGENTS.md` file (also available as `CLAUDE.md`).

## Installation

You can install this skill for your AI coding assistant using the `npx skills add` command:

```bash
npx skills add claudiocebpaz/vite-react-best-practices
```

This command will install the skill for all supported AI agents found on your system, such as Gemini CLI, Claude Code, OpenCode, and others.

## Usage

Once installed, your AI coding assistant will automatically use this skill when you ask for help with tasks related to Vite and React. For example, you can ask it to:

-   "Review my Vite project for performance issues."
-   "Help me set up route splitting in my React application."
-   "What are the best practices for caching in a Vite SPA?"

## Contributing

This skill is open source and contributions are welcome. If you have suggestions for improvements or new rules, please open an issue or a pull request in the [GitHub repository](https://github.com/claudiocebpaz/vite-react-best-practices).

See `CHANGELOG.md` for the release history.

## More Information

You can find more information about this skill and discover others on [skills.sh](https://skills.sh/claudiocebpaz/vite-react-best-practices) and [SkillsMP](https://skillsmp.com/).
