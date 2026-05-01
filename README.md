# SmartStore AI 🏪

An intelligent full-stack inventory and vendor management platform for small and medium-sized retail businesses. SmartStore AI automates stock monitoring, supplier purchase orders, demand forecasting, and invoice processing — powered by Groq LLM with real database tool-calling.

> **Demo runs entirely in Google Colab** — single Python file, no Docker or Node.js required.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     GOOGLE COLAB                            │
│              smartstore_ai_final.py                         │
└──────────────────────┬──────────────────────────────────────┘
                       │ Cloudflare Tunnel (trycloudflare.com)
┌──────────────────────▼──────────────────────────────────────┐
│                 FASTAPI BACKEND (Port 8000)                  │
│                                                             │
│  /auth          /products       /suppliers                  │
│  /purchase-orders               /ai/chat                    │
│  /invoices/parse                /dashboard/summary          │
│  /products/{id}/forecast        /automation/run/{job}       │
│  /reports                       /health                     │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         APScheduler (3 Background Jobs)              │   │
│  │  low_stock_agent │ weekly_summary │ expiry_alert     │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────────┘
                       │ SQLAlchemy ORM
┌──────────────────────▼──────────────────────────────────────┐
│                  SQLite Database                             │
│  smartstore.db (auto-created in Colab /content/)            │
│                                                             │
│  users │ categories │ suppliers │ products                  │
│  stock_movements │ purchase_orders │ po_line_items          │
│  automation_logs │ reports                                  │
└──────────────────────┬──────────────────────────────────────┘
                       │ Groq SDK
┌──────────────────────▼──────────────────────────────────────┐
│                    GROQ AI APIs                             │
│  llama-3.3-70b-versatile  — chat + 5 tool functions         │
│  llama-4-scout-17b-16e-instruct — invoice OCR vision        │
└─────────────────────────────────────────────────────────────┘
```

---

## ✨ Modules Built

| Module | What Was Built |
|--------|---------------|
| **A — Inventory Dashboard** | 8 seeded products with SKU, stock, expiry, reorder threshold. Status indicators: 🟢 ok 🟡 low 🔴 critical ⚫ expired. `/dashboard/summary`, `/products` with pagination + filters |
| **B — Supplier & PO Management** | 3 seeded suppliers. Full CRUD. PO workflow: Draft → Sent → Acknowledged → Received. Mock email logged to console |
| **C — AI Store Assistant** | `POST /ai/chat` — Groq LLM with 5 real DB tools. Answers stock/expiry/PO questions from live data. No hallucination |
| **D — Demand Forecast API** | `GET /products/{id}/forecast` — 7-day exponential smoothing using last 30 days of stock movements |
| **E — Invoice OCR Parser** | `POST /invoices/parse` — Groq Vision extracts supplier, date, line items from uploaded image/PDF |
| **F — Agentic Automation** | 3 APScheduler jobs: daily low-stock PO draft, Monday weekly summary, daily expiry alert — all persisted to DB |

---

## 🛠️ Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Runtime | Python | 3.12 |
| API Framework | FastAPI | 0.115+ |
| Data Validation | Pydantic | V2 (`ConfigDict`, `@field_validator`) |
| ORM | SQLAlchemy | 2.x |
| Database | SQLite | (auto-created) |
| Auth | JWT — python-jose | — |
| Password Hashing | **bcrypt direct** (no passlib — compatible with 3/4/5.x) | 5.0.0 |
| Scheduler | APScheduler | 3.x |
| AI / LLM | Groq — `llama-3.3-70b-versatile` | — |
| OCR / Vision | Groq — `llama-4-scout-17b-16e-instruct` | — |
| Public Tunnel | Cloudflared (no account needed) | latest |
| HTTP Client | httpx | — |
| Numerics | numpy | — |

---

## 📁 Repository Structure

```
smartstore-ai/
│
├── smartstore_ai_final.py      # ← entire backend in one file
│                                  (FastAPI + DB + AI + Scheduler)
│
├── docs/
│   ├── sample_invoice_1.png    # Test invoice for OCR demo
│   └── sample_invoice_2.png    # Test invoice for OCR demo
│
├── .env.example                # Required environment variables
└── README.md
```

---

## ⚙️ Environment Variables

```env name=.env.example
# ── REQUIRED ─────────────────────────────────────────────────

