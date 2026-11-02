# <Product name>

<One or two sentences: what the product does and for whom.>

## Parts

| Folder | What | Stack |
|---|---|---|
| `backend/` | REST API (`/api/v1`) | Laravel 13, PostgreSQL |
| `dashboard/` | Admin panel | React + Vite |
| `mobile/customer_app/` | Customer app | Flutter |

## Run locally

Requirements: PHP 8.4, Composer, Node 22.22+, PostgreSQL 16, (Flutter stable).

```bash
cd backend && cp .env.example .env && composer install && php artisan key:generate && php artisan migrate --seed
php artisan serve                                  # http://127.0.0.1:8000
cd ../dashboard && npm ci && npm run dev           # http://127.0.0.1:5173
```

Create an admin: `php artisan <product>:create-admin`.

## Tests

```bash
cd backend && php artisan test
cd dashboard && npm test
```

## Docs

- `docs/SPEC.md`: what the product must do
- `docs/ARCHITECTURE.md`: how it is built
- `docs/DEPLOY.md`: servers, deploy, backups
- `docs/decisions/`: why we chose what we chose

Coding standards: NGP standards (`ngp-general` + the stack skill).
