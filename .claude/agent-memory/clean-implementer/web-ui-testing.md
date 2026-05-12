# Web UI Testing Patterns (packages/web-ui)

## Stubbing Svelte 5 child components in vi.mock

When testing a parent Svelte 5 component (e.g. `App.svelte`) and you need to
stub child components, you cannot return a JS class from `vi.mock` -- Svelte 5
components are functions, and the harness errors with "Class constructors
cannot be invoked without 'new'".

The working pattern is to create real `.svelte` stub files and dynamically
import them in the mock factory. Example: `src/__test_stubs__/RouteStub.svelte`
and `src/__test_stubs__/Empty.svelte` (already in the repo).

```ts
vi.mock('./routes/Dashboard.svelte', async () => await import('./__test_stubs__/RouteStub.svelte'));
```

### The "$props with no read" gotcha

A stub component that calls `$props()` will trigger Svelte's
`state_referenced_locally` warning if the destructured binding is read
anywhere at module scope (`void _props`, `_props.foo`, etc). The fix is to
destructure into nothing so no binding is ever read:

```svelte
<script lang="ts">
  const {}: Record<string, unknown> = $props();
</script>
```

`$props()` is required to be a top-level `const` initializer (Svelte enforces
`props_invalid_placement`), so you cannot just call-and-discard it. The
empty-destructure trick satisfies both rules.

## Mocking `appState` for App.svelte

`App.svelte` reads many fields off `appState` from `./lib/stores.svelte.js`.
For chrome-level tests (sidebar/drawer), provide a `vi.hoisted` mock object
covering the read fields used by the connected layout. See
`src/App.test.ts` for the canonical shape (connected, hasToken, sessions,
pendingEscalations, etc.).

Also stub `matchMedia` globally before importing App -- the reduced-motion
detection runs synchronously at module evaluation and crashes jsdom otherwise:

```ts
vi.stubGlobal('matchMedia', vi.fn().mockImplementation(() => ({
  matches: false,
  addEventListener: vi.fn(),
  removeEventListener: vi.fn(),
})));
```

## Querying drawers/dialogs that toggle aria-hidden

When a drawer uses `aria-hidden={open ? undefined : 'true'}` to hide itself
from the a11y tree, `screen.getByRole('dialog', { name })` fails to find it
in the closed state. `getByLabelText('Main navigation')` works regardless of
aria-hidden state because it queries by `aria-label` directly.