# Groq API key — get free at https://console.groq.com
GROQ_API_KEY=gsk_your_groq_api_key_here

# JWT signing secret — must be long and random
# Generate: python -c "import secrets; print(secrets.token_hex(32))"
SECRET_KEY=change-me-to-a-long-random-secret-string

# ── OPTIONAL ─────────────────────────────────────────────────

# Database — SQLite default, swap for PostgreSQL in production
# PostgreSQL: postgresql://user:password@localhost:5432/smartstore
DATABASE_URL=sqlite:///./smartstore.db

# Server port
PORT=8000
```

---

## 🚀 How to Run (Google Colab)

1. Open [Google Colab](https://colab.research.google.com)
2. Create a new notebook
3. Paste the full contents of `smartstore_ai_final.py` into a cell
4. Update the keys at the very top of the file:

```python
os.environ["GROQ_API_KEY"] = "gsk_your_key_here"
os.environ["SECRET_KEY"]   = "your-secret-here"
```

5. Run the cell (`Shift+Enter`)
6. Wait ~60 seconds for installation + startup
7. Look for the output box:

```
╔══════════════════════════════════════════════════╗
║  📖 SWAGGER  → https://xxxx.trycloudflare.com/docs
╚══════════════════════════════════════════════════╝
```

8. Click the Swagger URL → click **Authorize** → enter `admin` / `admin123`
9. Try any endpoint → **Try it out** → **Execute**

> No ngrok token, no Docker, no Node.js needed.

---

## 🔑 Default Login Credentials

| Role | Username | Password | Permissions |
|------|----------|----------|-------------|
| Admin | `admin` | `admin123` | Full access — all endpoints |
| Staff | `staff` | `staff123` | Read + stock updates only, no supplier edit, no PO send |

---

## 📡 All API Endpoints

### Auth
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/auth/token` | Login → access + refresh tokens |
| `POST` | `/auth/register` | Register new user |
| `POST` | `/auth/refresh` | Refresh access token |
| `GET`  | `/auth/me` | Logged-in user info |

### Products
| Method | Path | Description |
|--------|------|-------------|
| `GET`  | `/products` | List — `?page=1&page_size=20&category=&status=&keyword=` |
| `POST` | `/products` | Create product |
| `GET`  | `/products/{id}` | Product detail |
| `PATCH`| `/products/{id}` | Partial update (stock, price, threshold) |
| `DELETE`| `/products/{id}` | Soft delete (is_active=false) |
| `GET`  | `/products/{id}/forecast` | 7-day demand forecast |

### Categories
| Method | Path | Description |
|--------|------|-------------|
| `GET`  | `/categories` | List all categories |
| `POST` | `/categories` | Create category (admin) |

### Suppliers
| Method | Path | Description |
|--------|------|-------------|
| `GET`  | `/suppliers` | List active suppliers |
| `POST` | `/suppliers` | Create supplier (admin) |
| `GET`  | `/suppliers/{id}` | Supplier detail |
| `PATCH`| `/suppliers/{id}` | Update supplier (admin) |
| `DELETE`| `/suppliers/{id}` | Deactivate supplier (admin) |

### Purchase Orders
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/purchase-orders` | Create PO with line items |
| `GET`  | `/purchase-orders` | List — `?supplier_id=&status=&days=90` |
| `PATCH`| `/purchase-orders/{id}/status` | Advance status (Draft→Sent→Acknowledged→Received) |
| `POST` | `/purchase-orders/{id}/send-email` | Mock email — logs to console, sets status=Sent |

### AI & Inventory
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/ai/chat` | Chat with AI — `{"messages": [{"role":"user","content":"..."}]}` |
| `POST` | `/invoices/parse` | Upload invoice image → structured JSON |
| `POST` | `/inventory/receive` | Confirm goods received → updates stock levels |

