---
name: ngp-react
description: NGP TechWorld standard for React single-page apps built with Vite, TypeScript, Tailwind CSS, React Router and TanStack Query (dashboards, admin panels, customer web apps). Use when creating, editing or reviewing any NGP React/Vite frontend - components, hooks, API calls, forms, routing, styling, tests.
---

# NGP React + Vite Standard

> DRAFT v0.1. Items marked ⏳ wait for the approved stack list.
> Also follow `ngp-general`. For public sites that need SEO, see `ngp-nextjs`.

## 1. Approved stack

| Area | Choice | Notes |
|---|---|---|
| Language | TypeScript, `"strict": true` written explicitly | No new `.jsx`/`.js` files in `src/`. |
| UI | React 19 | Lama and 3D Gear Designer (React 18) upgrade when next touched. |
| Build | Vite 8 + `@vitejs/plugin-react` | |
| Styling | Tailwind CSS 4, CSS-first config with tokens in `src/shared/styles/index.css` | Projects on Tailwind 3 migrate when next touched. Plain CSS files only for things Tailwind can't express. |
| Routing | React Router, data router (`createBrowserRouter`), lazy routes | ⏳ v7 vs v8 |
| Server state | TanStack Query 5 | All data from the API goes through it. |
| Client state | React state + context. Zustand only for real cross-page client state (cart, wizard). | No Redux. |
| Forms | react-hook-form + zod + `@hookform/resolvers` | |
| HTTP | Native `fetch` wrapper in `src/shared/api/client.ts` | No axios in new projects. |
| API types | `openapi-typescript` generated from the backend's Scramble OpenAPI | It needs TypeScript 5 as a peer; with TypeScript 7 run it through a pinned `npx --yes openapi-typescript@x.y.z` in `api:types` instead of a devDependency. Commit the generated `schema.d.ts`. |
| i18n | i18next + react-i18next ⏳ | Not a custom i18n module. |
| Icons | lucide-react | |
| Accessible primitives | Radix UI (dialog, popover, tabs...) wrapped in our own `shared/ui` | No full UI kit (MUI, Ant). |
| Lint | oxlint (react, typescript, oxc plugins) ⏳ oxlint vs ESLint | `react-hooks/rules-of-hooks` = error. |
| Format | Prettier + `prettier-plugin-tailwindcss` ⏳ | Every project, no exceptions. |
| Unit tests | Vitest + Testing Library + MSW | |
| E2E | Playwright, mobile viewport, `ar` locale | |
| Package manager | npm ⏳ (pnpm only for multi-app workspaces like Lama) | |

## 2. Folder structure (feature-based)

Taken from Dawri, the cleanest frontend we have:

```
src/
├── app/                   app shell: App.tsx, router.tsx, providers.tsx, guards/, layouts/
├── features/
│   └── bookings/
│       ├── api.ts         query keys + useXxxQuery / useXxxMutation hooks for this feature
│       ├── types.ts       feature types (or re-exports of generated API types)
│       ├── schemas.ts     zod schemas for this feature's forms
│       ├── components/    components used only by this feature
│       ├── pages/         route components (BookingsPage.tsx)
│       └── index.ts       public exports of the feature (only what other code may use)
├── shared/
│   ├── api/               client.ts (fetch wrapper, ApiError), generated schema.d.ts
│   ├── ui/                our design system: Button, Input, Dialog, Toast, EmptyState...
│   ├── lib/               framework-free helpers (format, dates, money, phone)
│   ├── hooks/             generic hooks (useDebounce, useMediaQuery)
│   ├── i18n/              setup + locales/ar.json, en.json
│   └── styles/            index.css (Tailwind + design tokens)
└── main.tsx
```

Rules:
- A feature may import from `shared/` and from another feature's `index.ts` only, never from
  another feature's internal files.
- `shared/` never imports from `features/`.
- Something used by two features moves to `shared/` (or stays in one feature and is exported).
- Use the `@/` path alias for `src/` (`import { Button } from '@/shared/ui'`). No `../../../`.

Several apps (customer / vendor / admin) in one product: one Vite app per folder in the repo
(`web/`, `dashboard/`), each with this structure. Share code through a workspace package only when
the duplication is real.

## 3. Components

- Function components only. One component per file; file name = component name (`BookingCard.tsx`).
- Files: components `PascalCase.tsx`, hooks `useCamelCase.ts`, everything else `camelCase.ts`.
- Named exports (`export function BookingCard`). Default exports only where a tool requires it
  (lazy routes can use `lazy(() => import(...).then(m => ({ default: m.X })))` or `Component` export).
- Props typed with a `type XxxProps = {...}` right above the component.
- Keep components under ~150 lines. When a component fetches data *and* renders a complex UI,
  split it: the page fetches, child components receive props.
- No business logic in JSX. Compute values above the `return`, or in a hook.
- Every list/data screen handles **loading, error and empty** states with the shared components.

