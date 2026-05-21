# frontend.md

Review criteria for the React + Vite + TanStack Router frontends under
`apps/web/`. `apps/web/product-innovation-lab/` (`product.air-closet.ai`) is the
reference implementation.

## Authentication

### Always provide login and logout

Every protected app must include the following.

- A logout trigger in the top-right of the header (either an icon button or an
  item inside the display-name menu)
- A `/login` route for unauthenticated users (or a full-page redirect to the
  Edge Auth Worker)
- An automatic redirect on 401 responses (honor the `X-Edge-Auth-Start` header,
  with `/login` as a fallback)

Reference: `loginMutation` / `logoutMutation` in
`apps/web/product-innovation-lab/src/hooks/useAuth.ts`, and
`apps/web/product-innovation-lab/src/components/auth/ProtectedRoute.tsx`.

| severity | check |
|---|---|
| Critical | Is the logout UI reachable at all times (one click from any protected route)? |
| Critical | When an unauthenticated user accesses a protected route directly, are they redirected to a login URL? |
| Major | Does the API client navigate to the Edge Auth start URL when it sees a 401 + `X-Edge-Auth-Start`? |

## Theme

### Ship light and dark themes with a switcher

- Place a toggle (Sun / Moon icon) in the top-right of the header
- Persist the selection to `localStorage` (preserved across reloads)
- Honor `prefers-color-scheme` on the first visit
- Pull colors through CSS variables (e.g. `var(--color-bg-base)`) and stay
  within Tailwind semantic classes like `bg-bg-base`

Reference: `apps/web/repository-console/src/hooks/useTheme.ts` and
`apps/web/product-innovation-lab/src/hooks/useTheme.ts`.

| severity | check |
|---|---|
| Major | Is the theme toggle present in the global header? |
| Major | Is the selection persisted to `localStorage`? |
| Minor | Does the first load honor `prefers-color-scheme`? |
| Minor | Are colors pulled through CSS variables / semantic Tailwind classes rather than hardcoded? |

## Sidebar

### Always collapsible to an icon-only state

- Two states: expanded and icon-only
- Persist the state to `localStorage` (preserved across reloads)
- While collapsed, hover should expand it temporarily (operators check labels
  frequently in business UIs)
- Switch to a drawer at mobile widths (toggled by a hamburger menu)

Reference: `apps/web/repository-console/src/hooks/useSidebar.ts` and the
`AppLayout` in `apps/web/repository-console/src/app.tsx`.

| severity | check |
|---|---|
| Major | Is there a toggle to collapse the sidebar to icons only? |
| Major | Is the collapsed state persisted to `localStorage`? |
| Minor | Does hover expand the collapsed sidebar temporarily, and is there a mobile drawer? |

## Loading

### Show skeletons via Suspense fallbacks

Never leave the screen blank during data fetching. Wrap content in
`<Suspense fallback={<XxxSkeleton />}>` and render a skeleton that matches the
shape of the real content.

- Match the skeleton to the content's layout (card outlines, table rows, chart
  heights) to minimize visual jump
- Centralize skeletons under `components/feedback/` and reuse them at the right
  granularity (`DashboardSkeleton`, `KpiTableSkeleton`, etc.)
- Use a single spinner only for cases where the shape is genuinely unknown,
  such as top-level auth bootstrap. Everything else must use a skeleton
- Prefer `suspense: true` + `<Suspense>` over branching on `useQuery`'s
  `isLoading` flag (the code stays flat, and the loading representation is
  centralized in one place)

Reference: `apps/web/product-innovation-lab/src/components/feedback/DashboardSkeleton/`,
`KpiTableSkeleton/`, `LoadingScreen/`, `ErrorState/`.

| severity | check |
|---|---|
| Major | For routes / panels that fetch data, is there a matching skeleton wrapped in `<Suspense>`? |
| Major | Does the skeleton's shape (card count, row count, height) match the real content? |
| Minor | Are the skeleton components centralized under `components/feedback/`? |

## Routing

### Deep links must not 404

Deep links that jump directly to detail screens from external sources — push
notifications, email, GitHub Check Run summaries, Slack — must be treated with
the same quality as normal navigation from the sidebar or a list view.

- Always render a fallback (empty state, link back to the list) when the target
  ID cannot be found
- The detail screen must initialize from path parameters alone, without
  depending on the previous screen state or `sessionStorage`
- When you can distinguish fetch failure, missing permission, and deletion,
  tailor the display to the user's next available action

Reference: the `/secretlint/runs/$runId` deep link in
`apps/web/repository-console/src/routes/SecretlintPage.tsx`.

| severity | check |
|---|---|
| Major | Is there a "resource not found" fallback when the deep link arrives? |
| Major | From the fallback state, can the user get back to a list, history, or other recovery path? |
| Major | Is the data needed for the initial detail render fetched from path parameters alone? |

## Layout

### Each pane scrolls independently in two-pane (or more) list/detail layouts

In screens with multiple horizontally-placed panes — for example, list +
detail (master-detail) — every pane must scroll independently. Do not collapse
the whole page or `<body>` into a single scrollbar.

- Pin the root container to a fixed height (e.g.
  `flex h-[calc(100vh-<header>-<padding>)] flex-col`) and let the inner grid /
  flex absorb it with `min-h-0 flex-1 overflow-hidden`
- Apply `min-h-0 overflow-y-auto` directly to each pane (inside an
  `overflow-hidden` parent, forgetting `min-h-0` means the flex/grid item
  refuses to shrink and never scrolls)
- Fix each pane's header and toolbar with `shrink-0`, and let only the body
  beneath them scroll with `flex-1 overflow-y-auto`
- Do not sync scroll position across panes

Reference: `apps/web/repository-console/src/routes/SecretlintPage.tsx`.

| severity | check |
|---|---|
| Major | In two-pane (or more) layouts, does each pane have `overflow-y-auto` + `min-h-0`? |
| Major | Does the root absorb a fixed height (`h-[calc(...)]` / `h-screen`)? |
| Minor | Are pane headers and toolbars pinned with `shrink-0`? |

## Display

### Omit the organization prefix in list views when there is only one organization

When every row has the same owner — for example, `air-closet/foo` — display
only `foo`. Repeating the organization prefix on every line is noisy and hurts
scannability. Use the full name only for link UIs intended for copy-paste.

| severity | check |
|---|---|
| Minor | In a list bound to a single organization, are you avoiding the `owner/name` full-name format? |
