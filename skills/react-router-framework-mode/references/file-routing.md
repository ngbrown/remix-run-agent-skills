---
title: File Routing
description: File-convention route configuration, nested routes, layouts, dynamic segments
tags: [routing, routes.ts, fs-routes, file-routing, nested-routes, layout, dynamic-segments, params]
---

# File Routing

For file conventions (`root.tsx`, `routes.ts`, etc.), see [special-files.md](./special-files.md).

If using manual route config helpers like `route`, `index`, and `layout`, see [routing.md](./routing.md).

## Route Configuration

File-convention routes are configured in `app/routes.ts` with `flatRoutes()` from `@react-router/fs-routes`:

```ts
import { type RouteConfig } from "@react-router/dev/routes";
import { flatRoutes } from "@react-router/fs-routes";

export default flatRoutes() satisfies RouteConfig;
```

### Complete Example

```ts
import { type RouteConfig } from "@react-router/dev/routes";
import { flatRoutes } from "@react-router/fs-routes";

export default flatRoutes() satisfies RouteConfig;
```

```text
app/
├── routes/
│   ├── _index.tsx
│   ├── about.tsx
│   ├── concerts.tsx
│   ├── concerts._index.tsx
│   ├── concerts.$city.tsx
│   └── concerts.trending.tsx
├── root.tsx
└── routes.ts
```

### Custom Route Directory

Use `rootDirectory` to scan a different folder relative to `app/`:

```ts
import { type RouteConfig } from "@react-router/dev/routes";
import { flatRoutes } from "@react-router/fs-routes";

export default flatRoutes({
  rootDirectory: "file-routes",
}) satisfies RouteConfig;
```

## Configuration Options

| API                                 | Purpose                            |
| ----------------------------------- | ---------------------------------- |
| `flatRoutes()`                      | Generate routes from `app/routes/` |
| `flatRoutes({ ignoredRouteFiles })` | Skip matching files                |
| `flatRoutes({ rootDirectory })`     | Read routes from a custom folder   |

## Nested Routes

Child routes are created by matching dot-delimited filenames:

```text
app/
├── routes/
│   ├── dashboard.tsx
│   ├── dashboard._index.tsx
│   └── dashboard.settings.tsx
└── root.tsx
```

Parent path is automatically included: creates `/dashboard` and `/dashboard/settings`.

### Outlet

Child routes render through `<Outlet />` in the parent:

```tsx
import { Outlet } from "react-router";

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Outlet />
    </div>
  );
}
```

## Root Route

**Every file route is nested inside `app/root.tsx`.** Put global navigation, footer, providers, and fonts there.

