---
name: ngp-nextjs
description: NGP TechWorld standard for Next.js App Router sites (public, SEO-critical web frontends that consume a Laravel API). Use when creating, editing or reviewing an NGP Next.js project - routes, server and client components, data fetching, i18n, SEO, deployment.
---

# NGP Next.js Standard

> DRAFT v0.1. Items marked ⏳ wait for the approved stack list.
> Also follow `ngp-general`. Everything in `ngp-react` about components, TypeScript, forms,
> styling/RTL, naming and testing applies here too. This file only covers what is different.

## 1. When to use Next.js (and when not)

| Use **Next.js** | Use **React + Vite** (`ngp-react`) |
|---|---|
| Public pages that must rank on Google or show rich link previews: shops, marketing sites, product catalogs, company site | Anything behind a login: dashboards, admin, merchant/vendor panels |
| | Tools/studios that run fully in the browser (3D Gear Designer) |

Decide at project start and write it in `docs/decisions/`. Don't mix both in one app.

## 2. Approved stack

| Area | Choice |
|---|---|
| Framework | Next.js 16, App Router only (no `pages/`), `output: 'standalone'` |
| UI / language / styling | React 19, TypeScript strict, Tailwind 4 (same tokens as `ngp-react`) |
| Server data | `fetch` from Server Components through `src/lib/api/server.ts` (`import 'server-only'`) |
| Client data | TanStack Query, only for interactive data after load (cart, live order status) |
| Client state | Zustand, small stores (cart, auth session, UI) |
| Forms | react-hook-form + zod; mutations through Server Actions or the API client |
| i18n | `[lang]` segment + dictionaries; ⏳ `next-intl` vs current custom |
| Lint | ESLint with `eslint-config-next` (the Next plugin has no oxlint equivalent yet) + Prettier |
| Tests | Vitest + Testing Library; Playwright for critical flows |

## 3. Folder structure

```
src/
├── app/
│   ├── [lang]/
│   │   ├── layout.tsx            html lang/dir, fonts, providers
│   │   ├── page.tsx              home
│   │   ├── (shop)/products/[slug]/page.tsx
│   │   ├── (account)/orders/page.tsx
│   │   ├── loading.tsx, error.tsx, not-found.tsx
│   ├── sitemap.ts, robots.ts
│   └── api/                      route handlers only if really needed (webhooks)
├── features/<feature>/           same idea as ngp-react: components/, api.ts, schemas.ts, actions.ts
├── shared/
│   ├── api/  server.ts (server-only fetch), client.ts (browser fetch), errors.ts
│   ├── ui/   lib/   i18n/   styles/
│   └── stores/                   zustand stores
└── proxy.ts                      locale redirect, auth cookie checks (Next 16 replacement of middleware)
```

Route groups `(name)` organize pages without changing URLs. Keep `page.tsx` files thin: they fetch
data and compose feature components.

## 4. Server vs client components

- **Server Component by default.** Add `'use client'` only to the smallest component that needs
  state, effects, event handlers or browser APIs (a button, a form, a map), not to whole pages.
- Never import a server-only module (`shared/api/server.ts`, secrets) from a client component;
  `import 'server-only'` enforces it.
- Pass serializable props only from server to client components.
- Secrets and the internal API URL stay on the server (`API_URL`). Only `NEXT_PUBLIC_*` reaches the
  browser.

## 5. Data fetching & caching

```ts
// app/[lang]/(shop)/products/[slug]/page.tsx
export default async function ProductPage({ params }: PageProps<'/[lang]/products/[slug]'>) {
  const { lang, slug } = await params;
  const product = await getProduct(slug, lang);   // features/products/api.ts → shared/api/server.ts
  if (!product) notFound();

  return <ProductDetails product={product} />;
}
```

- Every server fetch states its caching on purpose: `next: { revalidate: 300, tags: ['products'] }`
  for catalog data, `cache: 'no-store'` for user-specific data.
- After a mutation, call `revalidateTag` / `revalidatePath` (Server Action) or invalidate the
  TanStack query (client).
- Fetch in parallel (`Promise.all`) when a page needs several independent resources.
- `loading.tsx` / `<Suspense>` for slow sections; `error.tsx` per segment that can fail.

## 6. SEO (the reason we use Next)

- Every page exports `generateMetadata` with translated title, description, canonical URL,
  `alternates.languages` (hreflang), and Open Graph image.
- `sitemap.ts` lists all public pages per language; `robots.ts` blocks account/checkout pages.
- Structured data (JSON-LD) for products, organization, breadcrumbs.
- Images through `next/image` with explicit sizes; fonts through `next/font` (self-hosted).
- `<html lang={lang} dir={lang === 'ar' ? 'rtl' : 'ltr'}>` set in `[lang]/layout.tsx`.
- `SITE_URL` comes from env and must be the real HTTPS domain in production, never an IP.

## 7. Auth

- Customer tokens are stored in an **httpOnly, Secure, SameSite=Lax cookie** set by a Server Action
  or route handler, not in `localStorage`.
- `proxy.ts` redirects unauthenticated users away from account routes; server code reads the
  cookie and calls the Laravel API with it.

## 8. Deployment

- `next build` in CI, ship `.next/standalone` + `.next/static` + `public`, run with `node server.js`
  under systemd behind nginx (HTTPS), or on Vercel ⏳.
- Remove dev-only settings (`allowedDevOrigins` LAN IPs) from committed config, or read them from
  env.

## 9. Checklist before opening a PR

- [ ] `npm run lint && npm run typecheck && npm test && npm run build` pass
- [ ] `'use client'` only where needed; no secrets in client bundles
- [ ] New public pages have metadata, are in the sitemap, and work in `ar` (RTL) and other locales
- [ ] Caching/revalidation chosen explicitly for each new fetch
