# 🍽️ Smart Restaurant Management System (HEMA)
## 📌 Realistic MVP Implementation Plan & Progress Tracker

> **Project Goal:** Single-restaurant, single-floor smart QR ordering & kitchen execution system.  
> **Team:** 2 Part-time Developers (Student workload: ~10–12 hours/week each).  
> **Target Timeline:** 12 Weeks to Pilot Deployment.  
> **Current Status:** `Sprint 0: Setup & Architecture Lock`

---

## 📊 Quick Progress Summary

| Module / Phase | Owner | Target Week | Status | Completion |
|---|---|---|---|---|
| **Phase 0: Setup & Architecture** | Partner A + B | Week 1 | 🟡 In Progress | 0% |
| **Phase 1: Admin & Menu Management** | Partner A (API) / B (UI) | Week 2–3 | ⚪ Not Started | 0% |
| **Phase 2: Customer Ordering Flow** | Partner A (Session/Cart) / B (PWA) | Week 4–5 | ⚪ Not Started | 0% |
| **Phase 3: Realtime KDS & Sound Alerts**| Partner A (Events) / B (KDS Screen) | Week 6–7 | ⚪ Not Started | 0% |
| **Phase 4: Waiter Floor Map & Calls** | Partner A (Requests) / B (Floor UI) | Week 8 | ⚪ Not Started | 0% |
| **Phase 5: Billing & Cash/UPI Closing** | Partner A (Math/DB) / B (Checkout) | Week 9–10 | ⚪ Not Started | 0% |
| **Phase 6: Manager Live Dashboard** | Partner A (Aggregates) / B (Charts) | Week 11 | ⚪ Not Started | 0% |
| **Phase 7: Hardening, Tests & Pilot** | Partner A + B | Week 12 | ⚪ Not Started | 0% |

---

## 🎯 1. MVP Boundaries (Strictly Defined)

### ✅ What IS in the MVP:
1. **Table QR Landing:** Instant mobile PWA, table session token bound.
2. **Digital Menu & Synced Cart:** Multiple guests at the same table see the same live cart.
3. **Kitchen Display System (KDS):** Live order ticket cards with item-level status (`COOKING` ➔ `READY`).
4. **Audio Chime:** Simple, reliable web audio chime for kitchen and waiters.
5. **Waiter Module:** Visual floor map with real-time table colors and "Call Waiter / Water" notifications.
6. **Cash & UPI QR Billing:** Itemized bill with integer paise calculation, CCPA-compliant service charge toggle, sequential invoice number, and staff cash/UPI confirmation.
7. **Basic Manager Live View:** Live sales, active table counts, cash vs UPI totals.

### ❌ What is POST-MVP (Out of Scope for 12 Weeks):
- 🚫 Complex Razorpay webhooks (Use static QR + staff confirmation in MVP).
- 🚫 Browser-dependent Hindi TTS (Replace with audio chime alerts).
- 🚫 Direct Web Bluetooth / iOS thermal printing (Use browser print dialog or simple PC print script later).
- 🚫 Inventory / Raw-material tracking and multi-tier day-end audits.
- 🚫 Multi-tenant SaaS complexity (Build for 1 restaurant first).

---

## 🛠️ 2. Revised Tech Stack (Zero DevOps Friction)

| Layer | Technology | Justification |
|---|---|---|
| **Frontend Framework** | **Next.js 14+ (App Router)** | Single client codebase for Customer, KDS, Waiter, and Admin screens. Fast SSR & PWA capabilities. |
| **Styling & Components**| **Tailwind CSS + shadcn/ui + Lucide Icons** | Pre-built accessible components, clean mobile-first UI. |
| **Backend & Realtime** | **Supabase (PostgreSQL + Realtime + Auth + Storage)** | Eliminates separate Node/Express servers and Vercel WebSocket incompatibilities. Built-in instant pub/sub. |
| **State & Fetching** | **Zustand + TanStack React Query** | Local UI state management and server state caching with auto-reconnect. |
| **Money Handling** | **Integer Paise (`BIGINT` / `INTEGER`)** | 100% immune to JavaScript floating-point rounding errors (e.g., ₹240.50 = `24050`). |
| **Audio Alert** | **HTML5 Audio API (Local `.mp3` / `.wav` chime)** | 100% cross-platform reliable on Chrome, Safari, Android & iOS without OS voice dependency. |

