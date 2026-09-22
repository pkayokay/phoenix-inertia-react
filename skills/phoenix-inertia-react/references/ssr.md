# SSR — Inertia Phoenix (optional)

Skip unless the project enables `ssr: true`. Full detail:
https://hexdocs.pm/inertia/readme.html#server-side-rendering

Assumes React + TypeScript client.

## Essentials

1. Add `assets/js/ssr.tsx` exporting `render(page)` via `createInertiaApp` +
   `ReactDOMServer.renderToString` (no Node HTTP server — the Hex package pools
   Node workers).
2. esbuild target for SSR: `--platform=node --format=cjs` → `priv/ssr.js`
3. Supervise `{Inertia.SSR, path: Path.join([Application.app_dir(:my_app), "priv"])}`
4. `config :inertia, ssr: true` and `raise_on_ssr_failure: config_env() != :prod`
5. Client `app.tsx`: `hydrateRoot` instead of `createRoot`
6. Production image needs Node.js; set `NODE_ENV=production`
7. Gitignore generated `/priv/ssr.js`

```tsx
// assets/js/ssr.tsx
import React from "react";
import ReactDOMServer from "react-dom/server";
import { createInertiaApp } from "@inertiajs/react";

export function render(page: unknown) {
  return createInertiaApp({
    page,
    render: ReactDOMServer.renderToString,
    resolve: (name) => import(`./pages/${name}.tsx`),
    setup: ({ App, props }) => <App {...props} />,
  });
}
```

Per-response override: `render_inertia(conn, "Page", props, ssr: true)`.