### Dashboard & Automation
| Method | Path | Description |
|--------|------|-------------|
| `GET`  | `/dashboard/summary` | total_products, low_stock_alerts, expired_items, top_5_fast_movers |
| `POST` | `/automation/run/{job}` | Manually trigger: `low_stock_agent`, `expiry_alert`, `weekly_summary` |
| `GET`  | `/automation/logs` | Last 100 scheduler run logs |
| `GET`  | `/reports` | All generated automation reports |
| `GET`  | `/health` | Server status + model + bcrypt version |

---

## 🤖 AI Integration Details

### LLM: `llama-3.3-70b-versatile` via Groq

> Note: `llama3-groq-70b-8192-tool-use-preview` was used initially but was **decommissioned by Groq**. Switched to `llama-3.3-70b-versatile` which fully supports tool calling.

### 5 Tool Functions (real DB queries, no mock data)

| Tool | Parameters | Returns |
|------|-----------|---------|
| `get_low_stock_products` | `threshold_pct` (optional) | Products at/below reorder threshold with reorder qty |
| `get_product_detail` | `product_id` or `product_name` | Full product info + computed status |
| `get_po_history` | `supplier_name`, `days` (default 30) | POs with line items and status |
| `get_expiring_products` | `days_ahead` (default 14) | Products expiring within N days |
| `create_draft_po` *(bonus)* | `supplier_id`, `line_items[]` | Creates Draft PO and returns PO ID |

### Sample AI Conversations (from live test)

```
❓ Which products are low on stock?
🤖 The products low on stock are Wheat Flour 2kg, Full Cream Milk 1L,
   Potato Chips 150g, and Shampoo 200ml. A draft purchase order has
   been created for these products.

❓ Which products are expiring soon?
🤖 Products expiring soon include Full Cream Milk 1L (expires in 5 days),
   Potato Chips 150g (already expired), and Butter 100g (expires in 10 days).

❓ Show me purchase order history.
🤖 Here is the purchase order history for the last 30 days:
   PO #12: Draft — Full Cream Milk 1L: 38 units, Wheat Flour 2kg: 32 units ...
```

### Demand Forecast Method: Exponential Smoothing

```
s_t = α × x_t + (1 − α) × s_{t−1}     α = 0.3
```

- Source data: last 30 days of `stock_movements` where `delta < 0` (sales)
- Fallback: mean daily sales if fewer than 3 data points exist
- Output: 7 `{date, forecast_demand}` objects — ready for Recharts/Chart.js

**Why not Prophet/ARIMA?** Simpler implementation, zero extra dependencies, adequate accuracy for a 7-day horizon with sparse retail data.

### Invoice OCR: `llama-4-scout-17b-16e-instruct`

Accepts JPG, PNG, WEBP. Returns:
```json
{
  "supplier_name": "AgriSupply Co",
  "invoice_date": "2026-05-01",
  "line_items": [
    {"name": "Basmati Rice 5kg", "qty": 50, "unit_price": 45.0, "total": 2250.0}
  ],
  "grand_total": 2250.0
}
```

---

## ⏰ Scheduled Automation

All three jobs run **automatically in the background** — no button click required.

| Job Name | Cron | What It Does |
|----------|------|-------------|
| `low_stock_agent` | Daily 08:00 | Finds all products ≤ reorder threshold → groups by supplier → creates Draft POs → logs outcome |
| `weekly_summary` | Monday 08:00 | Counts products/low-stock/expired/pending POs → saves markdown Report to DB |
| `expiry_alert` | Daily 08:30 | Finds products expiring ≤14 days → generates markdown table with actions (EXPIRED / Discount30% / Notify) → saves Report |