---

## 🗄️ 3. Robust Database Schema (PostgreSQL / Supabase)

```sql
-- 1. Restaurants & Infrastructure
CREATE TABLE restaurants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    address TEXT,
    gstin TEXT,
    service_charge_percent INT DEFAULT 0, -- e.g. 5 for 5% (voluntary)
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tables (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id UUID REFERENCES restaurants(id) ON DELETE CASCADE,
    table_number INT NOT NULL,
    capacity INT DEFAULT 4,
    qr_token TEXT UNIQUE NOT NULL, -- Cryptographic hash, not sequential id
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(restaurant_id, table_number)
);

-- 2. Staff & Authentication
CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id UUID REFERENCES restaurants(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    role TEXT CHECK (role IN ('ADMIN', 'WAITER', 'KITCHEN')) NOT NULL,
    pin_code TEXT NOT NULL, -- 4-digit hashed PIN for quick terminal switch
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Table Sessions (Solves Multi-Guest & Shared Cart)
CREATE TABLE table_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_id UUID REFERENCES tables(id) ON DELETE CASCADE,
    status TEXT CHECK (status IN ('ACTIVE', 'BILL_REQUESTED', 'CLOSED')) DEFAULT 'ACTIVE',
    guest_count INT DEFAULT 1,
    opened_at TIMESTAMPTZ DEFAULT NOW(),
    closed_at TIMESTAMPTZ
);

-- 4. Menu & Modifiers
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id UUID REFERENCES restaurants(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE menu_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT,
    price_paise INT NOT NULL, -- Always in paise (₹100 = 10000)
    tax_percent INT DEFAULT 5, -- Per-item GST (e.g., 5%)
    is_veg BOOLEAN DEFAULT TRUE,
    is_available BOOLEAN DEFAULT TRUE,
    image_url TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE modifiers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id UUID REFERENCES menu_items(id) ON DELETE CASCADE,
    name TEXT NOT NULL, -- e.g., "Extra Cheese"
    extra_price_paise INT DEFAULT 0
);

-- 5. Orders & Item-Level Kitchen Status
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES table_sessions(id) ON DELETE CASCADE,
    order_token_seq INT NOT NULL, -- Daily counter like K-101
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID REFERENCES orders(id) ON DELETE CASCADE,
    item_id UUID REFERENCES menu_items(id),
    name_snapshot TEXT NOT NULL, -- Frozen name at order time
    price_snapshot_paise INT NOT NULL, -- Frozen price at order time
    tax_percent_snapshot INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    selected_modifiers JSONB DEFAULT '[]'::jsonb,
    customer_notes TEXT,
    status TEXT CHECK (status IN ('PLACED', 'ACCEPTED', 'COOKING', 'READY', 'SERVED', 'CANCELLED')) DEFAULT 'PLACED',
    cancelled_reason TEXT,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 6. Service Requests (Waiter Call / Water)
CREATE TABLE service_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES table_sessions(id) ON DELETE CASCADE,
    type TEXT CHECK (type IN ('CALL_WAITER', 'WATER', 'BILL', 'CUTLERY')) NOT NULL,
    status TEXT CHECK (status IN ('PENDING', 'ACKNOWLEDGED', 'RESOLVED')) DEFAULT 'PENDING',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 7. Bills, Payments & Invoices (Sequential GST Compliance)
CREATE TABLE bills (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES table_sessions(id) ON DELETE CASCADE,
    invoice_number TEXT UNIQUE, -- e.g., "INV-2026-0001"
    subtotal_paise INT NOT NULL,
    tax_amount_paise INT NOT NULL,
    service_charge_paise INT DEFAULT 0,
    total_amount_paise INT NOT NULL,
    status TEXT CHECK (status IN ('UNPAID', 'PARTIALLY_PAID', 'PAID')) DEFAULT 'UNPAID',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bill_id UUID REFERENCES bills(id) ON DELETE CASCADE,
    amount_paise INT NOT NULL,
    payment_method TEXT CHECK (payment_method IN ('CASH', 'UPI_QR')) NOT NULL,
    verified_by_staff_id UUID REFERENCES staff(id),
    verified_at TIMESTAMPTZ DEFAULT NOW()
);

-- 8. Audit Log
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id UUID REFERENCES staff(id),
    action TEXT NOT NULL, -- e.g. "ITEM_CANCELLED", "PRICE_OVERRIDE", "BILL_VOIDED"
    details JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 🔄 4. Order & Item State Machine Rules

All transitions must pass through a strict backend validation function:

```
[PLACED] ──(Staff/System Accept)──> [ACCEPTED]
   │                                   │
   ├─(Cancel Allowed: Guest/Staff)     ├─(Cancel Allowed: Staff with Reason)
   ▼                                   ▼
