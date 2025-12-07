# Stripe Checkout – Technical Assessment

## Tech Stack

* **Database:** Supabase Postgres
* **Backend:** Supabase Edge Functions (TypeScript / Deno)
* **Frontend:** Next.js + TypeScript
* **Payments:** Stripe Checkout + Webhooks

---

## Project Structure

```
backend/
├── schema.sql                     # Database schema (payment_requests, applications)
├── rls_policies.sql               # RLS policies for secure access
└── edge-functions/
    └── create-checkout/
        └── index.ts               # Creates Stripe Checkout session

frontend/
├── pages/
│   ├── index.tsx                  # Landing page
│   └── application/
│       └── pay.tsx                # Pay Application Fee page
├── lib/
│   ├── supabaseClient.ts          # Supabase client setup
│   └── stripe.ts                  # Stripe client wrapper
├── package.json
└── .env.local                     # Environment variables (not committed)
```

---

## Setup Instructions

### 1. Create a Supabase Project

* Go to **supabase.com** → create a new project
* Copy your **Project URL** and **Anon Service Role Keys**
  (Settings → API)

### 2. Set Up the Database

Run the SQL files **in order** inside the Supabase SQL Editor:

1. `backend/schema.sql` – Creates `payment_requests`, `applications`, and related indexes
2. `backend/rls_policies.sql` – Enables RLS for secure access

### 3. Configure Stripe

Add to **frontend/.env.local** and **backend Edge Function environment vars**:

```
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
```

### 4. Install Dependencies & Run

```bash
cd frontend
npm install
npm run dev
```

Visit:
**[http://localhost:3000](http://localhost:3000)**

---

## Implementation Summary

### Task 1: Database Schema

**File:** `backend/schema.sql`

Created tables:

* **applications** – stores basic application records
* **payment_requests** – tracks payment attempts for app fees

Key constraints:

* `payment_requests.status` restricted to: `pending`, `completed`, `failed`
* Foreign key: `application_id → applications.id`
* Indexes on `status`, `session_id`, and `application_id`

---

### Task 2: RLS Policies

**File:** `backend/rls_policies.sql`

Policies include:

* **SELECT:** Users can only view payment_requests for their own applications
* **INSERT:** Only authenticated users can create payment requests
* **UPDATE:** Only backend Edge Functions (service role) can update payment status

---

### Task 3: Edge Function – Create Checkout Session

**File:** `backend/edge-functions/create-checkout/index.ts`

This function:

* Inserts a `payment_requests` row (status = `pending`)
* Calls `stripe.checkout.sessions.create()` with:

  * Amount
  * Success + Cancel URLs
  * `metadata.payment_request_id`
* Stores `session_id` back into Supabase
* Returns the Checkout URL to the frontend

Handles:

* 400 → Invalid input
* 500 → Stripe/internal errors

---

### Task 4: Frontend Payment Page

**File:** `frontend/pages/application/pay.tsx`

Features:

* Calls Edge Function to create a Stripe checkout session
* Redirects user to `session.url`
* Displays payment status after returning from Stripe
* Polls backend or waits for webhook update

---

## Stripe Answer

To implement a Stripe Checkout flow for an application fee:

When the user clicks “Pay Application Fee,” I first insert a `payment_requests` row with `application_id`, `amount`, and `status = 'pending'`. Then I call `stripe.checkout.sessions.create()` with the fee amount, success/cancel URLs, and include the `payment_request_id` inside `metadata`. I store the generated `session_id` in the database so that webhook events can be matched later.
The user is redirected to `session.url` where Stripe securely processes payment.
A webhook endpoint listens for `checkout.session.completed` and verifies the payload using Stripe’s signature header. On successful payment, I fetch `payment_request_id` from metadata, update its status to `completed`, and store the `payment_intent_id`. I then update the associated application record to mark it as paid (e.g., `payment_status = 'paid'`, `paid_at = now()`), moving the application into the next workflow stage.
For expired or failed sessions, I update the row to `failed` and notify the user to retry.
