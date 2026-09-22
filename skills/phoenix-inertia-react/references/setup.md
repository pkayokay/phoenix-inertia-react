# Setup — Phoenix + Inertia + React + TypeScript

Assumes a Phoenix app using the official Hex `inertia` package
([inertiajs/inertia-phoenix](https://github.com/inertiajs/inertia-phoenix)),
React, and TypeScript.

Default bundler is **esbuild** (what `mix inertia.install` configures). If the
project uses Vite instead, use Vite-style `import.meta.glob` — do not mix the two.

## Preferred install (Igniter)

```sh
mix archive.install hex igniter_new
mix igniter.install inertia --client-framework react --typescript --camelize-props
```

Official installer flags:

| Flag | Effect |
|------|--------|
| `--client-framework react\|vue\|svelte` | Configures client packages |
| `--camelize-props` | Sets `camelize_props: true` in config |
| `--history-encrypt` | Sets `history: [encrypt: true]` |
| `--typescript` | TS config + dev dependencies |
| `--yes` | Skip prompts |

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
  # Used to build asset URLs for the version hash (triggers full reload when assets change)
  endpoint: MyAppWeb.Endpoint,

  # Static file paths to track for changes (include JS that needs a refresh when modified)
  static_paths: ["/assets/app.js"],

  # Fallback version string when not tracking static assets (default: "1")
  default_version: "1",

  # snake_case → camelCase prop keys for JS/TS (recommended for this stack)
  camelize_props: true,

  # Encrypt page object in window history (also overridable per-request)
  history: [encrypt: false],

  # Server-side rendering (requires extra setup — see references/ssr.md)
  ssr: false,

  # Raise on SSR failure (enable in non-prod; disable in prod so SSR falls back to CSR)
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

Replace LiveView title helpers; add inertia head:

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

// Adapter sets XSRF-TOKEN cookie; Phoenix expects x-csrf-token header
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
- esbuild `>= 0.19` for glob/dynamic imports (required for page resolution)
- `--target=es2020` minimum
- After bumping esbuild version: `mix esbuild.install`

```elixir
config :esbuild,
  version: "0.21.5",
  my_app: [
    args:
      ~w(js/app.tsx --bundle --target=es2020 --outdir=../priv/static/assets --external:/fonts/* --external:/images/*),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]
```

### Code splitting (recommended for larger apps)

- `--format=esm`
- `--splitting`
- optional `--chunk-names=chunks/[name]-[hash]`
- Root layout script `type="module"` (see above)

```elixir
args:
  ~w(js/app.tsx --bundle --chunk-names=chunks/[name]-[hash] --splitting --format=esm --target=es2020 --outdir=../priv/static/assets --external:/fonts/* --external:/images/*),
```

ESM splitting needs modern browsers — verify against your audience.

### Vite (only if the project already uses Vite)

```tsx
resolve: (name) => {
  const pages = import.meta.glob("./pages/**/*.tsx", { eager: true });
  return pages[`./pages/${name}.tsx`];
},
```

Do **not** use `import.meta.glob` under esbuild.

## Asset versioning

`endpoint` + `static_paths` compute a version hash. When a tracked file changes,
Inertia forces a full reload so clients get the new bundle. Include every JS
entry that should invalidate the page when modified. If you skip tracking, set
`default_version` and bump it manually when needed.

## camelize_props (recommended for this stack)

With `camelize_props: true` (recommended):

| Elixir | TypeScript |
|--------|------------|
| `assign_prop(:current_user, …)` | `currentUser` |
| `assign_prop(:inserted_at, …)` | `insertedAt` |

Per-request:

```elixir
conn
|> assign_prop(:first_name, "Bob")
|> camelize_props()
|> render_inertia("Home")
```

Opt out per key:

```elixir
conn
|> assign_prop(preserve_case(:snake_stays), "value")
|> assign_prop(:will_camelize, "other")
|> camelize_props()
|> render_inertia("Home")
```

If the project leaves `camelize_props: false`, TypeScript prop names stay `snake_case`.
Detect from `config/config.exs` before writing types.

## Shared TypeScript types

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
import type { SharedProps } from "../types/page";

const { currentUser, flash } = usePage<SharedProps>().props;
```

Path aliases (`@/`) only if `tsconfig` / bundler already define them — do not invent
aliases the build does not resolve.

## CSRF

The adapter sets the `XSRF-TOKEN` cookie. Configure Axios once in `app.tsx`:

```ts
axios.defaults.xsrfHeaderName = "x-csrf-token";
```

Phoenix expects that header name. `<Form>` / `useForm` use the Inertia client
(axios under the hood).
