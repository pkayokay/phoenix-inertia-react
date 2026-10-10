---
name: phoenix-inertia-react
description: >-
  Phoenix + Inertia.js + React + TypeScript via the official `inertia` Hex package
  (inertiajs/inertia-phoenix). Assumes that stack. Use when installing Inertia,
  writing assign_prop/assign_shared_prop/render_inertia controllers, TSX pages/forms,
  shared props, deferred/merge/prepend/once/scroll props, Inertia.Errors /
  Inertia.Paginated protocols, validation errors, flash, CSRF, asset versioning,
  Inertia.Testing, or SSR. NEVER useEffect+fetch for page data. NEVER react-hook-form.
  External URLs use Phoenix redirect (auto-converted); same-origin non-Inertia routes
  need force_inertia_redirect.
---

# Phoenix + Inertia + React + TypeScript

Assumed stack: **Phoenix** + Hex **`inertia`** (`inertiajs/inertia-phoenix`) +
**React** + **TypeScript** (`@inertiajs/react`, pages as `.tsx`).

Phoenix owns routing, data, and auth. React renders typed props — not a
traditional SPA.

**Package name:** Hex dep is `{:inertia, "~> 3.0"}`. Do not confuse with the
older community `inertia_phoenix` package. Pair with **`@inertiajs/react` v3**.

