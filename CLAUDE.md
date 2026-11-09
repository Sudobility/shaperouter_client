# ShapeRouter Client

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit` / `git push` / `gh pr create` only when the user explicitly asks
> in that turn. Finishing a task is not permission to commit. Stay on the current
> branch; do not create feature branches for code changes.

React client library for the ShapeRouter API: a `ShapeRouterClient` API class plus
TanStack Query hooks that wrap it. Package: `@sudobility/shaperouter_client`.

## Stack

- TypeScript (strict), **ESM only**, target ES2020, `moduleResolution: "bundler"`.
- Runtime / package manager: **Bun** (do not use npm/yarn/pnpm to install).
- Build: `tsc` + a post-build ESM-import fixup script.
- Server state: TanStack React Query 5. Peer dep: React >= 18.
- Tests: Vitest + React Testing Library in jsdom.

## Commands (all defined in package.json)

```bash
bun install
bun run build         # tsc -p tsconfig.build.json + node scripts/fix-esm-imports.mjs
bun run test:unit     # vitest run   (NOTE: there is no `bun run test`)
bun run test:watch    # vitest watch
bun run typecheck     # tsc --noEmit
bun run lint          # eslint src --ext .ts,.tsx
bun run format        # prettier --write
bun run verify        # typecheck + lint + test:unit + build  (pre-commit gate)
```

## Architecture

```
src/
  index.ts        # public barrel: re-exports network, hooks, utils, local types
  types.ts        # FirebaseIdToken, UserApiKey* types, QUERY_KEYS constant
  network/
    ShapeRouterClient.ts   # API class, large (~1900 lines, ~50 async methods)
  hooks/          # one hook per resource (see table below)
  utils/
    shaperouter-helpers.ts # ShapeRouterApiError, header/url/query/error helpers
```

Hooks delegate to `ShapeRouterClient` internally; the client calls
`networkClient.get/post/...`. All HTTP goes through a `NetworkClient` interface
(from `@sudobility/di`) that the caller injects — this is what makes everything
mockable in tests (`MockNetworkClient` from `@sudobility/di/mocks`).

Hooks: `useKeys`, `useProjects`, `useEndpoints`, `useAnalytics`, `useSettings`,
`useUserApiKeys`, `useAiExecute`, `useEntities`, `useProviders` /
`useProviderModels`, `useStorageConfig`.

## Key conventions (do not break — downstream wrappers depend on them)

- **Hook signature**: `(networkClient, baseUrl, scopeParam, token, options?)` where
  `options` is `{ testMode?: boolean; enabled?: boolean }`. There is NO positional
  `testMode`/`autoFetch` argument — those live inside `options`.
- **Client constructor** takes one options object:
  `new ShapeRouterClient({ baseUrl, networkClient, testMode? })`.
- **Return shape** is consistent across hooks: data (named per hook, e.g. `keys`),
  `isLoading`, `error: string | null`, `refetch` (no args), `clearError`, `reset`,
  plus CRUD methods. The method is `refetch`, not `refresh`.
- **Mutations return `BaseResponse<T>` and do not throw**; they auto-invalidate the
  relevant `QUERY_KEYS` cache on success.
- **Null-gated fetching**: passing `null` for a scope param (`entitySlug`,
  `projectId`) or `token` sets `enabled: false`. Intentional.
- **Stable empty defaults**: each hook has a module-level `EMPTY_X` constant used as
  the data default. Do not inline `[]` — it causes re-renders.
- `ShapeRouterClient` methods throw `ShapeRouterApiError` on non-ok responses;
  hooks translate that into the `error` string.

## Adding a hook / client method

1. Add query key(s) to `QUERY_KEYS` in `src/types.ts`.
2. Add the method(s) to `ShapeRouterClient` (use `createAuthHeaders(token)` for
   authed routes; return `BaseResponse<T>`).
3. Create `src/hooks/useX.ts` following the signature + return-shape conventions;
   add a module-level `EMPTY_X` default.
4. Export from `src/hooks/index.ts` and (transitively) `src/index.ts`.
5. Add `src/hooks/useX.test.ts` using `MockNetworkClient` + a
   `QueryClientProvider` wrapper with `retry: false`.

## Gotchas

- **Published to GitHub Packages**, not public npm. `.npmrc` sets
  `@sudobility:registry=https://npm.pkg.github.com`; `publishConfig.access` is
  `"public"`. Publishing is done by CI (`.github/workflows/ci-cd.yml`,
  `publish-target: github`) on push to `main`/`develop` — not by a local
  `npm publish`.
- **ESM build needs the fixup script**: `tsc` under `moduleResolution: "bundler"`
  emits extensionless relative imports; `scripts/fix-esm-imports.mjs` rewrites them
  to explicit `.js` so Node's ESM loader resolves them. `build` runs both steps.
- `useAiExecute` is mutation-only (no caching); params are path-based
  (`organizationPath` / `projectName` / `endpointName`), not UUIDs.
- `useSettings` uses optimistic updates (`setQueryData`) instead of invalidation.
- `useStorageConfig` returns null (not throw) on 404.
- `useProviderModels` uses `keepPreviousData` to avoid flashing on provider switch.
- Scoping: keys/projects/endpoints/storage are entity-scoped; analytics/settings/
  userApiKeys are user-scoped; providers are global/public (no auth).
- `src/types.ts` temporarily redeclares `UserApiKey` / `UserApiKeyCreated` /
  `CurrentUser` (see TODO there) until the types package ships them; prefer the
  types package once available.
