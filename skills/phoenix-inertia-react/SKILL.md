---
name: phoenix-inertia-react
description: >-
  Phoenix + Inertia.js + React + TypeScript via the official `inertia` Hex package
  (inertiajs/inertia-phoenix). Assumes that stack. Use when installing Inertia,
  writing assign_prop/render_inertia controllers, TSX pages/forms, shared props,
  deferred/merge/once/scroll props, validation errors, flash, CSRF,
  Inertia.Testing, or SSR. NEVER useEffect+fetch for page data. NEVER
  react-hook-form. External URLs use Phoenix redirect (auto-converted);
  same-origin non-Inertia routes need force_inertia_redirect.
---

# Phoenix + Inertia + React + TypeScript

Assumed stack: **Phoenix** + Hex **`inertia`** (`inertiajs/inertia-phoenix`) +
**React** + **TypeScript** (`@inertiajs/react`, pages as `.tsx`).

Phoenix owns routing, data, and auth. React renders typed props — not a
traditional SPA.

**Package name:** Hex dep is `{:inertia, "~> 2.6"}`. Do not confuse with the
older community `inertia_phoenix` package.

**Target API:** stable v2.x. Only use v3 `assign_shared_prop` / `inertia_share`
if `mix.exs` pins `3.x`.

**Props casing:** Prefer `camelize_props: true` so Elixir `snake_case` becomes
JS `camelCase` matching TypeScript types. If disabled, TS prop names stay snake_case.

## Core Mental Model

1. Controller builds props → `render_inertia("Page/Name")`
2. TSX page receives those props (typed) and renders
3. Navigation and forms go through Inertia (`Link`, `Form`, `router`) — never a
   parallel client data layer

**Before building a feature, ask:**
- **Where does data come from?** Server → `assign_prop`. User UI chrome → `useState`.
- **Needed on every page?** Shared plug with `assign_prop`, not per-action duplication.
- **Expensive?** `inertia_defer` / `inertia_optional` / bare `fn -> ... end` lazy eval.
- **Reaching for SPA patterns?** Check the decision matrix first.

## Decision Matrix

| Need | Solution | NOT This |
|------|----------|----------|
| Page data | `assign_prop` + `render_inertia` | `useEffect` + `fetch` |
| Global data (auth, flash) | Shared plug `assign_prop` / auto `flash` | React Context / Redux |
| Forms | `<Form>` / `useForm` | `fetch` / axios / react-hook-form |
| Navigate | `<Link>` / `router.visit` | react-router / `<a>` full reload |
| Refresh subset | `router.reload({ only: [...] })` | React Query / SWR |
| Expensive props | `inertia_defer` | Client loading + fetch |
| Infinite scroll | `inertia_scroll` + `<InfiniteScroll>` | Client-only pagination |
| Stable reference data | `inertia_once` | Cache in React state |
| External / non-Inertia URL | `redirect(to: url)` or `force_inertia_redirect` | Broken 302 JSON parse |
| Ephemeral UI | `useState` | Server props |

## Critical Rules

| # | Rule | Why |
|---|------|-----|
| 1 | Never `useEffect`+`fetch` for page data | Second data lifecycle drifts from Inertia props |
| 2 | Never check auth only in React | Client guards are spoofable; plug/controller gate |
| 3 | Use `<Form>` / `useForm`, not react-hook-form | CSRF, redirects, error map, files already handled |
| 4 | Use `<Link>` / `router`, not bare `<a>` for app routes | Full reload destroys layout state |
| 5 | Always `assign_prop` (or inline props map) — assigns are NOT auto-props | `conn.assigns` alone never reach React |
| 6 | Validation: `assign_errors(changeset)` then `redirect` | Errors preserved across redirect; flat keys for forms |
| 7 | CSRF: `axios.defaults.xsrfHeaderName = "x-csrf-token"` | Phoenix expects that header; cookie is set by adapter |
| 8 | External redirects: Phoenix `redirect` is auto-converted; same-origin non-Inertia needs `force_inertia_redirect` | Else Inertia client mis-handles the response |
| 9 | Type page props explicitly; keep TS names in sync with `camelize_props` | Mismatched casing = silent `undefined` at runtime |

## Quick Patterns

### Controller

```elixir
def index(conn, _params) do
  conn
  |> assign_prop(:users, Accounts.list_users())
  |> assign_prop(:stats, inertia_defer(fn -> Accounts.stats() end))
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
| `fn -> ... end` | Included on first visit; lazy on partial reload |
| `inertia_optional(fn -> ... end)` | Only when requested via partial reload |
| `inertia_defer(fn -> ... end)` | After first paint (async client fetch) |
| `inertia_defer(fn, "group")` | Deferred in named parallel group |
| `inertia_merge(value)` | Merge/append on client (pagination) |
| `inertia_deep_merge(value)` | Deep merge nested objects |
| `inertia_once(fn -> ... end)` | Client-cached across pages |
| `inertia_scroll(page)` | Infinite scroll metadata + merge |
| `inertia_always(value)` | Included even on partial reloads that omit it |
| `preserve_case(:key)` | Wrap the **key** in `assign_prop` to skip camelization |

```elixir
|> assign_prop(preserve_case(:snake_stays), "value")
|> assign_prop(:csrf_token, inertia_always(get_csrf_token()))
```

## References (read when needed)

- Setup / Igniter / TS client boot → [references/setup.md](references/setup.md)
- Controllers, props, errors, flash, redirects → [references/controllers.md](references/controllers.md)
- TSX pages, layouts, navigation, deferred UI → [references/react-pages.md](references/react-pages.md)
- Forms and validation UX → [references/react-forms.md](references/react-forms.md)
- `Inertia.Testing` → [references/testing.md](references/testing.md)
- SSR → [references/ssr.md](references/ssr.md)

## Docs

- HexDocs: https://hexdocs.pm/inertia
- Repo: https://github.com/inertiajs/inertia-phoenix
- Client: https://inertiajs.com