**Target API:** stable v3.x (covers the full
[inertia-phoenix README](https://github.com/inertiajs/inertia-phoenix) and
[upgrading guide](https://hexdocs.pm/inertia/upgrading_to_v3.html)).

**Props casing:** Prefer `camelize_props: true` so Elixir `snake_case` becomes
JS `camelCase` matching TypeScript types. If disabled, TS prop names stay snake_case.

## Core Mental Model

1. Controller builds props → `render_inertia("Page/Name")`
2. TSX page receives those props (typed) and renders
3. Navigation and forms go through Inertia (`Link`, `Form`, `router`) — never a
   parallel client data layer

**Before building a feature, ask:**
- **Where does data come from?** Server → `assign_prop`. User UI chrome → `useState`.
- **Needed on every page?** Shared plug with `assign_shared_prop` (or `assign_prop`),
  not per-action duplication.
- **Expensive?** `inertia_defer` / `inertia_optional` / bare `fn -> ... end` or `&fun/0`.
- **Reaching for SPA patterns?** Check the decision matrix first.

## Decision Matrix

| Need | Solution | NOT This |
|------|----------|----------|
| Page data | `assign_prop` + `render_inertia` | `useEffect` + `fetch` |
| Global data (auth) | Shared plug `assign_shared_prop` | React Context / Redux |
| Flash | Auto top-level `usePage().flash` | `assign_prop(:flash, …)` / `props.flash` |
| Forms | `<Form>` / `useForm` | `fetch` / axios / react-hook-form |
| Navigate | `<Link>` / `router.visit` | react-router / `<a>` full reload |
| Refresh subset | `router.reload({ only: [...] })` | React Query / SWR |
| Expensive props | `inertia_defer` | Client loading + fetch |
| Infinite scroll | `inertia_scroll` + `<InfiniteScroll>` | Client-only pagination |
| Stable reference data | `inertia_once` | Cache in React state |
| Custom error types | `Inertia.Errors` protocol + `assign_errors` | Hand-rolled JSON error maps |
| Pagination library structs | `Inertia.Paginated` + `inertia_scroll` | Manual meta shaping every time |
| External / non-Inertia URL | `redirect(to: url)` or `force_inertia_redirect` | Broken 302 JSON parse |
| Ephemeral UI | `useState` | Server props |

## Critical Rules

| # | Rule | Why |
|---|------|-----|
| 1 | Never `useEffect`+`fetch` for page data | Second data lifecycle drifts from Inertia props |
| 2 | Never check auth only in React | Client guards are spoofable; plug/controller gate |
| 3 | Use `<Form>` / `useForm`, not react-hook-form | CSRF, redirects, error map, files already handled |
| 4 | Use `<Link>` / `router`, not bare `<a>` for app routes | Full reload destroys layout state |
| 5 | Always `assign_prop` / `assign_shared_prop` (or inline props map) — assigns are NOT auto-props | `conn.assigns` alone never reach React |
| 6 | Validation: `assign_errors(changeset)` then `redirect` | Errors preserved across redirect; flat keys for forms |
| 7 | CSRF: `createInertiaApp({ http: { xsrfHeaderName: "x-csrf-token" } })` | Phoenix expects that header; cookie is set by adapter |
| 8 | External redirects: Phoenix `redirect` is auto-converted; same-origin non-Inertia needs `force_inertia_redirect` | Else Inertia client mis-handles the response |
| 9 | Type page props explicitly; keep TS names in sync with `camelize_props` | Mismatched casing = silent `undefined` at runtime |
| 10 | Flash is top-level: `usePage().flash`, not `usePage().props.flash` | v3 page object; nesting under props is wrong |
| 11 | Production SSR requires Node + `NODE_ENV=production` (+ optional `nodejs` Hex dep) | Without it, SSR renders are extremely slow / missing |

## Quick Patterns

### Controller

```elixir
def index(conn, _params) do
  conn
  |> assign_prop(:users, Accounts.list_users())
  |> assign_prop(:stats, inertia_defer(&Accounts.stats/0))
  |> render_inertia("Users/Index")
end

def create(conn, %{"user" => params}) do
  case Accounts.create_user(params) do
    {:ok, user} ->
      conn
      |> put_flash(:info, "Created")
      |> redirect(to: ~p"/users/#{user}")

    {:error, changeset} ->
      conn
      |> assign_errors(changeset)
      |> redirect(to: ~p"/users/new")
  end
end
```

### React page (TSX)

```tsx
import { Head, Link } from "@inertiajs/react";

type Props = {
  users: { id: number; name: string }[];
};

export default function Index({ users }: Props) {
  return (
    <>
      <Head title="Users" />
      <ul>
        {users.map((u) => (
          <li key={u.id}>
            <Link href={`/users/${u.id}`}>{u.name}</Link>
          </li>
        ))}
      </ul>
    </>
  );
}
```

### Form (prefer `<Form>`)

Error keys match **changeset field names** (e.g. `"name"`), not the Phoenix
param nest (`user[name]`). Nested embeds use `"team.name"` / `"items[0].price"`.

```tsx
import { Form } from "@inertiajs/react";

export default function New() {
  return (
    <Form action="/users" method="post">
      {({ errors, processing }) => (
        <>
          <input type="text" name="user[name]" defaultValue="" />
          {errors.name && <p>{errors.name}</p>}
          <button type="submit" disabled={processing}>
            Save
          </button>
        </>
      )}
    </Form>
  );
}
```

## Prop Helpers Cheat Sheet

| Helper | Behavior |
|--------|----------|
| value / map | Always included, always evaluated |
| `fn -> ... end` or `&Mod.fun/0` | Included on first visit; lazy on partial reload |
| `inertia_optional(fun)` | Only when requested via partial reload |
| `inertia_defer(fun)` | After first paint (async client fetch) |
| `inertia_defer(fun, "group")` | Deferred in named parallel group |
| `inertia_defer(fun, on_error: :ignore)` | Omit failed deferred prop; path in `rescuedProps` |
| `inertia_merge(value)` | Merge/append on client (pagination) |
| `inertia_merge(value, match_on: "id")` | Merge with client-side dedupe |
| `inertia_prepend(value)` | Prepend (not append) on merge |
| `inertia_defer(fun) |> inertia_merge()` | Deferred then merged |
| `inertia_deep_merge(value)` | Deep merge nested objects |
| `inertia_once(fun)` | Client-cached across pages (`fresh`, `until`, `as`) |
| `inertia_scroll(page)` | Infinite scroll metadata + merge |
| `inertia_always(value)` | Included even on partial reloads that omit it |
| `inertia_share(value)` | Mark inline prop as shared (`sharedProps` metadata) |
| `assign_shared_prop(conn, key, value)` | Shared prop helper for plugs |
| `preserve_case(:key)` | Wrap the **key** in `assign_prop` to skip camelization |

```elixir
|> assign_prop(preserve_case(:snake_stays), "value")
|> assign_prop(:csrf_token, inertia_always(get_csrf_token()))
|> assign_prop(:stats, &Accounts.stats/0)
|> assign_shared_prop(:current_user, serialize_user(user))
```

Wrappers (`inertia_defer`, `inertia_merge`, `inertia_optional`, `inertia_once`,
`inertia_scroll`, etc.) work at **any nesting depth**, including inside closures.

Protocols (when built-ins are not enough):

- `Inertia.Errors` — custom error structs → flat error map for `assign_errors`
- `Inertia.Paginated` — pagination library structs → scroll meta (+ optional `:entries`) for `inertia_scroll`

## References (read when needed)

- Setup / Igniter / TS client / asset versioning / CSRF → [references/setup.md](references/setup.md)
- Controllers, props, Errors/Paginated protocols, flash, redirects, history → [references/controllers.md](references/controllers.md)
- TSX pages, layouts, navigation, deferred UI, infinite scroll → [references/react-pages.md](references/react-pages.md)
- Forms and validation UX → [references/react-forms.md](references/react-forms.md)
- `Inertia.Testing` → [references/testing.md](references/testing.md)
- SSR (dev + production Node/`NODE_ENV`) → [references/ssr.md](references/ssr.md)

## Docs

- HexDocs: https://hexdocs.pm/inertia
- Upgrade v2 → v3: https://hexdocs.pm/inertia/upgrading_to_v3.html
- Repo: https://github.com/inertiajs/inertia-phoenix
- Client: https://inertiajs.com
- `Inertia.Errors`: https://hexdocs.pm/inertia/Inertia.Errors.html
- `Inertia.Paginated`: https://hexdocs.pm/inertia/Inertia.Paginated.html