See [special-files.md](./special-files.md#roottsx-required) for `root.tsx` customization patterns.

## Layout Routes (Use Them!)

**Prefer nested routes over flat structures.** Layouts reduce code duplication and enable shared UI.

Create nesting without adding URL segments:

```text
app/
├── routes/
│   ├── _marketing.tsx
│   ├── _marketing._index.tsx
│   └── _marketing.contact.tsx
└── root.tsx
```

Both routes render into `_marketing.tsx`'s `<Outlet />`.

### Anti-Pattern: Flat Routes

```text
# ❌ DON'T: Flat structure with no shared layout
app/routes/dashboard._index.tsx
app/routes/dashboard.settings._index.tsx
app/routes/dashboard.profile._index.tsx

# ✅ DO: Use a parent layout route with child files
app/routes/dashboard.tsx
app/routes/dashboard._index.tsx
app/routes/dashboard.settings.tsx
app/routes/dashboard.profile.tsx
```

## Index Routes

you typically want to add an index route when you add nested routes so that something renders inside the parent's outlet when users visit the parent URL directly.

```text
app/
├── routes/
│   ├── _index.tsx
│   ├── dashboard.tsx
│   ├── dashboard._index.tsx
│   └── dashboard.settings.tsx
└── root.tsx
```

| URL                   | Matched Route                       |
| --------------------- | ----------------------------------- |
| `/`                   | `app/routes/_index.tsx`             |
| `/dashboard`          | `app/routes/dashboard._index.tsx`   |
| `/dashboard/settings` | `app/routes/dashboard.settings.tsx` |

## Route Prefixes

Use a trailing underscore on the parent segment to opt out of layout nesting:

```text
app/
├── routes/
│   ├── projects.tsx
│   └── projects_.$pid.tsx
└── root.tsx
```

`projects_.$pid.tsx` creates `/projects/:pid` without rendering into `concerts.tsx`.

## Dynamic Segments

Prefix a filename segment with `$` to make it dynamic:

```text
app/
├── routes/
│   └── teams.$teamId.tsx
└── root.tsx
```

```tsx
import type { Route } from "./+types/team";

export async function loader({ params }: Route.LoaderArgs) {
  // params.teamId is typed as string
  return fetchTeam(params.teamId);
}

export default function Team({ params }: Route.ComponentProps) {
  return <h1>Team {params.teamId}</h1>;
}
```

Multiple dynamic segments:

```text
app/
├── routes/
│   └── c.$categoryId.p.$productId.tsx
└── root.tsx
```

The param name comes directly from the filename.

## Optional Segments

Wrap a segment in parentheses to make it optional:

```text
app/
├── routes/
│   ├── ($lang).categories.tsx
│   └── users.$userId.(edit).tsx
└── root.tsx
```

| URL               | Matched Route                       |
| ----------------- | ----------------------------------- |
| `/categories`     | `app/routes/($lang).categories.tsx` |
| `/en/categories`  | `app/routes/($lang).categories.tsx` |
| `/fr/categories`  | `app/routes/($lang).categories.tsx` |
| `/users/123`      | `app/users.$userId.(edit).tsx`      |
| `/users/123/edit` | `app/users.$userId.(edit).tsx`      |

Optional segments match eagerly.

## Splats (Catch-All)

Match any remaining path with a final `$` segment:

```text
app/
├── routes/
│   └── files.$.tsx
└── root.tsx
```

```tsx
export async function loader({ params }: Route.LoaderArgs) {
  const filePath = params["*"]; // e.g., "docs/intro.md"
  return getFile(filePath);
}

// Destructure with rename
const { "*": splat } = params;
```

### 404 Catch-All

Create `app/routes/$.tsx` for unmatched paths:

```tsx
export function loader() {
  throw new Response("Page not found", { status: 404 });
}
```

## Escaping Special Characters

Use `[]` to escape special filename conventions:

| Filename                            | URL                 |
| ----------------------------------- | ------------------- |
| `app/routes/sitemap[.]xml.tsx`      | `/sitemap.xml`      |
| `app/routes/[sitemap.xml].tsx`      | `/sitemap.xml`      |
| `app/routes/weird-url.[_index].tsx` | `/weird-url/_index` |
| `app/routes/dolla-bills-[$].tsx`    | `/dolla-bills-$`    |
| `app/routes/reports.$id[.pdf].ts`   | `/reports/123.pdf`  |

## Folders for Organization

Routes can also be folders with `route.tsx` as the route module. This allows you to organize your code closer to the routes that use them instead of repeating the feature names across other folders:

```text
app/
├── routes/
│   ├── app/
│   │   ├── primary-nav.tsx
│   │   └── route.tsx
│   ├── app._index/
│   │   ├── route.tsx
│   │   └── stats.tsx
│   └── app.projects/
│       ├── project-card.tsx
│       └── route.tsx
└── root.tsx
```

Equivalent route modules:

```text
app/routes/app.tsx
app/routes/app/route.tsx

app/routes/app._index.tsx
app/routes/app._index/route.tsx
```

## See Also

- [Route Module](./route-modules.md) - Route module exports
- [Routing](./routing.md) - Manual `routes.ts` config with route helpers
- [React Router File Route Conventions](https://reactrouter.com/how-to/file-route-conventions)
