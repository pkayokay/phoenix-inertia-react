# Setup — Phoenix + Inertia + React + TypeScript

Assumes a Phoenix app using the official Hex `inertia` package, React, and TypeScript.
Default bundler is **esbuild** (what `mix inertia.install` configures). If the project
uses Vite instead, use Vite-style `import.meta.glob` — do not mix the two.

## Preferred install

```sh
mix archive.install hex igniter_new
mix igniter.install inertia --client-framework react --typescript --camelize-props
```

Useful flags: `--history-encrypt`, `--yes`.

Or after adding the dep manually:

```sh
mix inertia.install --client-framework react --typescript --camelize-props
```

## Manual pieces (if not using the installer)

```elixir
# mix.exs
{:inertia, "~> 2.6"}
```

```elixir
# config/config.exs
config :inertia,
  endpoint: MyAppWeb.Endpoint,
  static_paths: ["/assets/app.js"],
  default_version: "1",
  camelize_props: true,
  history: [encrypt: false],
  ssr: false,
  raise_on_ssr_failure: config_env() != :prod
```

### Wire helpers

```elixir
# lib/my_app_web.ex
def controller do
  quote do
    use Phoenix.Controller, formats: [:html, :json]
    import Inertia.Controller
    # ...
  end
end

def html do
  quote do
    use Phoenix.Component
    import Inertia.HTML
    # ...
  end
end
```

### Plug

```elixir
pipeline :browser do
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_live_flash
  plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
  plug :protect_from_forgery
  plug :put_secure_browser_headers
  plug Inertia.Plug
end
```

### Root layout

```heex
<.inertia_title>{@page_title}</.inertia_title>
<.inertia_head content={@inertia_head} />
```

With esbuild code-splitting (installer enables this), load as ESM:

```heex
<script type="module" defer phx-track-static src={~p"/assets/app.js"}></script>
```

## Client (React + TypeScript)

```sh
cd assets
npm install @inertiajs/react react react-dom axios
npm install -D typescript @types/react @types/react-dom
```

```tsx
// assets/js/app.tsx
import React from "react";
import axios from "axios";
import { createInertiaApp } from "@inertiajs/react";
import { createRoot } from "react-dom/client";

axios.defaults.xsrfHeaderName = "x-csrf-token";

createInertiaApp({
  resolve: (name) => {
    const page = import(`./pages/${name}.tsx`);
    return page;
  },
  setup({ App, el, props }) {
    createRoot(el).render(<App {...props} />);
  },
});
```

Pages live under `assets/js/pages/` as **`.tsx`** default exports.
`render_inertia("Users/Index")` → `pages/Users/Index.tsx`.

### esbuild (default)

- Entry: `js/app.tsx`
- esbuild `>= 0.19` for glob/dynamic imports
- `--target=es2020` minimum
- Installer typically adds `--splitting --format=esm`

```elixir
config :esbuild,
  version: "0.21.5",
  my_app: [
    args:
      ~w(js/app.tsx --bundle --chunk-names=chunks/[name]-[hash] --splitting --format=esm --target=es2020 --outdir=../priv/static/assets --external:/fonts/* --external:/images/*),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]
```

### Vite (only if the project already uses Vite)

```tsx
resolve: (name) => {
  const pages = import.meta.glob("./pages/**/*.tsx", { eager: true });
  return pages[`./pages/${name}.tsx`];
},
```

Do **not** use `import.meta.glob` under esbuild.

## camelize_props (recommended for this stack)

With `camelize_props: true` (recommended):

| Elixir | TypeScript |
|--------|------------|
| `assign_prop(:current_user, …)` | `currentUser` |
| `assign_prop(:inserted_at, …)` | `insertedAt` |

Opt out per key:

```elixir
conn
|> assign_prop(preserve_case(:snake_stays), "value")
|> assign_prop(:will_camelize, "other")
|> render_inertia("Home")
```

If the project leaves `camelize_props: false`, TypeScript prop names stay `snake_case`.
Detect from `config/config.exs` before writing types.

## Shared TypeScript types

Keep a small shared props type for `usePage`:

```tsx
// assets/js/types/page.ts
export type Flash = {
  info?: string;
  error?: string;
};

export type SharedProps = {
  currentUser: { id: number; email: string } | null;
  flash: Flash;
};
```

```tsx
import { usePage } from "@inertiajs/react";
import type { SharedProps } from "@/types/page";

const { currentUser, flash } = usePage<SharedProps>().props;
```

Path aliases (`@/`) only if `tsconfig` / bundler already define them — do not invent
aliases the build does not resolve.
