# @sudobility/shaperouter_client

React client library for the ShapeRouter API. Ships a `ShapeRouterClient` API
class plus a set of TanStack Query hooks that wrap it with caching, loading/error
state, and cache invalidation.

- **Package**: `@sudobility/shaperouter_client` (ESM only)
- **Published to**: GitHub Packages (`npm.pkg.github.com`), not the public npm registry
- **License**: BUSL-1.1

## Installation

This package is scoped to the `@sudobility` GitHub Packages registry. Consumers
need an `.npmrc` with:

```
@sudobility:registry=https://npm.pkg.github.com
```

Then install with Bun (the package manager used by this repo):

```bash
bun add @sudobility/shaperouter_client
```

### Peer dependencies

These must be present in the consuming app:

- `react` >= 18
- `@tanstack/react-query` >= 5
- `@sudobility/shaperouter_types`
- `@sudobility/types`

HTTP requests go through a `NetworkClient` from `@sudobility/di`, which the
consumer supplies to every hook and to `ShapeRouterClient`.

## Usage

Every hook takes the same leading arguments: a `NetworkClient`, the API base URL,
a scope parameter (e.g. `entitySlug`), and an auth token. Passing `null` for the
scope or token disables fetching.

```typescript
import { useKeys } from '@sudobility/shaperouter_client';

const { keys, isLoading, error, refetch, createKey, deleteKey } = useKeys(
  networkClient,            // NetworkClient from @sudobility/di
  'https://api.shaperouter.com',
  entitySlug,               // string | null (null disables fetching)
  firebaseToken,            // FirebaseIdToken | null (null disables fetching)
  { testMode: false, enabled: true } // optional
);

await createKey({ key_name: 'OpenAI', provider: 'openai', api_key: 'sk-...' });
refetch(); // manual refresh (takes no arguments)
```

Direct use of the API class, without hooks:

```typescript
import { ShapeRouterClient } from '@sudobility/shaperouter_client';

const client = new ShapeRouterClient({ baseUrl, networkClient, testMode: false });
const response = await client.getKeys(entitySlug, token);
```

## Hooks

| Hook | Purpose | Scope |
|------|---------|-------|
| `useKeys` | LLM API key CRUD | entitySlug |
| `useProjects` | Project CRUD + project API key | entitySlug |
| `useEndpoints` | Endpoint CRUD | entitySlug + projectId |
| `useAnalytics` | Usage analytics (query only) | entitySlug |
| `useSettings` | User settings (optimistic update) | userId |
| `useUserApiKeys` | Personal `shroute_...` API key CRUD | userId |
| `useAiExecute` | AI inference (mutation only, no caching) | — |
| `useEntities` | Entity/org CRUD + members + invitations | — |
| `useProviders` | Public provider list (query only) | — |
| `useProviderModels` | Models for a provider (keepPreviousData) | providerId |
| `useStorageConfig` | Cloud storage config (404 → null) | entitySlug |

Each hook returns a consistent shape: data (named per hook, e.g. `keys`),
`isLoading`, `error` (`string | null`), `refetch`, `clearError`, `reset`, plus
hook-specific CRUD methods. Mutations return `BaseResponse<T>` (they do not
throw) and auto-invalidate the relevant query cache on success.

`QUERY_KEYS` (exported from the package) holds the TanStack Query cache keys;
see `src/types.ts`.

## Development

```bash
bun install
bun run build         # Compile to dist/ (tsc) + fix ESM import extensions
bun run build:watch   # Watch-mode build
bun run clean         # Remove dist/
bun run test:unit     # Run tests once (vitest run)
bun run test:watch    # Vitest watch mode
bun run typecheck     # tsc --noEmit
bun run lint          # ESLint
bun run lint:fix      # ESLint --fix
bun run format        # Prettier write
bun run format:check  # Prettier check
bun run verify        # typecheck + lint + test:unit + build (run before committing)
```

Tests use Vitest + React Testing Library in a jsdom environment, and mock the
network layer with `MockNetworkClient` from `@sudobility/di/mocks`.

## Build notes

`bun run build` runs `tsc` (with `moduleResolution: "bundler"`, which emits
extensionless relative imports) and then `scripts/fix-esm-imports.mjs`, which
rewrites those specifiers to explicit `.js` paths so Node's native ESM loader can
resolve them.

Publishing is handled by CI (`.github/workflows/ci-cd.yml`) on pushes to `main`
/ `develop`, targeting GitHub Packages.

## Related packages

- `@sudobility/shaperouter_types` — shared type definitions (peer dependency)
- `@sudobility/di` — `NetworkClient` interface and mocks

## License

BUSL-1.1
