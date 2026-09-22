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

Helpers: `inertia_component/1`, `inertia_props/1`, `inertia_errors/1`,
`inertia_response?/1`.

## Assert component + props

Prop keys in `inertia_props/1` match what the **server** assigned (atoms / snake_case
in Elixir tests), even when `camelize_props` renames keys for the JSON client.

```elixir
test "renders users index", %{conn: conn} do
  conn = get(conn, ~p"/users")

  assert inertia_component(conn) == "Users/Index"
  assert %{users: users} = inertia_props(conn)
  assert length(users) == 2
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
- Shared props from plugs appear in `inertia_props/1`
- Deferred props may be absent or marked deferred depending on request headers —
  test the initial visit and partial-reload cases separately
- Assert serialized maps, not raw Ecto structs
- Use `inertia_response?/1` when an action might return a non-Inertia response
- Error keys are flat strings (`"name"`, `"team.name"`) — same shape as the client
