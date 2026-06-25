# Elite Trading App — Production Readiness Plan

Date: 2026-06-25

## Executive summary

The app is a useful prototype, but it is **not production-ready** for real crypto trades, customer funds, payment proofs, BVN/NIN, ID documents, or admin operations.

The main issue is not that the app is built with plain HTML/JavaScript instead of React. The main issue is that the browser currently has too much authority. In a financial app, the browser should collect input and display results, but it should not be trusted to decide rates, order amounts, order status, KYC status, admin permissions, referral rewards, or access to sensitive files.

The production goal should be:

```text
Browser = display and input only
Backend/database = authority
Admin actions = server-verified
Money/rates/status = server-controlled
KYC files = private and audited
```

Until the security and backend architecture are fixed, the app should not process live transactions or collect real KYC data.

---

## Current status

The current implementation has made some improvements, including better HTTP error handling in some Supabase helper functions and safer rendering in part of the admin order detail flow.

However, the core production blockers remain:

- Users may still be able to update sensitive profile fields if the embedded Row Level Security policies are applied as written.
- Financial values are still calculated and submitted from the browser.
- KYC documents and payment proofs are still handled in a way that can expose sensitive files.
- Some user-controlled database values are still rendered with `innerHTML`, creating stored XSS risk.
- The database setup is not fully version-controlled in proper migration files.
- Admin privileges are still partly treated as a frontend concern.
- There is no clear audit trail for sensitive admin actions.

---

# Phase 1 — Freeze production usage

## 1. Do not use this for live customer trades yet

Until the fixes below are completed, the app should not process:

- real customer money;
- crypto releases;
- real payment receipts;
- BVN;
- NIN;
- ID documents;
- selfies;
- live KYC approvals.

## 2. Rotate exposed credentials where appropriate

Supabase anon keys are normally public, but the app currently depends heavily on correct database policies. After the RLS policies are redesigned, rotate/regenerate exposed credentials where appropriate and remove unused keys.

## 3. Disable public KYC and proof access

KYC documents and payment receipts should not be publicly accessible.

Required changes:

- Make storage buckets private.
- Do not store public URLs for KYC files.
- Store private object paths instead.
- Generate short-lived signed URLs only when an authorized admin needs to view a file.

---

# Phase 2 — Fix the database foundation

## 1. Move schema into migration files

The database schema should not live only inside `index.html`.

Create version-controlled migration files, for example:

```text
supabase/migrations/
  001_create_profiles.sql
  002_create_orders.sql
  003_create_settings.sql
  004_create_audit_logs.sql
  005_storage_policies.sql
  006_rls_policies.sql
```

The repo should contain the exact database structure required to run the app.

## 2. Create the real schema used by the app

The app currently uses fields that are not fully represented in the embedded SQL setup.

Recommended minimum schema:

### `profiles`

