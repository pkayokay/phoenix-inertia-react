# Testing — Inertia Phoenix

Import helpers in `ConnCase` (recommended so every controller test has them):

```elixir
defmodule MyAppWeb.ConnCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      import Plug.Conn
      import Phoenix.ConnTest
      import Inertia.Testing

      @endpoint MyAppWeb.Endpoint
    end
  end
end
```

| Helper | Description |
|--------|-------------|
| `inertia_component/1` | Component name |
| `inertia_props/1` | Props map |
| `inertia_errors/1` | Validation errors (props or session) |
| `inertia_flash/1` | Flash map (top-level page key) |
| `inertia_page/1` | Full page object |
| `inertia_shared_props/1` | Shared prop keys |
| `inertia_deferred_props/1` | Deferred prop groups |
| `inertia_merge_props/1` | Merge prop paths |
| `inertia_scroll_props/1` | Scroll pagination metadata |
| `inertia_once_props/1` | Once prop metadata |
| `inertia_response?/1` | Whether the response is Inertia |

## Assert component + props

Prop keys in `inertia_props/1` match what the **server** assigned (atoms / snake_case
in Elixir tests), even when `camelize_props` renames keys for the JSON client.

```elixir
test "renders users index", %{conn: conn} do
  conn = get(conn, ~p"/users")

  assert inertia_component(conn) == "Users/Index"
  assert %{users: users} = inertia_props(conn)
  assert length(users) == 2
  assert inertia_flash(conn) == %{}
end
```

```elixir
test "renders home", %{conn: conn} do
  conn = get(conn, ~p"/")
  assert inertia_component(conn) == "Home"
  assert %{user: %{id: 1}} = inertia_props(conn)
end
```

## Assert errors after failed create

`inertia_errors/1` reads errors from the current props **or** the session after
redirect — you usually assert on the redirect response without `follow_redirect`.

```elixir
test "rejects blank name", %{conn: conn} do
  conn = post(conn, ~p"/users", %{"name" => ""})

  assert redirected_to(conn) == ~p"/users/new"
  assert %{user: %{id: 1}} = inertia_props(conn) # shared props still present
  assert inertia_errors(conn) == %{"name" => "can't be blank"}
end
```

## Tips

- Follow redirects when you need the **final page props** after PRG
- Shared props from plugs appear in `inertia_props/1`; keys also via `inertia_shared_props/1`
- Flash is **not** inside props — use `inertia_flash/1`
- Deferred props may be absent or marked deferred depending on request headers —
  test the initial visit and partial-reload cases separately; use `inertia_deferred_props/1`
- Scroll / merge / once metadata: `inertia_scroll_props/1`, `inertia_merge_props/1`,
  `inertia_once_props/1`
- Assert serialized maps, not raw Ecto structs
- Use `inertia_response?/1` when an action might return a non-Inertia response
- Error keys are flat strings (`"name"`, `"team.name"`) — same shape as the client
- Keep `config :inertia, ssr: false` in test so the Node SSR pool is not required
