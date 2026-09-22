# Controllers — Inertia Phoenix

Import `Inertia.Controller` via `MyAppWeb, :controller`.

Stack assumption: React + TypeScript client; prefer `camelize_props: true`.

Official adapter: https://github.com/inertiajs/inertia-phoenix

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

### Flash messages

Phoenix flash is included automatically under `props.flash`. Do **not** re-share
flash with `assign_prop`.

```elixir
conn
|> put_flash(:info, "Settings updated")
|> redirect(to: ~p"/settings")
```

Client receives:

```ts
props.flash // e.g. { info: "Settings updated" }
```

Read via `usePage().props.flash` (type it on your shared props).

## Prop types

```elixir
conn
|> assign_prop(:cheap, cheap())
|> assign_prop(:lazy, fn -> expensive() end)
|> assign_prop(:named_lazy, &Accounts.calculate_stats/0) # named function reference
|> assign_prop(:optional, inertia_optional(fn -> very_expensive() end))
|> assign_prop(:deferred, inertia_defer(fn -> slow() end))
|> assign_prop(:grouped, inertia_defer(fn -> chart() end, "analytics"))
|> assign_prop(:items, inertia_merge(list))
|> assign_prop(:merged_defer, inertia_defer(&calculate_next_page/0) |> inertia_merge())
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
| `fn -> ... end` or `&fun/0` | yes | if requested | when needed |
| `inertia_optional` | no | only if requested | when needed |
| `inertia_defer` | stub, then async fetch | — | after first paint |
| `inertia_always` | yes | **always**, even if omitted from `only` | always |

Anonymous functions **and** named function references (`&Mod.fun/0`) are both
valid for lazy / optional / defer / once / scroll.

### Once props options

```elixir
inertia_once(fn -> Plans.list() end, fresh: true)
inertia_once(fn -> Rates.current() end, until: 3600)
inertia_once(fn -> Rates.current() end,
  until: DateTime.utc_now() |> DateTime.add(1, :day)
)
inertia_once(fn -> Roles.all() end, as: "roles")
```

- `fresh: true | boolean` — force refresh (ignore client cache)
- `until: integer | DateTime` — expire after N seconds or at a time
- `as: "key"` — share the same cache across different prop names

Compose with other helpers:

```elixir
# Deferred + once: load after paint, then cache
|> assign_prop(:permissions, inertia_once(inertia_defer(fn -> Permissions.for_user(user) end)))

# Merge + once: merge with existing data and cache
|> assign_prop(:activity, inertia_once(inertia_merge(fn -> Activity.recent(user) end)))
```

### Merge / deep merge

```elixir
|> assign_prop(:paginated_list, inertia_merge(["a", "b", "c"]))
|> assign_prop(:paginated_list, inertia_defer(&calculate_next_page/0) |> inertia_merge())
|> assign_prop(:complex_object, inertia_deep_merge(%{a: %{b: %{c: %{d: 1}}}}))
```

### Scroll props

Expected shape (or a struct implementing `Inertia.ScrollMetadata`):

```elixir
%{
  data: [%{id: 1, name: "Alice"}],
  meta: %{
    current_page: 1,
    next_page: 2,
    previous_page: nil,
    page_name: "page" # optional, defaults to "page"
  }
}
```

This produces `mergeProps` for the data path (e.g. `"users.data"`) plus
`scrollProps` metadata for the client `<InfiniteScroll>` component.

```elixir
# Basic
|> assign_prop(:users, inertia_scroll(paginated_users))

# Lazy
|> assign_prop(:users, inertia_scroll(fn -> User.paginate(params) end))

# Custom wrapper + page name (multiple scroll containers on one page)
|> assign_prop(:users, inertia_scroll(users, wrapper: "items", page_name: "users_page"))
|> assign_prop(:orders, inertia_scroll(orders, page_name: "orders_page"))