### Trigger Manually for Demo

```bash
# Requires admin JWT token
curl -X POST https://YOUR-TUNNEL.trycloudflare.com/automation/run/low_stock_agent \
     -H "Authorization: Bearer YOUR_TOKEN"

curl -X POST https://YOUR-TUNNEL.trycloudflare.com/automation/run/expiry_alert \
     -H "Authorization: Bearer YOUR_TOKEN"

curl -X POST https://YOUR-TUNNEL.trycloudflare.com/automation/run/weekly_summary \
     -H "Authorization: Bearer YOUR_TOKEN"
```

**Or in Swagger UI:**
`POST /automation/run/{job_name}` → Try it out → set `job_name` → Execute

View results at `GET /reports` and `GET /automation/logs`.

---

## 🌱 Seeded Demo Data

The app auto-seeds the following on first run:

**Users:** `admin / admin123` (admin) · `staff / staff123` (staff)

**Suppliers:** AgriSupply Co · DairyFresh Ltd · CleanBrand Inc

**Products (8):**

| SKU | Product | Stock | Status |
|-----|---------|-------|--------|
| SKU001 | Basmati Rice 5kg | 150 | 🟢 ok |
| SKU002 | Wheat Flour 2kg | 8 | 🟡 low |
| SKU003 | Full Cream Milk 1L | 12 | 🟡 low |
| SKU004 | Orange Juice 1L | 200 | 🟢 ok |
| SKU005 | Potato Chips 150g | 5 | ⚫ expired |
| SKU006 | Detergent 1kg | 45 | 🟢 ok |
| SKU007 | Shampoo 200ml | 0 | 🔴 critical |
| SKU008 | Butter 100g | 30 | 🟢 ok |

30 days of synthetic `StockMovement` sales history is seeded for each product to power the forecast endpoint.

---

## 🧪 Sample Test Invoices

Included in `/docs/` for testing `POST /invoices/parse`:

| File | Contents |
|------|---------|
| `docs/sample_invoice_1.png` | Single supplier, 4 line items, clear layout |
| `docs/sample_invoice_2.png` | Multi-item FMCG invoice with subtotals and grand total |

---

## ⚠️ Known Limitations

| Area | Current Limitation | Improvement With More Time |
|------|--------------------|---------------------------|
| Database | SQLite — not suitable for concurrent production use | PostgreSQL + Alembic migrations |
| AI Model | `llama-3.3-70b-versatile` occasionally generates malformed tool calls | Retry with backoff + fallback model |
| Forecast | No seasonality detection | Holt-Winters triple exponential smoothing |
| OCR | Fails on low-quality or handwritten invoices | Tesseract fallback + image preprocessing |
| Auth | No password reset or email verification | SMTP integration + reset tokens |
| Real-time | Dashboard does not auto-refresh | WebSocket stock updates |
| Testing | No automated test suite | pytest + 80% coverage + GitHub Actions CI |
| Frontend | Backend-only (Swagger UI for demo) | React frontend with all pages |
| Multi-tenancy | Single store | Per-store data isolation |

---

## 🤝 AI-Assisted Development

Built with assistance from **GitHub Copilot** and **Claude** for:
- FastAPI boilerplate and route generation
- SQLAlchemy relationship definitions
- Pydantic V2 schema migration (`ConfigDict`, `@field_validator`)
- Debugging `bcrypt 5.x` / passlib incompatibility
- Groq tool-calling schema formatting

All AI-generated code was reviewed, tested, and understood. Every part of this codebase can be explained and extended in a live code review.

---

## 📜 Commit Convention

```
feat:      new feature added
fix:       bug fix
refactor:  code restructure without behaviour change
chore:     dependency or tooling update
docs:      documentation only
test:      tests added or updated
```

---

## 📄 License

MIT — see `LICENSE` file.

---

*Submitted for SmartStore AI Technical Assessment — Full-Stack AI Developer (2–3 Years Experience)*
