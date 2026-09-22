# Controllers — Inertia Phoenix

Import `Inertia.Controller` via `MyAppWeb, :controller`.

Stack assumption: React + TypeScript client; prefer `camelize_props: true`.

## Render

```elixir
conn
|> assign_prop(:user, serialize(user))
|> render_inertia("Profile/Show")

# Inline props map (merged with assign_prop)
conn
|> render_inertia("Profile/Show", %{title: "Hello"})

# With options
conn
|> render_inertia("Profile/Show", %{title: "Hello"}, ssr: true)
```

**CRITICAL:** `conn.assigns` are NOT sent to React. Only props from
`assign_prop/2` or the inline map in `render_inertia/3` reach the page.

Serialize explicitly — do not dump full Ecto schemas into props.

## Shared data

Share in a plug used by the browser pipeline (auth plug is typical):

```elixir
defmodule MyAppWeb.UserAuth do
  import Inertia.Controller
  import Plug.Conn

  def fetch_current_user(conn, _opts) do
    user = get_session_user(conn)

    conn
    |> assign(:current_user, user) # server-side auth checks
    |> assign_prop(:current_user, serialize_user(user)) # React (→ currentUser if camelized)
  end
end
```

Flash is automatic under `props.flash` — do not re-share flash manually.

## Prop types

```elixir
conn
|> assign_prop(:cheap, cheap())
|> assign_prop(:lazy, fn -> expensive() end)
|> assign_prop(:optional, inertia_optional(fn -> very_expensive() end))
|> assign_prop(:deferred, inertia_defer(fn -> slow() end))
|> assign_prop(:grouped, inertia_defer(fn -> chart() end, "analytics"))
|> assign_prop(:items, inertia_merge(list))
|> assign_prop(:tree, inertia_deep_merge(map))
|> assign_prop(:roles, inertia_once(fn -> Roles.all() end))
|> assign_prop(:users, inertia_scroll(paginated))
|> assign_prop(:csrf_token, inertia_always(get_csrf_token()))
|> assign_prop(preserve_case(:raw_key), "unchanged")
|> render_inertia("Dashboard")
```

### Lazy vs optional vs defer

| Form | First visit | Partial reload | When evaluated |
|------|-------------|----------------|----------------|
| plain value | yes | if requested | always |
| `fn -> ... end` | yes | if requested | when needed |
| `inertia_optional` | no | only if requested | when needed |
| `inertia_defer` | stub, then async fetch | — | after first paint |
| `inertia_always` | yes | **always**, even if omitted from `only` | always |

### Once props options

```elixir
inertia_once(fn -> Plans.list() end, fresh: true)
inertia_once(fn -> Rates.current() end, until: 3600)
inertia_once(fn -> Roles.all() end, as: "roles")
```

Compose: `inertia_once(inertia_defer(fn -> … end))`.

### Scroll props shape

```elixir
%{
  data: [%{id: 1, name: "Alice"}],
  meta: %{
    current_page: 1,
    next_page: 2,
    previous_page: nil,
    page_name: "page"
  }
}
```

Options: `inertia_scroll(data, wrapper: "items", page_name: "users_page")`.
Implement `Inertia.ScrollMetadata` for pagination library structs.
Lazy: `inertia_scroll(fn -> Accounts.paginate_users(params) end)`.

## Validations (PRG)

```elixir
def update(conn, %{"id" => id} = params) do
  case Settings.update(id, params) do
    {:ok, _} ->
      conn
      |> put_flash(:info, "Saved")
      |> redirect(to: ~p"/settings")

    {:error, %Ecto.Changeset{} = changeset} ->
      conn
      |> assign_errors(changeset)
      |> redirect(to: ~p"/settings")
  end
end
```

- `assign_errors/2` accepts changeset or flat `%{field => "message"}` map
- Optional msg function: `assign_errors(conn, changeset, &MyAppWeb.ErrorHelpers.translate_error/1)`
- Nested errors flatten: `"team.name"`, `"items[1].price"`
- Top-level schema fields are `"name"`, **not** `"user.name"` (param nesting ≠ error keys)
- `errors` prop is always present (empty object when none)
- Errors survive the redirect (readable via `inertia_errors/1` on the redirect response)

## Redirects

- Internal Inertia pages: normal `redirect(to: ~p"/...")`
- External URLs: `redirect(to: "https://...")` — adapter converts to 409 + `X-Inertia-Location`
- Same-origin **non-Inertia** route: `conn |> force_inertia_redirect() |> redirect(to: "/legacy")`
- PUT/PATCH/DELETE 302 → automatically upgraded to 303

## History

```elixir
conn |> encrypt_history()   # sensitive props in history state
conn |> clear_history()     # e.g. on logout
```

Or globally: `config :inertia, history: [encrypt: true]`.

## Serialization tips

- Prefer explicit maps you control for TypeScript-friendly shapes
- Preload associations you need; avoid N+1 inside prop lambdas
- Keep payloads small; defer the rest
- Match camelCase field names in serializers when `camelize_props` is on (or rely on auto-camelize for map keys)
