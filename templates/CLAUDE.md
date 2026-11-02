# <Product name>

Follow the NGP standards: `ngp-general` always, plus `ngp-laravel` in `backend/`,
`ngp-react` in `dashboard/` and `web/`, `ngp-nextjs` for Next.js apps, `ngp-flutter` in `mobile/`.

## Local setup
- API: http://127.0.0.1:8000 (PostgreSQL on port <port>, DB `<product>` and `<product>_test`)
- Dashboard: http://127.0.0.1:5173 (Vite proxies /api, /sanctum, /storage to the API)

## Commands
- Backend checks: `vendor/bin/pint && vendor/bin/phpstan analyse && php artisan test`
- Frontend checks: `npm run lint && npm run typecheck && npm test && npm run build`

## Don't
- Don't edit migrations that already ran on staging/production; add a new one.
- Don't commit `.env*` files (except `.env.example`).
- <product-specific rules>
