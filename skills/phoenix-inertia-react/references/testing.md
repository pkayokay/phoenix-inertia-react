# Testing — Inertia Phoenix

Import helpers in `ConnCase`:

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

## Assert errors after failed create

`inertia_errors/1` reads errors from the current props **or** the session after
redirect — you usually assert on the redirect response without `follow_redirect`.

```elixir
test "rejects blank name", %{conn: conn} do
  conn = post(conn, ~p"/users", user: %{name: ""})

  assert redirected_to(conn) == ~p"/users/new"
  assert inertia_errors(conn) == %{"name" => "can't be blank"}
end
```

## Tips

- Follow redirects when you need the **final page props** after PRG
- Shared props from plugs appear in `inertia_props/1`
- Deferred props may be absent or marked deferred depending on request headers —
  test the initial visit and partial-reload cases separately
- Assert serialized maps, not raw Ecto structs
- Use `inertia_response?/1` when a action might return a non-Inertia response
