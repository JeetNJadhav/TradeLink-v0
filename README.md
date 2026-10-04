# TradeLink

TradeLink connects small shop owners (retailers) with their suppliers (distributors). Retailers search products across distributors, see each distributor's live price and stock, place orders, and keep track of their own shelf stock so they reorder at the right time: no over-ordering that ties up cash, and no empty shelves that lose sales. Distributors manage their product listings and stock, and accept, reject and deliver incoming orders.

> Status: active development. The retailer and distributor workspaces are built; the admin workspace is a placeholder.

## Features

**For retailers**

- Search the product catalog with type-ahead suggestions (products, brands, distributors), backed by OpenSearch.
- Compare distributors for a product, with each one's price and available stock.
- Place orders. Stock is reserved at order time, so two shops can never buy the same last unit.
- Order history with status and per-item details.
- **My stock**: track the products on your shelf with on-hand quantity, a reorder point and a target level. Items running low are highlighted, counting what is already on order. Delivered orders are added to your stock automatically.

**For distributors**

- **My products**: list products from the shared catalog with your own price and stock; edit or remove listings.
- Order inbox: accept or reject (with a reason) incoming orders. Rejecting releases the reserved stock.
- **Mark delivered** on accepted orders, which also tops up the retailer's stock.

**For everyone**

- Registration as a retailer or distributor, with shop/business details and location ("Use my current location").
- Profile page and password change (changing the password signs out other devices).

## Tech stack

