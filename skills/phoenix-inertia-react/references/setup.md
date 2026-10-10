# Setup — Phoenix + Inertia + React + TypeScript

Assumes a Phoenix app using the official Hex `inertia` package
([inertiajs/inertia-phoenix](https://github.com/inertiajs/inertia-phoenix)),
React, and TypeScript. Targets **inertia Hex ~> 3.0** and **`@inertiajs/react` v3**.

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
{:inertia, "~> 3.0"}
# Most Phoenix apps already have ecto — required only for assign_errors(changeset).
# SSR with the default adapter also needs:
# {:nodejs, ">= 3.1.4 and < 4.0.0"}
```

```elixir
# config/config.exs
config :inertia,
  # Used to build asset URLs for the version hash (triggers full reload when assets change)
  endpoint: MyAppWeb.Endpoint,

  # Static file paths to track for changes (include JS that needs a refresh when modified)
  # Installer / Phoenix 1.8 esbuild default output:
  static_paths: ["/assets/js/app.js"],

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

  # Optional: CSP nonce assign key applied to the page-data <script> tag
  # csp_nonce_assign_key: :csp_nonce

  # Optional: disable SSR for path prefixes / regexes
  # ssr_exclude_paths: ["/admin"]
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

With esbuild code-splitting (installer enables this), load as ESM from the
`assets/js` output dir:

```heex
<script type="module" defer phx-track-static src={~p"/assets/js/app.js"}></script>
```

## Client (React + TypeScript)

```sh
cd assets
npm install @inertiajs/react react react-dom
npm install -D typescript @types/react @types/react-dom
```

Inertia v3 ships a built-in HTTP client — **do not add axios** just for Inertia.

```tsx
// assets/js/app.tsx
import React from "react";
import { createInertiaApp } from "@inertiajs/react";
import { createRoot } from "react-dom/client";

createInertiaApp({
  resolve: (name) => {
    const page = import(`./pages/${name}.tsx`);
    return page;
  },
  setup({ App, el, props }) {
    createRoot(el).render(<App {...props} />);
  },
  http: {
    // Adapter sets XSRF-TOKEN cookie; Phoenix expects x-csrf-token header
    xsrfHeaderName: "x-csrf-token",
  },
});
```

Pages live under `assets/js/pages/` as **`.tsx`** default exports.
`render_inertia("Users/Index")` → `pages/Users/Index.tsx`.

### esbuild (default)

- Entry: `js/app.tsx`
- esbuild `>= 0.19` for glob/dynamic imports (required for page resolution)
- Prefer Phoenix's current default (e.g. `0.25+`) with `--target=es2022`
- After bumping esbuild version: `mix esbuild.install`
- Add `cmd --cd assets npm install` to the `assets.setup` Mix alias

```elixir
config :esbuild,
  version: "0.25.4",
  my_app: [
    args:
      ~w(js/app.tsx --bundle --chunk-names=chunks/[name]-[hash] --splitting --format=esm --target=es2022 --outdir=../priv/static/assets/js --external:/fonts/* --external:/images/*),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]
```

### Code splitting (recommended)

Installer enables `--format=esm` + `--splitting`. Root layout script must use
`type="module"` (see above). ESM splitting needs modern browsers — verify against
your audience.

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
entry that should invalidate the page when modified. Keep paths aligned with the
real compiled asset path (e.g. `/assets/js/app.js`). If you skip tracking, set
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

Flash is a **top-level** page key in v3 (not under `props`):

```tsx
// assets/js/types/page.ts
export type Flash = {
  info?: string;
  error?: string;
};

export type SharedProps = {
  currentUser: { id: number; email: string } | null;
};
```

```tsx
import { usePage } from "@inertiajs/react";
import type { Flash, SharedProps } from "../types/page";

const { currentUser } = usePage<SharedProps>().props;
const { flash } = usePage<{ flash: Flash }>();
// or: const flash = usePage().flash as Flash
```

Path aliases (`@/`) only if `tsconfig` / bundler already define them — do not invent
aliases the build does not resolve.

## CSRF

The adapter sets the `XSRF-TOKEN` cookie. Configure the header name on
`createInertiaApp` (built-in HTTP client — not axios):

```ts
createInertiaApp({
  http: {
    xsrfHeaderName: "x-csrf-token",
  },
  // ...
});
```

Phoenix expects that header name. `<Form>` / `useForm` use the Inertia client.
