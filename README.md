# Campus DoorDash

An on-campus food delivery platform for the UMBC community. Students, faculty and staff order from campus dining locations, and student workers ("dashers") accept and deliver those orders in their spare time — the DoorDash model, scoped to a single campus.

The system is made of four parts that talk to one another:

- **Customer app** — browse campus restaurants and menus, build a cart, pay with Stripe, and follow an order through to delivery.
- **Dasher app** — see available orders, accept one, and drive it through the delivery states to completion.
- **Backend API** — Go service with JWT auth, the order state machine, restaurant/menu endpoints and Stripe webhooks.
- **Data pipeline** — Python scrapers that pull live menu and location data for the 15 campus dining spots into PostgreSQL.

---

## Architecture

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   Expo / React Native app   │        │      Python data pipeline   │
│  customer flow + dasher flow│        │  scrapers → fuse → insert   │
└──────────────┬──────────────┘        └──────────────┬──────────────┘
               │ REST + JWT                           │ Supabase client
               ▼                                      ▼
┌───────────────────────────────────────────────────────────────────────┐
│                     Go API  (Gin, :8080)                              │
│  /auth   /api/restaurants   /api/orders   /api/dashers   /webhooks    │
└──────────────┬────────────────────────────────────┬───────────────────┘
               │ pgx                                 │ Stripe SDK
               ▼                                     ▼
        PostgreSQL (Supabase)                     Stripe
```

The same React Native codebase serves both roles. Routing is decided at startup from the stored session: no token → auth screens, `is_dasher=true` → dasher flow, otherwise the customer flow.

---

## Features

### Customer

- Sign up / log in with JWT access + refresh tokens
- Browse all campus restaurants and drill into a restaurant's menu
- Filter and search the menu, add items to a cart with quantity merging
- Cart with live subtotal, delivery fee, dasher fee and total
- Stripe checkout (Payment Element on web, native Stripe SDK on mobile)
- Order history and per-order detail views
- Live order status through the delivery lifecycle

### Dasher

- Dasher accounts and a dedicated dasher home
- List of available (unassigned) orders
- Accept an order, which claims it atomically
- Active-order view with the current status and customer details
- Status progression: picked up → delivered, with a delivery confirmation screen

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React Native 0.81, React 19, Expo 54, React Navigation, React Native Paper |
| Platform splits | `.web.js` / `.native.js` files resolved by the Metro bundler |
| Client state | React Context (`AuthContext`, `CartContext`) + AsyncStorage |
| Payments (client) | `@stripe/stripe-js` on web, `@stripe/stripe-react-native` on device |
| Backend | Go 1.23, Gin, `gin-contrib/cors` |
| Database | PostgreSQL via `pgx/v5` `pgxpool`, Supabase |
| Auth | JWT access + refresh tokens, bcrypt password hashing, Supabase GoTrue |
| Payments (server) | `stripe-go/v82` with webhook handling |
| Data pipeline | Python, Supabase client, pandas, python-dotenv |
| Tooling | TypeScript config, ESLint (expo config), Go modules |

---

## Repository layout

```
.
├── App.js                     Expo entry point for the root workspace
├── backend/
│   ├── cmd/main.go            server bootstrap, CORS, route table
│   └── internal/
│       ├── auth/              JWT issue/refresh, register, login, middleware
│       ├── orders/            order model, state machine, handlers
│       ├── payments/          Stripe payment intents + webhooks
│       └── restaurants/       restaurant and menu queries
├── data_retrieval/
│   ├── ExternalAPIs/          one scraper per campus dining location
│   └── data_filtering/        cleaning, de-duplication, fusion helpers
├── data_insertion/
│   └── insert_data.py         loads the fused menu data into Supabase
├── frontend/
│   ├── App.web.js             web app root
│   ├── App.native.js          native app root
│   ├── api/                   order + dasher request helpers
│   ├── context/               AuthContext, CartContext
│   ├── screens/               customer screens
│   ├── screens/dasher/        dasher screens
│   └── utils/                 platform-specific Stripe shims
└── my-app/                    Expo Router scratch project (not wired in)
```

---

## Getting started

### Prerequisites

- Go 1.23+
- Node 18+ and npm
- Python 3.10+ (data pipeline only)
- A PostgreSQL database (Supabase works well)

### 1. Backend

```bash
cd backend
cp .env.example .env        # then fill in the values below
go mod download
go run ./cmd/main.go
```

The API listens on `http://localhost:8080`.

### 2. Frontend

```bash
cd frontend
npm install
npx expo start              # then press w / i / a, or scan the QR code
```

The client points at `http://localhost:8080` on web and iOS, and `http://10.0.2.2:8080` on the Android emulator. Both are set in [`frontend/config.js`](frontend/config.js).

### 3. Data pipeline

```bash
cd data_retrieval/data_filtering
python data_fuse.py         # scrape + clean + merge
cd ../../data_insertion
python insert_data.py       # push into Supabase
```

---

## Environment variables

| Variable | Used by | Purpose |
|---|---|---|
| `SUPABASE_URL` | backend, data pipeline | Supabase project URL |
| `SUPABASE_KEY` | data pipeline | Supabase service key for inserts |
| `STRIPE_API_KEY` | backend | Stripe secret key |
| `STRIPE_WEBHOOK_SECRET` | backend | Verifies webhook signatures |
| `EXPO_PUBLIC_STRIPE_PUBLIC_KEY` | frontend | Stripe publishable key |

Nothing secret is committed. All credentials are read from the environment at runtime.

---

## API reference

Public:

| Method | Path | Description |
|---|---|---|
| `GET` | `/auth` | Health / current auth state |
| `POST` | `/auth/register` | Create an account (customer or dasher) |
| `POST` | `/auth/login` | Log in, returns access + refresh tokens |
| `POST` | `/auth/refresh` | Exchange a refresh token |
| `POST` | `/webhooks/stripe` | Stripe payment webhook |

All `/api/*` routes require a `Bearer` access token.

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/restaurants` | List campus restaurants |
| `GET` | `/api/restaurants/:id` | Restaurant details |
| `GET` | `/api/restaurants/:id/menu` | Menu for a restaurant |
| `POST` | `/api/orders` | Create an order and payment intent |
| `GET` | `/api/orders/:id` | Order details |
| `GET` | `/api/customers/:customer_id/orders` | Orders for a customer |
| `GET` | `/api/customers/orders/history` | Customer order history |
| `GET` | `/api/restaurants/:id/orders` | Orders for a restaurant |
| `POST` | `/api/orders/:id/status` | Advance the order status |
| `POST` | `/api/orders/:id/dasher` | Assign a dasher |
| `GET` | `/api/dashers/orders/available` | Unassigned orders |
| `POST` | `/api/dashers/orders/accept/:id` | Claim an order |
| `GET` | `/api/dashers/orders/active` | The dasher's active orders |
| `POST` | `/api/dashers/orders/:id/complete` | Mark delivered |

---

## Order lifecycle

```
pending → confirmed → preparing → ready → picked_up → delivered
   └──────────────────── cancelled ─────────────────────┘
```

Each transition stamps a timestamp column (`confirmed_at`, `ready_at`, `picked_up_at`, `delivered_at`) so the UI can show progress and the backend keeps an audit trail.

---

## Status

The customer and dasher flows are implemented end to end, including Stripe checkout and the full order state machine. Remaining work is mostly hardening: broader test coverage, rate limiting, and moving the remaining mock restaurant imagery over to the scraped dataset.

---

## Author

**Fabricio Rodriguez** — [@rodriguezzfabricio](https://github.com/rodriguezzfabricio)
