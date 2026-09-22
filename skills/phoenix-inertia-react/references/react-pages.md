# React + TypeScript Pages

Pages are `.tsx` default exports. Props come from the controller — type them
explicitly. With `camelize_props: true`, use camelCase in types.

## Page components

```tsx
type Props = {
  posts: { id: number; title: string }[];
  currentUser: { id: number; email: string } | null;
};

export default function Index({ posts }: Props) {
  return (
    <ul>
      {posts.map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}
```

Resolve path: `render_inertia("Posts/Index")` → `assets/js/pages/Posts/Index.tsx`.

Prefer `type Props = { … }` over `interface` for page props.

## Persistent layouts

```tsx
import type { ReactNode } from "react";
import AppLayout from "../layouts/AppLayout";

type Props = { post: { id: number; title: string } };

export default function Show({ post }: Props) {
  return <article>{post.title}</article>;
}

Show.layout = (page: ReactNode) => <AppLayout>{page}</AppLayout>;
```

Or set a default layout in `createInertiaApp`'s resolve callback. Do **not** only
wrap JSX in the page return if you need preserved layout state.

## Navigation

```tsx
import { Head, Link, router } from "@inertiajs/react";

<Head title="Posts" />

<Link href="/posts">Posts</Link>
<Link href="/posts" prefetch>
  Prefetch
</Link>

router.visit("/posts");
router.get("/posts", { q: "elixir" }, { preserveState: true });
router.reload({ only: ["posts"] });
```

**NEVER** use React Router for Inertia pages.
**NEVER** use bare `<a href>` for in-app Inertia navigation.

`~p` verified routes are Elixir-only — pass path strings from the server as props
when you need a single source of truth, or keep a small typed routes helper on the
client. Do not invent `@/routes` unless the project already has one.

## Shared / flash via usePage

```tsx
import { usePage } from "@inertiajs/react";
import type { SharedProps } from "../types/page";

export function Flash() {
  const { flash } = usePage<SharedProps>().props;
  return flash.info ? <div role="status">{flash.info}</div> : null;
}
```

Flash is under **`props.flash`** (auto from Phoenix). Auth is whatever you
`assign_prop` in plugs (`currentUser` when camelized).

## Deferred UI

Server: `assign_prop(:stats, inertia_defer(fn -> … end))`.

```tsx
import { Deferred, usePage } from "@inertiajs/react";

type Props = { stats?: { count: number } };

function StatsPanel() {
  const { stats } = usePage<Props>().props;
  return <p>{stats?.count}</p>;
}

export default function Dashboard(_props: Props) {
  return (
    <Deferred data="stats" fallback={<p>Loading…</p>}>
      <StatsPanel />
    </Deferred>
  );
}
```

`<Deferred>` children do **not** receive data as a render-prop argument.

## Infinite scroll

Server:

```elixir
conn
|> assign_prop(:users, inertia_scroll(fn -> Accounts.paginate_users(params) end))
|> render_inertia("Users/Index")
```

Client (Inertia v2):

```tsx
import { InfiniteScroll } from "@inertiajs/react";

type User = { id: number; name: string };
type Props = {
  users: {
    data: User[];
    meta: { currentPage?: number; nextPage?: number | null };
  };
};

export default function Index({ users }: Props) {
  return (
    <InfiniteScroll data="users">
      {users.data.map((u) => (
        <div key={u.id}>{u.name}</div>
      ))}
    </InfiniteScroll>
  );
}
```

Keep query param names aligned with `page_name` / scroll metadata. Adjust prop
shape if `camelize_props` renames nested keys.

## URL-driven UI

Dialogs, tabs, filters: controller reads `params`, passes props; update with
`router.get` / `Link`. Do not sync URL ↔ React with `useEffect` + `window.location`.

## Partial reloads

```tsx
router.reload({ only: ["comments"] });
router.reload({ except: ["heavyChart"] });
```

Request optional props by including their keys in `only` (camelCase keys if
camelize is on).

## Anti-patterns

- `useEffect(() => { fetch(...) }, [])` for initial page data
- React Query / SWR mirroring Inertia props
- Parsing `window.location.search` instead of controller props
- Checking `currentUser` in React as the only auth gate
- Snake_case TS props when `camelize_props: true` (or the reverse)
