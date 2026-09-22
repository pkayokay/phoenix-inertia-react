# SSR — Inertia Phoenix

Full detail: https://hexdocs.pm/inertia/readme.html#server-side-rendering  
Source: https://github.com/inertiajs/inertia-phoenix

Skip unless the project enables `ssr: true`. Assumes React + TypeScript.

Unlike some backends, this adapter does **not** run a separate Node HTTP server.
It pools Node.js worker processes inside the Elixir supervision tree and calls
an exported `render` function.

## 1. Server entry (`ssr.tsx`)

Alongside `app.tsx`, export `render` — do not start a Node server:

```tsx
// assets/js/ssr.tsx
import React from "react";
import ReactDOMServer from "react-dom/server";
import { createInertiaApp } from "@inertiajs/react";

export function render(page: unknown) {
  return createInertiaApp({
    page,
    render: ReactDOMServer.renderToString,
    resolve: async (name) => {
      return await import(`./pages/${name}.tsx`);
    },
    setup: ({ App, props }) => <App {...props} />,
  });
}
```

## 2. esbuild SSR profile

```elixir
# config/config.exs
config :esbuild,
  version: "0.21.5",
  my_app: [
    args: ~w(js/app.tsx --bundle --target=es2020 --outdir=../priv/static/assets --external:/fonts/* --external:/images/*),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ],
  ssr: [
    args: ~w(js/ssr.tsx --bundle --platform=node --outdir=../priv --format=cjs),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]
```

Dev watchers (`config/dev.exs`):

```elixir
watchers: [
  esbuild: {Esbuild, :install_and_run, [:my_app, ~w(--sourcemap=inline --watch)]},
  ssr: {Esbuild, :install_and_run, [:ssr, ~w(--sourcemap=inline --watch)]},
  # ...
]
```

Mix aliases:

```elixir
"assets.build": ["tailwind my_app", "esbuild my_app", "esbuild ssr"],
"assets.deploy": [
  "tailwind my_app --minify",
  "esbuild my_app --minify",
  "esbuild ssr",
  "phx.digest"
]
```

Gitignore generated bundle:

```gitignore
/priv/ssr.js
```

## 3. Supervise `Inertia.SSR`

```elixir
# lib/my_app/application.ex
children = [
  # ...
  # path = directory containing ssr.js
  {Inertia.SSR, path: Path.join([Application.app_dir(:my_app), "priv"])},
  MyAppWeb.Endpoint
]
```

## 4. Enable SSR in config

```elixir
config :inertia,
  endpoint: MyAppWeb.Endpoint,
  static_paths: ["/assets/app.js"],
  default_version: "1",
  ssr: true,
  # Recommended: raise in non-prod; in prod fall back to CSR instead of 500
  raise_on_ssr_failure: config_env() != :prod
```

Per-response override: `render_inertia(conn, "Page", props, ssr: true)`.

## 5. Client hydration

```tsx
// assets/js/app.tsx
import { hydrateRoot } from "react-dom/client";
import { createInertiaApp } from "@inertiajs/react";

createInertiaApp({
  resolve: (name) => import(`./pages/${name}.tsx`),
  setup({ App, el, props }) {
    hydrateRoot(el, <App {...props} />);
  },
});
```

Use `hydrateRoot` instead of `createRoot` when SSR is on.

## 6. Production: Node.js in the runner image

SSR needs Node on the production host. Example Dockerfile additions after
`FROM ${RUNNER_IMAGE}`:

```dockerfile
RUN apt-get update -y && \
    apt-get install -y libstdc++6 openssl curl libncurses5 locales ca-certificates && \
    apt-get clean && rm -f /var/lib/apt/lists/*_*

# install Node.js (pin a current LTS in real apps)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get update && \
    apt-get install -y nodejs

ENV MIX_ENV="prod"
# CRITICAL: cache the SSR script in memory — without this, renders are very slow
ENV NODE_ENV="production"
```

**Always set `NODE_ENV=production` in production** so the SSR script stays cached.

## Checklist

- [ ] `ssr.tsx` exports `render`
- [ ] esbuild `ssr` profile → `priv/ssr.js`
- [ ] Dev watcher + `assets.build` / `assets.deploy` include `esbuild ssr`
- [ ] `/priv/ssr.js` gitignored
- [ ] `{Inertia.SSR, path: ...}` in supervision tree
- [ ] `ssr: true` (+ sensible `raise_on_ssr_failure`)
- [ ] Client uses `hydrateRoot`
- [ ] Production image has Node + `NODE_ENV=production`