[CANCELLED]                         [COOKING] ──> [NO CANCELLATION ALLOWED]
                                       │
                                (Chef Ready)
                                       ▼
                                    [READY]
                                       │
                                (Waiter Serves)
                                       ▼
                                    [SERVED]
```

### Cancellation Rules:
- `PLACED`: Guest or staff can cancel directly from UI.
- `ACCEPTED`: Only staff can cancel (requires entering a reason).
- `COOKING` / `READY` / `SERVED`: Cancellation strictly prohibited via client API; requires Manager PIN override logged to `audit_logs`.

---

## 👥 5. Two-Person Role Division Matrix

| Responsibility Domain | Partner A (Backend & Data Lead) | Partner B (Frontend & UX Lead) |
|---|---|---|
| **Core Specialty** | Supabase, PostgreSQL, APIs, Realtime Channels, Security & Calculations | Next.js App Router, Tailwind, shadcn/ui, Responsive Views, Client Cache & Audio |
| **Authentication & Access** | Supabase Auth, Row Level Security (RLS), Staff PIN verification RPCs | Staff PIN login screen, role switcher UI, persistent session storage |
| **Menu & Catalog** | Menu tables, image upload to Supabase Storage, tax config logic | Menu CRUD admin view, category tabs, item card components, search filter |
| **Ordering Flow** | Cryptographic QR tokens, Table Session management, Order API + State machine | Mobile QR scanner landing page, real-time shared cart drawer, order tracker timeline |
| **Kitchen (KDS)** | Real-time channel broadcasts (`kitchen-orders`), status update RPCs | KDS responsive grid screen, ticket cards, timer counters, HTML5 audio chime |
| **Waiter Operations** | Service requests API, staff alert dispatch, order item status mutations | Interactive visual table floor map, color badges, waiter alert drawer |
| **Billing & Settlement** | Integer paise math, per-item tax calculation, sequential invoice generator | Digital receipt UI, CCPA service charge voluntary toggle, Cash/UPI modal |
| **Manager Dashboard** | Aggregation SQL queries (today's revenue, active tables, payments breakdown) | Executive dashboard layout, metric cards, order history table, QR exporter |

---

## 📅 6. Step-by-Step 12-Week Implementation Checklist

### 🏁 Phase 0: Setup & Architecture Lock (Week 1)
- [ ] **Both:** Initialize GitHub repo, Next.js 14 project, Tailwind CSS, shadcn/ui.
- [ ] **Partner A:** Create Supabase project, execute initial SQL schema migrations.
- [ ] **Partner A:** Set up Row Level Security (RLS) policies and test connection with `@supabase/supabase-js`.
- [ ] **Partner B:** Configure global design tokens, mobile-first container, Lucide icons, and base UI layout.
- [ ] **Both:** Insert seed data (1 Restaurant, 10 Tables, 4 Categories, 15 Menu Items, 3 Staff members).
- [ ] **Milestone 0:** Project runs locally, connects to Supabase, and queries seed menu items.

---

### 📋 Phase 1: Admin & Menu Management (Weeks 2–3)
- [ ] **Partner A:** Build API / Supabase queries for Category CRUD and Menu Item CRUD.
- [ ] **Partner A:** Configure Supabase Storage bucket for dish photos with public read policy.
- [ ] **Partner A:** Build staff PIN authentication helper function.
- [ ] **Partner B:** Build Admin dashboard navigation and layout.
- [ ] **Partner B:** Build Category management screen (Add, re-order, delete).
- [ ] **Partner B:** Build Menu Item manager modal (Image upload, name, description, price in ₹, tax %, veg toggle).
- [ ] **Partner B:** Build Table generator screen (Generate table 1–10 and view cryptographic QR links).
- [ ] **Milestone 1 Demo:** Admin can log in with PIN, add a new dish with an image, and toggle dish availability.

---

### 📱 Phase 2: Customer Ordering PWA & Shared Cart (Weeks 4–5)
- [ ] **Partner A:** Implement secure QR token decoder endpoint: maps `token` ➔ `table_id` & creates/fetches active `table_session`.
- [ ] **Partner A:** Implement Cart sync endpoints / table session state so multiple phones see identical cart.
- [ ] **Partner A:** Implement `POST /api/orders` with strict price snapshotting and state machine initialization (`PLACED`).
- [ ] **Partner B:** Build mobile landing page `/table/[token]` (zero app download, sleek header showing Table number).
- [ ] **Partner B:** Build interactive menu catalog (Category tabs, Veg/Non-Veg badges, search bar, item modal).
- [ ] **Partner B:** Build synchronized sliding Cart drawer with quantity increment/decrement.
- [ ] **Partner B:** Build Live Order Status timeline card (`Placed` ➔ `Cooking` ➔ `Ready` ➔ `Served`).
- [ ] **Milestone 2 Demo:** Two different phones scan Table 3 QR, add items to the cart simultaneously, place the order, and see live status.

---

### 🍳 Phase 3: Realtime KDS & Sound Alerts (Weeks 6–7)
- [ ] **Partner A:** Configure Supabase Realtime broadcast channel `restaurant-[id]:kds`.
- [ ] **Partner A:** Build database trigger/function that broadcasts `NEW_ORDER` and `ITEM_STATUS_CHANGE` events.
- [ ] **Partner A:** Implement item-level status mutation RPC (`update_item_status(item_id, new_status)`).
- [ ] **Partner B:** Build KDS Screen (`/kitchen`) designed for a tablet or monitor landscape view.
- [ ] **Partner B:** Build order ticket cards showing elapsed time timer, table number, order token (e.g. `K-102`), and items.
- [ ] **Partner B:** Implement audio chime trigger on incoming order using HTML5 Web Audio API (with "Click to Enable Audio" banner).
- [ ] **Partner B:** Build item-level action buttons: `Start Cooking` ➔ `Mark Ready` ➔ `Mark Void`.
- [ ] **Milestone 3 Demo:** Placing an order from a phone triggers a chime on the kitchen screen within 200ms, displaying the order card instantly.

---

### 🛎️ Phase 4: Waiter Floor Map & Service Requests (Week 8)
- [ ] **Partner A:** Build `service_requests` realtime channel and mutation APIs (`CALL_WAITER`, `WATER`, `BILL`).
- [ ] **Partner A:** Create floor map summary query (Aggregates each table's active state: Free, Dining, Ready, Bill).
- [ ] **Partner B:** Build Waiter interface (`/waiter`) with PIN login.
- [ ] **Partner B:** Build visual Table Grid / Floor Map with dynamic color codes:
  - 🟢 Green: Free
  - 🔵 Blue: Dining / Order Cooking
  - 🟡 Yellow: Food Ready in Kitchen (Blinking badge)
  - 🔴 Red: Waiter Assistance or Bill Requested
- [ ] **Partner B:** Build slide-over notification drawer for incoming requests with quick action: `Acknowledge` / `Resolve`.
- [ ] **Partner B:** Add single-tap `Mark as Served` action on ready items.
- [ ] **Milestone 4 Demo:** Chef marks item "Ready" on KDS ➔ Waiter's phone flashes yellow with chime ➔ Waiter taps "Mark Served".

---

### 💳 Phase 5: Billing, Tax & Cash/UPI Settlement (Weeks 9–10)
- [ ] **Partner A:** Implement Bill Generation Engine:
  - Calculate subtotal from frozen item prices.
  - Compute per-item GST (5% default, custom per item).
  - Add optional service charge only if enabled in settings and confirmed by guest.
  - Generate sequential GST-compliant invoice format: `INV-YYYYMM-XXXX`.
- [ ] **Partner A:** Build payment settlement RPC (`settle_bill`): updates bill to `PAID` and table session to `CLOSED`.
- [ ] **Partner B:** Build Customer Digital Receipt view (`/table/[token]/bill`) with itemized breakdown and GST display.
- [ ] **Partner B:** Add voluntary "Service Charge" toggle with clear explanation per CCPA guidelines.
- [ ] **Partner B:** Build Payment Mode selector:
  - "Pay via UPI QR": Displays restaurant's static UPI QR with auto-filled total amount.
  - "Pay via Cash": Alerts waiter to collect cash at table.
- [ ] **Partner B:** Build Waiter/Cashier settlement confirmation modal (`Confirm Cash/UPI Received`).
- [ ] **Milestone 5 Demo:** Customer reviews bill on phone, waiter accepts cash, confirms on device, bill marks PAID, table resets to Free.

---

### 📈 Phase 6: Manager Operations Dashboard (Week 11)
- [ ] **Partner A:** Build reporting API queries (Today's Gross Sales, Total Orders, Active Tables, Cash vs UPI split).
- [ ] **Partner A:** Build audit log viewer API.
- [ ] **Partner B:** Build Manager Dashboard (`/admin/dashboard`):
  - Metric cards: Gross Sales (₹), Active Dining Tables (e.g. 4/10), Bills Settled count.
  - Payment split breakdown bar: Cash vs UPI.
  - Live orders stream table with status badges.
- [ ] **Partner B:** Add Printable QR PDF generation utility (Table cards ready to print on A6 standees).
- [ ] **Milestone 6 Demo:** Manager opens laptop, sees live daily earnings and transaction logs updating in real time.

---

### 🛡️ Phase 7: Hardening, Unit Tests & Pilot Testing (Week 12)
- [ ] **Partner A:** Write Vitest/Jest unit tests for:
  - Paise bill math (rounding, taxes, multi-item accuracy).
  - Order state machine invalid transition rejections.
  - Table token signature verification.
- [ ] **Partner B:** Implement Offline & Reconnection handling:
  - When connection drops and restores, client fetches full snapshot from server rather than relying on missed events.
- [ ] **Both:** Conduct physical end-to-end rehearsal (simulate 4 tables simultaneously with 4 phones and 1 kitchen screen).
- [ ] **Both:** Deploy to Vercel (Frontend) + Supabase Cloud (Database & Realtime).
- [ ] **Milestone 7 (Final):** Live pilot run at a friendly campus canteen, café, or dhaba.

---

## ⚠️ 7. Critical Technical Safeguards & Solutions

### 1. Zero Floating-Point Arithmetic
All prices are stored in database and calculated in backend as integer paise:
```typescript
// Calculation Helper
const subtotalPaise = items.reduce((acc, item) => acc + (item.price_paise * item.quantity), 0);
const taxPaise = items.reduce((acc, item) => acc + Math.round((item.price_paise * item.quantity * item.tax_percent) / 100), 0);
const serviceChargePaise = applyServiceCharge ? Math.round((subtotalPaise * servicePercent) / 100) : 0;
const totalPaise = subtotalPaise + taxPaise + serviceChargePaise;

// Format for UI Display
export const formatRupee = (paise: number) => `₹${(paise / 100).toFixed(2)}`;
```

### 2. Browser Audio Autoplay Policy
Browsers block audio autoplay without a prior user gesture.
- **Solution:** On the KDS and Waiter screens, render a persistent top banner if audio is locked:
  `🔔 Tap anywhere to activate audio alerts for new orders`.
  Once clicked, unlock an `AudioContext` that plays the chime smoothly on all subsequent WebSocket events.

### 3. Reconnect & State Synchronization
Never assume all WebSocket events are received over patchy restaurant Wi-Fi.
- **Rule:** Every client maintains a query hook that re-fetches the entire active state from Supabase whenever `window.ononline` or `socket.on('connect')` fires.

