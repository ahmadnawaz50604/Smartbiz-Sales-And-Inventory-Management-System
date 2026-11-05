# SmartBiz Backend (Flask JSON API)

Flask API for the SmartBiz sales & inventory system. Data lives in Neon PostgreSQL; login uses JWT tokens.
Runs on Vercel as a serverless function.

## Structure
```
backend/
├── api/index.py            Vercel entry point (exposes `app`)
├── smartbiz/
│   ├── __init__.py         app factory: JSON settings, CORS, registers all blueprints
│   ├── config.py           environment variables (SECRET_KEY, DATABASE_URL, CORS_ORIGIN)
│   ├── db.py               Neon connection helpers: db(), tx(), q()
│   ├── auth.py             JWT login, role guard (role_required), /api/health
│   ├── utils.py            body(), to_int(), run() - shared request helpers
│   ├── ui.py               builders for the JSON the frontend renders (kpi/table/form/chart)
│   ├── services.py         business rules, one DB transaction each:
│   │                       place_sale, create_po, receive_po, set_po_status, adjust_stock, add_customer
│   └── routes/             one file per stakeholder dashboard
│       ├── owner.py  inventory.py  sales.py  purchasing.py  supplier.py  customer.py
├── requirements.txt        Flask, psycopg2-binary, PyJWT
├── vercel.json             sends every URL to api/index.py
└── .env.example            variables to set
```

## Environment variables
| Name | Value |
|---|---|
| `DATABASE_URL` | Neon connection string (use the **pooled** one, host contains `-pooler`) |
| `SECRET_KEY` | long random string - signs login tokens |
| `CORS_ORIGIN` | `*` while testing, then your frontend URL |

## Endpoints (JSON, header `Authorization: Bearer <token>` except login/health)
| Method & path | Who | What |
|---|---|---|
| `GET /api/health` | anyone | status check |
| `POST /api/login` | anyone | `{username, password}` -> `{token, user}` |
| `GET /api/owner` | owner | KPIs, charts, top products |
| `GET /api/inventory?q=` | inventory, owner | stock, low stock, incoming shipments |
| `POST /api/inventory/adjust` | inventory, owner | `{sku, qty, loc, status}` |
| `POST /api/inventory/receive/<po_id>` | inventory, owner | receive a Shipped PO -> stock increases |
| `GET /api/sales` | sales, owner | orders, customers |
| `POST /api/sales/order` | sales, owner | `{customer_id, sku1, qty1, ..., method}` |
| `POST /api/sales/customer` | sales, owner | `{name, email, phone, city}` |
| `GET /api/purchasing` | purchasing, owner | low-stock reorder list, POs |
| `POST /api/purchasing/po` | purchasing, owner | `{sku, qty}` (blank qty = reorder quantity) |
| `POST /api/purchasing/cancel/<po_id>` | purchasing, owner | cancel a Pending PO |
| `GET /api/supplier` | supplier | own POs and products |
| `POST /api/supplier/ship/<po_id>` | supplier | mark own PO Shipped |
| `GET /api/customer?q=` | customer | catalog, own orders |
| `POST /api/customer/order` | customer | `{sku1, qty1, ..., method}` |

Errors return `{"error": "..."}` with status 400 (bad input), 401 (not signed in) or 403 (wrong role).

## Run locally
```
cd backend
pip install -r requirements.txt
export DATABASE_URL="postgresql://...neon.tech/neondb?sslmode=require"
export SECRET_KEY="dev-secret"
python api/index.py          # http://localhost:5000/api/health
```

## Deploy to Vercel
1. Push the repo to GitHub.
2. Vercel -> Add New -> Project -> import the repo -> **Root Directory: `backend`** -> Framework: Other.
3. Add the three environment variables above -> Deploy.
4. Open `https://<your-backend>.vercel.app/api/health` -> `{"status":"ok"}`.

Database setup: load `database/parts/part_01 ... part_14` into Neon first (see the repo's database folder).
Demo logins (password `Smart@123`): owner, inventory, sales, purchasing, supplier_katz, customer1.
