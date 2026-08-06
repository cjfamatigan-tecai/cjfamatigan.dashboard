# Aurora Retail — store dashboard

A retail dashboard site with real account login, password recovery, and
role-based access to customer data. Plain HTML/CSS/JS on the front end
(no build step), backed by [Supabase](https://supabase.com) for authentication
and a database with row-level security — so "confidential" isn't just a claim
in the UI, it's enforced by the database itself.

## Roles

Three tiers, each a superset of the one before:

| Role | Can see | Can change |
|---|---|---|
| `customer` | only their own profile and orders | their own profile |
| `support` | every customer's profile and orders (read-only) | their own profile |
| `admin` | everything `support` sees, plus revenue/inventory analytics | any profile, any order, any product, and anyone's role |

Every new signup starts as `customer`. You promote accounts to `support` or
`admin` either from the Admin console's **Customers** table (a role dropdown
per row, admin-only) or with SQL — see below. Admin can also permanently
**delete** a customer's account from that same table (see "Deleting
accounts" below) — it frees up their email for a fresh signup.

## Why a backend at all

A page that only runs in the browser can't actually keep anything secret:
every user can open dev tools and read whatever JavaScript or `localStorage`
holds. Real login and real confidentiality need a server-side database with
access rules the browser can't override. Supabase gives you that (hosted
Postgres + auth) without you having to run your own server.

## Files

- `index.html` — sign in
- `signup.html` — create an account
- `forgot-password.html` — "can't sign in?" recovery request
- `reset-password.html` — set a new password (reached via the emailed link)
- `dashboard.html` — customer view: own orders, loyalty points, catalog, account settings
- `support.html` — **support + admin** view: read-only lookup of every customer's contact details and order history
- `admin.html` — **admin-only** view: store-wide revenue, inventory, every customer's data, and role management
- `css/styles.css` — the shared "Aurora" design system (dark by default, light mode toggle, top-left)
- `js/` — `supabaseClient.js` (connection + shared helpers + role helpers), `theme.js`, `auth.js`, `dashboard.js`, `support.js`, `admin.js`
- `supabase/schema.sql` — the database tables and security policies
- `supabase/functions/delete-user/index.ts` — Edge Function that fully deletes a customer's login (admin-only, deploy via the Supabase dashboard)

## Set up your own backend (~5 minutes, free tier)

1. Create a free project at [supabase.com](https://supabase.com).
2. In your new project, go to **SQL Editor → New query**, paste in the full
   contents of `supabase/schema.sql`, and run it. This creates the
   `profiles`, `orders`, and `products` tables, seeds a few sample products,
   and sets up the row-level security rules described below.
3. Go to **Project Settings → API**. Copy the **Project URL** and the
   **anon public** key.
4. Open `js/supabaseClient.js` and paste them in at the top:
   ```js
   const SUPABASE_URL = "https://your-project.supabase.co";
   const SUPABASE_ANON_KEY = "your-anon-public-key";
   ```
5. Serve the folder with any static file server (opening `index.html`
   directly with `file://` will break the password-reset email link):
   ```
   python3 -m http.server 8000
   ```
   then visit `http://localhost:8000`.
6. In **Authentication → URL Configuration**, set the **Site URL** to
   wherever you're hosting the site (e.g. `http://localhost:8000` while
   testing, your real domain once you deploy).

## Make yourself the admin

1. Sign up through `signup.html` with your own email, exactly like a
   customer would.
2. Confirm your email (Supabase sends a confirmation link by default).
3. Back in the Supabase SQL Editor, run:
   ```sql
   update public.profiles set role = 'admin' where email = 'you@example.com';
   ```
4. Sign in again. You'll land on `admin.html` instead of the customer
   dashboard, and can now see every customer's orders, contact details, and
   spend, plus change anyone's role — the customer accounts you haven't
   promoted still see only their own data, both in the UI and if they
   inspect network requests directly.

From then on you don't need SQL to promote anyone else — as admin, open the
**Customers** table in `admin.html` and change a row's Role dropdown to
`support` or `admin` directly.

## How the confidentiality actually works

This is enforced in `supabase/schema.sql`, not by hiding buttons in the UI:

- Every table (`profiles`, `orders`, `products`) has **row-level security**
  turned on.
- A signed-in customer's queries are automatically filtered to `auth.uid() =
  id` (or `= user_id`) — the database itself refuses to return anyone else's
  row, regardless of what the page's JavaScript asks for.
- `is_staff()` and `is_admin()` database functions check the caller's own
  `role` column. Policies use `... OR is_staff()` (read access for
  `support`/`admin`) or `... OR is_admin()` (write access for `admin` only)
  so a plain `customer` row never reads or writes anyone else's data.
- A trigger blocks anyone from changing their own (or anyone else's) `role`
  through the app unless they're already an admin — you can only promote the
  very first account by running SQL yourself, as in the step above.

## Deleting accounts

The Admin console's Customers table has a **Delete** button per row (not
shown on your own row — you can't delete yourself from here). It fully
removes that person's login, not just their visible data, so they can sign
up again afterward with the same email if needed.

This needs one piece of one-time setup, because deleting a login
(`auth.users`) requires Supabase's **service role key** — a secret that
must never be shipped to the browser (unlike the anon key, it bypasses row
level security completely). It lives in a small server-side function
instead, hosted by Supabase itself:

1. In your Supabase project, go to **Edge Functions** in the left sidebar.
2. Create a new function named exactly `delete-user`.
3. Replace its contents with the code in `supabase/functions/delete-user/index.ts`
   in this project, then deploy it from the dashboard editor.
4. That's it — `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and
   `SUPABASE_SERVICE_ROLE_KEY` are provided to every Edge Function
   automatically, so there's no key to copy/paste anywhere.

The function re-checks server-side that the caller is really an admin
before deleting anything — the Delete button in the UI is a convenience,
not the security boundary; someone spoofing the request without an admin
session gets rejected by the function itself.

## Trouble signing in

The "Forgot password?" / "Trouble signing in?" links walk a customer through
account recovery: they enter their email, get a reset link (Supabase emails
it — no email in this flow ever confirms or denies whether an account
exists, which prevents someone from using it to discover registered emails),
click through to `reset-password.html`, and set a new password. If a login
attempt fails because the email isn't verified yet, the sign-in page offers
to resend the confirmation email inline.

## Before you launch for real

- Search for `[bracket]`-style placeholders — there are none by default, but
  swap the sample products in `supabase/schema.sql` for your real catalog.
- Consider turning on Supabase's leaked-password protection and adjusting
  session length under **Authentication → Settings**.
- The free Supabase tier is fine for testing; check current limits before
  pointing real customer traffic at it.