```sql
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text not null,
  email text not null unique,
  role text not null default 'user',
  kyc_status text not null default 'pending',
  kyc_bvn text,
  kyc_nin text,
  kyc_id_path text,
  kyc_selfie_path text,
  referral_code text unique,
  referred_by text,
  referral_credit numeric not null default 0,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

Important: regular users must not be able to directly update `role`, `kyc_status`, `referral_credit`, `referral_code`, or other sensitive fields.

### `orders`

```sql
create table orders (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references profiles(id),
  type text not null,
  coin text not null,
  crypto_amount numeric not null,
  ngn_amount numeric not null,
  rate numeric not null,
  status text not null default 'pending',
  wallet_address text,
  bank_name text,
  account_number text,
  proof_path text,
  referral_processed boolean not null default false,
  quoted_at timestamptz,
  expires_at timestamptz,
  completed_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

### `settings`

```sql
create table settings (
  key text primary key,
  value text not null,
  updated_at timestamptz not null default now()
);
```

Example setting keys:

- `bank_name`
- `account_number`
- `account_name`
- `spread`

### `audit_logs`

```sql
create table audit_logs (
  id uuid primary key default gen_random_uuid(),
  actor_id uuid not null references profiles(id),
  action text not null,
  target_table text,
  target_id uuid,
  metadata jsonb,
  created_at timestamptz not null default now()
);
```

## 3. Add database constraints

The database should reject bad data even if someone bypasses the UI.

Examples:

```sql
alter table profiles
add constraint profiles_role_check
check (role in ('user', 'admin'));

alter table profiles
add constraint profiles_kyc_status_check
check (kyc_status in ('pending', 'approved', 'rejected'));

alter table orders
add constraint orders_type_check
check (type in ('Buy', 'Sell'));

alter table orders
add constraint orders_status_check
check (status in (
  'pending',
  'payment_submitted',
  'payment_confirmed',
  'crypto_released',
  'completed',
  'rejected',
  'cancelled',
  'expired'
));

alter table orders
add constraint orders_coin_check
check (coin in ('USDT', 'BTC', 'ETH', 'BNB'));

alter table orders
add constraint orders_positive_amount_check
check (crypto_amount > 0 and ngn_amount > 0 and rate > 0);
```

---

# Phase 3 — Redesign RLS properly

## Current problem

The current embedded policy includes a broad rule like:

```sql
CREATE POLICY "Users update own profile"
  ON profiles FOR UPDATE USING (auth.uid() = id);
```

This appears safe at first, but it is not safe if users can update sensitive columns such as:

- `role`;
- `kyc_status`;
- `referral_credit`;
- `referral_code`;
- `referred_by`;
- KYC fields.

A user should never be able to approve their own KYC, make themselves an admin, or credit themselves referral money.

## Required RLS design

Users may:

- read their own profile;
- update only safe profile fields;
- submit KYC through a controlled server-side function;
- create orders only through a controlled server-side function;
- read their own orders.

Users must not directly update:

- `role`;
- `kyc_status`;
- `referral_credit`;
- `referral_code`;
- `referred_by`;
- order `status`;
- order `rate`;
- order `crypto_amount`;
- order `ngn_amount`;
- admin settings.

Admins may:

- view all users;
- view all orders;
- approve/reject KYC;
- confirm/reject orders;
- update bank details;
- update spread;
- view private KYC/proof files through controlled access.

Admin permissions must be enforced by the database/backend, not just by checking an email address in JavaScript.

---

# Phase 4 — Move financial logic server-side

## Current problem

The app calculates and submits rates, order amounts, spread, and order status from the browser.

That allows a malicious user to bypass the UI and submit manipulated data directly to Supabase.

Example malicious payload:

```json
{
  "type": "Buy",
  "coin": "USDT",
  "ngn_amount": 100000,
  "rate": 1,
  "crypto_amount": 100000,
  "status": "completed"
}
```

Even if the UI does not allow this, direct API calls can.

## Required fix

Order creation must happen server-side.

The browser should send only the user's intent, for example:

```json
{
  "type": "Buy",
  "coin": "USDT",
  "ngn_amount": 100000,
  "wallet_address": "..."
}
```

Then the backend should:

1. Verify the authenticated user.
2. Verify the user's KYC status.
3. Fetch the current market price.
4. Fetch the current spread.
5. Calculate the official rate.
6. Calculate the crypto amount.
7. Set the order status to the correct initial status.
8. Store `quoted_at` and `expires_at`.
9. Insert the order.
10. Return the official order result.

The client should never decide:

- final rate;
- crypto amount;
- order status;
- referral reward;
- whether KYC is approved;
- whether payment is confirmed.

---

# Phase 5 — Add backend or Supabase Edge Functions

At minimum, create controlled functions for:

```text
createOrder()
submitKYC()
approveKYC()
rejectKYC()
confirmOrder()
rejectOrder()
updateBankSettings()
updateSpread()
processReferralReward()
getSignedKycUrl()
getSignedProofUrl()
```

These functions should:

- verify the authenticated user;
- check whether the user is an admin where required;
- validate all input;
- perform sensitive writes server-side;
- write audit logs;
- return clear success/error responses.

---

# Phase 6 — Lock down admin actions

## Current problem

Admin status is partly determined in frontend code using a hardcoded admin email.

Frontend checks are useful for showing or hiding UI, but they are not security.

## Required fix

Admin status must be enforced server-side.

Admin-only actions:

- view all orders;
- confirm payment;
- reject order;
- approve KYC;
- reject KYC;
- change bank details;
- change spread;
- export records;
- view KYC documents;
- view payment proofs.

All admin actions should require a verified admin role from the database or custom JWT claims.

## Add admin audit logs

Log actions such as:

```text
ORDER_CONFIRMED
ORDER_REJECTED
KYC_APPROVED
KYC_REJECTED
SPREAD_CHANGED
BANK_DETAILS_CHANGED
REFERRAL_REWARD_GRANTED
SIGNED_KYC_URL_CREATED
SIGNED_PROOF_URL_CREATED
```

This is very important for a financial business.

---

# Phase 7 — Fix KYC properly

## Current problem

The app collects BVN, NIN, ID document, and selfie, but it does not truly verify them. It also stores sensitive data too casually.

## Required fix

For production:

- Store KYC files in private storage only.
- Store private file paths, not public URLs.
- Restrict access to authorized admins only.
- Generate short-lived signed URLs when needed.
- Add audit logs when an admin views or approves KYC.
- Consider encrypting BVN/NIN or storing only masked versions.
- Decide whether the business legally needs to store BVN/NIN at all.

Recommended:

- Use a proper KYC provider if possible.
- If manual review is still used, clearly treat it as manual review, not automated verification.
- Confirm legal/compliance requirements before collecting BVN/NIN.

---

# Phase 8 — Fix stored XSS risk

## Current problem

Some database values are still rendered with `innerHTML`.

That means a malicious user could submit a name, email, wallet address, order field, status, or other value containing HTML/JavaScript and potentially compromise the admin screen.

## Required fix

Use `textContent` for user-controlled values.

Avoid:

```js
element.innerHTML = user.full_name;
```

Use:

```js
element.textContent = user.full_name;
```

Best rule:

- `innerHTML` is allowed only for hardcoded UI markup.
- Anything from the database, user input, URL parameters, or external APIs must be inserted with `textContent` or escaped safely.

---

# Phase 9 — Fix file uploads

## Required changes

For payment proof uploads:

- validate file type;
- validate file size;
- check upload success with `response.ok`;
- store private file path;
- use signed URLs for admin viewing;
- prevent users from overwriting other users' files.

For KYC uploads:

- use private bucket only;
- apply strict size/type validation;
- do not return public URLs;
- allow admin-only access;
- audit access.

The upload helper must check whether the upload succeeded before returning a path.

---

# Phase 10 — Add operational safety

For a crypto/P2P business, technical security is only part of production readiness. The workflow should also protect the business operationally.

## Add clearer transaction states

Instead of only:

```text
pending
completed
rejected
```

Use clearer states:

```text
pending_payment
payment_submitted
payment_confirmed
crypto_released
completed
rejected
cancelled
expired
```

This makes disputes easier to handle.

## Add server-side rate expiry

The app currently has a client-side timer, but the backend/database must enforce expiry.

Each order should store:

```text
quoted_at
expires_at
```

The backend should reject expired quotes or require a fresh quote.

## Add idempotency

Prevent duplicate order submissions from double-clicks, retries, or network issues.

Use an idempotency key when creating orders.

## Add manual reconciliation fields

Admin should be able to record:

- bank payment reference;
- amount received;
- crypto transaction hash;
- admin notes;
- who confirmed the order;
- when it was confirmed.

---

# Phase 11 — Authentication hardening

Minimum production requirements:

- strong password rules;
- email confirmation;
- password reset flow;
- mandatory MFA for admins;
- session expiry;
- no hardcoded frontend-only admin trust;
- no ability for normal users to self-assign admin role.

Admin accounts should require MFA.

---

# Phase 12 — Add testing

## Database/RLS tests

Verify:

- normal user cannot become admin;
- normal user cannot approve their own KYC;
- normal user cannot update order status;
- normal user cannot view another user's orders;
- normal user cannot view another user's KYC documents;
- normal user cannot update settings;
- admin can approve/reject KYC;
- admin can confirm/reject orders.

## Backend tests

Verify:

- order rate is calculated server-side;
- fake client-submitted rate is ignored;
- expired quote is rejected;
- referral reward is applied once only;
- upload failure does not create fake successful proof;
- invalid order data is rejected;
- non-admin users cannot call admin functions.

## UI tests

Verify:

- registration;
- login;
- KYC submission;
- buy flow;
- sell flow;
- admin approval;
- admin confirm/reject order;
- records export.

---

# Phase 13 — Deployment readiness

Before launch, the repo should contain:

```text
index.html or frontend app
supabase/migrations/
supabase/functions/
.env.example
README.md
SECURITY.md
deployment instructions
test instructions
```

Also add:

- staging environment;
- production environment;
- version-controlled database migrations;
- backups;
- logging;
- error monitoring;
- admin activity logs;
- rollback procedure.

---

# Recommended implementation order

Do not try to fix everything randomly. Fix it in this order:

1. Create proper Supabase migration files.
2. Redesign tables and constraints.
3. Replace unsafe RLS policies.
4. Create backend/Supabase Edge Functions for sensitive actions.
5. Make storage buckets private.
6. Remove client-side authority over financial/admin fields.
7. Fix all `innerHTML` user-data rendering.
8. Add audit logs.
9. Add authentication hardening, especially admin MFA.
10. Add tests.
11. Run a staging pilot with fake trades.
12. Launch only after the security checklist passes.

---

# Minimum launch checklist

The app should not launch until all of these are true:

- [ ] Users cannot update their own `role`.
- [ ] Users cannot approve their own KYC.
- [ ] Users cannot update their own referral credit.
- [ ] Users cannot directly set order status.
- [ ] Users cannot submit final rates or crypto amounts directly.
- [ ] Order rates are calculated server-side.
- [ ] KYC files are private.
- [ ] Payment proofs are private or access-controlled.
- [ ] Admin actions are enforced server-side.
- [ ] Admin MFA is enabled.
- [ ] Audit logs exist for sensitive actions.
- [ ] All user-controlled rendering avoids unsafe `innerHTML`.
- [ ] Database schema is in migrations.
- [ ] RLS policies have been tested.
- [ ] Backend/Edge Function tests exist.
- [ ] Staging environment has been tested with fake trades.

---

# Final recommendation

The app should be treated as a prototype until the backend/security model is fixed.

It does not need to be rewritten just because it is not React. A plain HTML/JavaScript frontend can still be acceptable for a small internal or mobile-first tool. But a crypto/P2P trading app must have a trustworthy backend and locked-down database rules.

The most important production change is this:

```text
Do not trust the browser with money, identity, admin powers, or final transaction state.
```