```tsx
type BookingCardProps = {
  booking: Booking;
  onCancel: (id: string) => void;
};

export function BookingCard({ booking, onCancel }: BookingCardProps) {
  const { t } = useTranslation();
  const canCancel = booking.status === 'pending';

  return (
    <Card>
      <h3 className="text-lg font-semibold">{booking.salonName}</h3>
      <p className="text-muted">{formatDateTime(booking.startsAt)}</p>
      {canCancel && <Button variant="ghost" onClick={() => onCancel(booking.id)}>{t('bookings.cancel')}</Button>}
    </Card>
  );
}
```

## 4. Data fetching (TanStack Query)

- Components never call `fetch` directly. They call a hook from the feature's `api.ts`.
- Query keys are built by a small factory per feature, so invalidation is predictable:

```ts
// features/bookings/api.ts
export const bookingKeys = {
  all: ['bookings'] as const,
  list: (filters: BookingFilters) => [...bookingKeys.all, 'list', filters] as const,
  detail: (id: string) => [...bookingKeys.all, 'detail', id] as const,
};

export function useBookings(filters: BookingFilters) {
  return useQuery({
    queryKey: bookingKeys.list(filters),
    queryFn: () => api.get<Paginated<Booking>>('/bookings', { query: filters }),
  });
}

export function useCancelBooking() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => api.post(`/bookings/${id}/cancel`),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: bookingKeys.all }),
  });
}
```

- `src/shared/api/client.ts` is the only place that knows the base URL, credentials
  (`credentials: 'include'` + CSRF for Sanctum cookies), headers (`Accept-Language`), and turns
  every failure into one `ApiError { status, message, code, errors }` matching the backend error
  shape. Show `error.message` to the user; it is already translated by the backend.
- Validation errors (422) are mapped onto the form fields with `setError`.
- Server data is never copied into `useState` or a store. Read it from the query.

## 5. Forms

- react-hook-form + zod schema in the feature's `schemas.ts`. The TypeScript type comes from the
  schema (`z.infer<typeof schema>`), never written twice.
- Inputs come from `shared/ui` and show the field error under the input.
- Submit buttons are disabled and show a spinner while the mutation is pending.

## 6. Styling

- Tailwind utility classes in JSX. Design tokens (brand colors, radius, fonts) defined once in
  `index.css` with `@theme`; no raw hex colors in components.
- Variants with a small helper (`cn()` = `clsx` + `tailwind-merge`), not string concatenation.
- **RTL:** logical utilities only (`ms-*`, `me-*`, `ps-*`, `pe-*`, `start-*`, `end-*`, `text-start`).
  `ml-`/`mr-`/`pl-`/`pr-`/`left-`/`right-` are forbidden; add Dawri's `lint-rtl` script to CI.
- Mobile-first: base classes for phones, `md:`/`lg:` for larger screens.
- Fonts self-hosted with `@fontsource`; at most 2 families per app.

## 7. Routing

- One `router.tsx` in `src/app`. Every page route is lazy-loaded.
- Auth and role checks live in route guards (`app/guards/RequireAuth.tsx`, `RequireRole.tsx`),
  not inside pages.
- URL holds shareable state (filters, tabs, pagination) via search params.

## 8. TypeScript rules

- `strict`, `noUnusedLocals`, `noUnusedParameters`, `noUncheckedIndexedAccess` on.
- No `any`. Use `unknown` and narrow, or a proper type. `// @ts-expect-error` needs a comment why.
- API types are generated, not hand-written. Regenerate with `npm run api:types` when the backend
  changes.
- `type` for props and data shapes; `interface` only when extending.

## 9. Environment

- Only `VITE_`-prefixed variables reach the browser; never put secrets in them.
- Read env in one file (`src/shared/config.ts`) and validate with zod at startup.
- In dev, Vite proxies `/api`, `/sanctum`, `/storage` to the Laravel server so cookies behave
  like production (same origin).
- Node 24 LTS at its latest security release: `package.json` has
  `"engines": { "node": "^24.22.2" }` (the floor on 2026-11-03), `.npmrc` has
  `engine-strict=true`, and CI uses `node-version: '24'`. See `ngp-general` §10. The scripts:
  `dev`, `build`, `preview`, `lint`, `format`, `typecheck`, `test`, `test:e2e`, `api:types`.

## 10. Testing

- Vitest + Testing Library for components with logic, hooks and `shared/lib` helpers.
- MSW for API mocking in tests (not a hand-made mock mode inside the app). MSW 3: import `http`
  and `HttpResponse` from `msw`, and start with `server.listen({ onUnhandledFrame: 'error' })`
  (renamed from `onUnhandledRequest`).
- Test what the user sees (`getByRole`, `getByText`), not implementation details.
- Playwright for the 3-5 critical flows (login, main purchase/booking flow, admin approval);
  run in CI.

## 11. Checklist before opening a PR

- [ ] `npm run lint && npm run typecheck && npm test && npm run build` pass
- [ ] No `console.log`, no `any`, no unused files
- [ ] New strings are in `ar.json` (and other locales); no hard-coded Arabic in components
- [ ] Loading / error / empty states handled
- [ ] Checked on a 375px screen and in RTL