# Custom metadata extraction (e.g. Scrivener without a protocol impl)
|> assign_prop(:users, inertia_scroll(scrivener_page,
  wrapper: "entries",
  metadata: fn page ->
    %{
      page_name: "page",
      current_page: page.page_number,
      previous_page: if(page.page_number > 1, do: page.page_number - 1),
      next_page: if(page.page_number < page.total_pages, do: page.page_number + 1)
    }
  end
))
```

#### `Inertia.ScrollMetadata` protocol

For reusable pagination library support:

```elixir
defimpl Inertia.ScrollMetadata, for: Scrivener.Page do
  def to_scroll_metadata(page) do
    %{
      page_name: "page",
      current_page: page.page_number,
      previous_page: if(page.page_number > 1, do: page.page_number - 1),
      next_page: if(page.page_number < page.total_pages, do: page.page_number + 1)
    }
  end
end
```

Then:

```elixir
|> assign_prop(:users, inertia_scroll(scrivener_page, wrapper: "entries"))
```

Required metadata keys: `:page_name`, `:current_page`, `:previous_page`, `:next_page`.

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
- Nested errors flatten: `"team.name"`, `"items[1].price"`
- Top-level schema fields are `"name"`, **not** `"user.name"` (param nesting ≠ error keys)
- `errors` prop is always present (empty object when none)
- Errors survive the redirect (readable via `inertia_errors/1` on the redirect response)

### Custom message function (Gettext)

```elixir
conn
|> assign_errors(changeset, fn {msg, opts} ->
  if count = opts[:count] do
    Gettext.dngettext(MyAppWeb.Gettext, "errors", msg, msg, count, opts)
  else
    Gettext.dgettext(MyAppWeb.Gettext, "errors", msg, opts)
  end
end)
```

Default message func does simple `%{key}` string replacement (e.g. `"should be at least %{count} characters"` → `"should be at least 3 characters"`).

### Bare error maps

Must be a flat map of atom/string keys → **string** values:

```elixir
conn
|> assign_errors(%{
  name: "Name can't be blank",
  password: "Password must be at least 5 characters"
})
```

### `Inertia.Errors` protocol

Built-in for `Ecto.Changeset` and properly shaped maps. For custom error types:

```elixir
defimpl Inertia.Errors, for: MyApp.ValidationError do
  def to_errors(%MyApp.ValidationError{} = err) do
    %{
      "field_name" => err.message,
      "nested.field" => "Another error message"
    }
  end

  def to_errors(err, _msg_func), do: to_errors(err)
end
```

Then `assign_errors(conn, my_validation_error)` works. See
https://hexdocs.pm/inertia/Inertia.Errors.html.

## Redirects

- Internal Inertia pages: normal `redirect(to: ~p"/...")`
- External URLs: `redirect(to: "https://...")` — adapter converts to **409** + `X-Inertia-Location`
- Same-origin **non-Inertia** route: `conn |> force_inertia_redirect() |> redirect(to: "/legacy")`
- PUT/PATCH/DELETE 302 → automatically upgraded to **303**

## History

```elixir
conn |> encrypt_history()   # sensitive props in history state
conn |> clear_history()     # e.g. on logout
```

Or globally: `config :inertia, history: [encrypt: true]`.

## Asset versioning

The adapter hashes tracked static assets (via `endpoint` + `static_paths`) to
compute a version string. When assets change, the client does a full page
reload to pick up the new bundle.

```elixir
config :inertia,
  endpoint: MyAppWeb.Endpoint,
  static_paths: ["/assets/app.js"], # include JS that needs a refresh when changed
  default_version: "1"              # used when static_paths is empty / unused
```

Keep `static_paths` aligned with the real compiled asset path (esbuild/Vite
output under `priv/static`).

## Serialization tips

- Prefer explicit maps you control for TypeScript-friendly shapes
- Preload associations you need; avoid N+1 inside prop lambdas
- Keep payloads small; defer the rest
- Match camelCase field names in serializers when `camelize_props` is on (or rely on auto-camelize for map keys)