| Area | Technology |
| --- | --- |
| Frontend | React 19, TypeScript, Vite, React Router, Axios, SCSS |
| Backend | Node.js, TypeScript, [Hapi](https://hapi.dev/), Joi validation |
| Database | PostgreSQL with [Prisma 7](https://www.prisma.io/) (`pg` driver adapter) |
| Search | [OpenSearch](https://opensearch.org/) (full-text search, suggestions, geo distance) |
| Auth | JWT access token and rotating refresh tokens in HttpOnly cookies, CSRF double-submit token, bcrypt password hashing |
| Logging | pino, with a structured `[LEVEL][EVENT_CODE]: key=value` line format and a per-request id |
| Testing | Vitest (backend), ESLint (frontend) |
| Tooling | Docker Compose (OpenSearch), tsx |

## Repository layout

```
.
├── retailer-distributor-backend/    Hapi + Prisma + OpenSearch API
│   ├── prisma/                      schema, migrations, seed script
│   ├── src/
│   │   ├── modules/                 domain modules: auth, profile, product, distributor,
│   │   │                            distributorProduct, order, retailer, retailerStock, search
│   │   ├── infrastructure/          Prisma, OpenSearch, security and logging adapters
│   │   ├── middleware/              auth scheme, role and CSRF checks, request context
│   │   ├── config/                  env validation, routes, logger config
│   │   ├── scripts/                 search index scripts
│   │   ├── app.ts                   composition root (wires everything together)
│   │   └── server.ts                entry point
│   ├── tests/                       Vitest tests
│   └── docker-compose.opensearch.yml
└── retailer-distributor-frontend/   React + Vite single-page app
    └── src/
        ├── app/                     App and routes
        ├── features/                auth, profile, retailer/*, distributor/*
        ├── shared/                  API client, config, shared components and hooks
        └── styles/                  global SCSS
```

There is no root `package.json`. The backend and frontend are two independent npm projects; run each command from inside its folder.

The backend follows a ports-and-adapters layout: each module in `src/modules/` holds routes, controllers, services and repository **interfaces**, and `src/infrastructure/` holds the implementations (Prisma, OpenSearch, JWT, bcrypt, pino). See [CLAUDE.md](CLAUDE.md) for the full architecture and conventions.

## Getting started

### Prerequisites

- **Node.js 22 LTS** or newer, with npm
- **PostgreSQL** (any recent version), running locally or reachable by URL
- **Docker** with Docker Compose, for OpenSearch

### 1. Clone

```bash
git clone https://github.com/JeetNJadhav/TradeLinkMain.git
cd TradeLinkMain
git checkout develop
```

### 2. Start PostgreSQL and OpenSearch

PostgreSQL is not part of the compose file. Use a local install, or start one in Docker:

```bash
docker run -d --name tradelink-postgres \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=tradelink \
  -p 5432:5432 postgres:16
```

Start OpenSearch (single node, security plugin disabled, on port 9200):

```bash
cd retailer-distributor-backend
docker compose -f docker-compose.opensearch.yml up -d
```

### 3. Configure and run the backend

```bash
cd retailer-distributor-backend
npm install
cp .env.example .env
```

Fill in `.env`. The server validates it on startup and refuses to start if a required value is missing:

| Variable | Required | Example / default |
| --- | --- | --- |
| `DATABASE_URL` | yes | `postgresql://postgres:postgres@localhost:5432/tradelink` |
| `JWT_ACCESS_SECRET` | yes | a long random string |
| `ACCESS_TOKEN_TTL` | yes | `15m` |
| `REFRESH_TOKEN_TTL_DAYS` | yes | `7` |
| `ACCESS_TOKEN_TTL_SECONDS` | no | `900` (keep in sync with `ACCESS_TOKEN_TTL`) |
| `HOST` / `PORT` | no | `localhost` / `3000` |
| `CORS_ORIGIN` | no | `http://localhost:5173` |
| `OPENSEARCH_URL` | no | `http://localhost:9200` |
| `NODE_ENV` | no | `development` (set `production` when deploying) |
| `LOG_LEVEL` | no | `info` (`debug`, `info`, `warn`, `error`, `silent`) |
| `LOG_CONSOLE` / `LOG_FILE` | no | `true` / empty (e.g. `logs/app.log` to also write a file) |

Cookie settings (`COOKIE_SECURE`, `COOKIE_SAMESITE`) and cookie/header name overrides are listed in `.env.example`.

Then set up the database and the search index:

```bash
npx prisma generate          # generates the Prisma client into src/generated/prisma (required)
npx prisma migrate dev       # applies all migrations
npx prisma db seed           # optional: loads demo data (wipes every table first)
npm run search:reindex       # builds the OpenSearch "products" index from Postgres
```

Start the API:

```bash
npm run dev                  # http://localhost:3000, restarts on change
```

Check it is up: `curl http://localhost:3000/health`.

### 4. Configure and run the frontend

In a second terminal:

```bash
cd retailer-distributor-frontend
npm install
cp .env.example .env         # VITE_API_URL=http://localhost:3000
npm run dev                  # http://localhost:5173
```

Open http://localhost:5173.

### Demo accounts

After `npx prisma db seed`, every seeded user has the password `SeedPassword123!`:

| Role | Emails |
| --- | --- |
| Retailer | `user1@seed.retaildist.local` to `user600@seed.retaildist.local` |
| Distributor | `user601@seed.retaildist.local` to `user750@seed.retaildist.local` |

The seed creates 1,200 products and 6,000 distributor listings. You can also register a new retailer or distributor at `/register`.

## Scripts

### Backend (`retailer-distributor-backend/`)

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the API with live reload (tsx watch) |
| `npm run build` | Compile TypeScript to `dist/` (also the type check) |
| `npm start` | Run the compiled server from `dist/` |
| `npm test` | Run the Vitest test suite |
| `npm run search:create-index` | Create the `products` index if it does not exist |
| `npm run search:reindex` | Drop, recreate and refill the `products` index from Postgres |
| `npx prisma generate` | Regenerate the Prisma client (after install or a schema change) |
| `npx prisma migrate dev --name <name>` | Create and apply a new migration |
| `npx prisma db seed` | Load demo data (**deletes all existing data first**) |

### Frontend (`retailer-distributor-frontend/`)

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the Vite dev server on port 5173 |
| `npm run build` | Type-check and build for production into `dist/` |
| `npm run preview` | Serve the production build locally |
| `npm run lint` | Run ESLint |

## API overview

All responses use one envelope: `{ "success": true, "data": ... }` or `{ "success": false, "error": { "message": ... } }`. Every path is defined in `retailer-distributor-backend/src/config/routes.ts`.

| Area | Endpoints |
| --- | --- |
| Health | `GET /health` |
| Auth | `POST /auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/password`, `GET /auth/me` |
| Profile | `GET` / `PUT /profile` |
| Search | `GET /search/q` (product search, optional `sortBy=nearest` with `latitude`/`longitude`), `GET /search/suggestions` |
| Catalog | `GET /catalog/products`, `/products/{id}/distributors`, `/distributors/{distributorId}/products`, `/distributor-products/{distributorProductId}` |
| Retailer orders | `POST /orders`, `GET /orders`, `GET /orders/{orderId}` |
| Retailer stock | `GET` / `POST /retailer/stock`, `PUT` / `DELETE /retailer/stock/{retailerStockId}` |
| Distributor listings | `GET` / `POST /distributor/products`, `PATCH` / `DELETE /distributor/products/{distributorProductId}` |
| Distributor orders | `GET /distributor/orders`, `GET /distributor/orders/{orderId}`, `POST .../accept`, `.../reject`, `.../deliver` |

Protected routes need the access cookie; state-changing routes also need the `x-csrf-token` header to match the CSRF cookie. The frontend's API client handles this, including a silent token refresh.

## Good to know

- **Search is served by OpenSearch, not Postgres.** Orders and listing changes made through the app keep the index in sync. After seeding or editing products or distributors directly in the database, run `npm run search:reindex`.
- **Stock is reserved when an order is placed** and released if the distributor rejects it.
- **The seed is destructive.** `npx prisma db seed` deletes every table's data before inserting.
- **Logs** look like `2026-10-03 17:49:10.180 [INFO][ORDER_CREATED]: orderId=o_981`. Every request gets an id, returned in the `x-request-id` header, so a failed request can be found in the log.

## Contributing

1. Branch from `develop`.
2. Follow the conventions in [CLAUDE.md](CLAUDE.md).
3. Before opening a pull request, run the checks:
   - backend: `npm test` and `npm run build`
   - frontend: `npm run lint` and `npm run build`
