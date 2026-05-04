# Zany's Play World — engineering case study

A walk-through of the production booking and operations platform I built (and currently maintain) for an indoor play center in Newnan, Georgia. The business opened November 7, 2025 and runs on this code every day.

**Live:** [book.zanysplayworld.com](https://book.zanysplayworld.com)

The application repository is private (it's a closed business, not an OSS project). This repo exists so engineering hiring managers can see the architecture, decisions, and scope without being asked to "trust me, bro."

## TL;DR

| Metric | Value |
|---|---|
| Lines of TypeScript | ~80,700 |
| API endpoints | 161 |
| Postgres tables | 50 |
| Production cron jobs | 13 |
| Built by | One person, end to end |
| Time to first live transaction | 11 weeks |
| Uptime since launch | Has not gone down |

Real customers, real refunds when something breaks, no test environment to hide in.

## What the platform actually does

It runs the whole business — not just the storefront.

**Customer-facing**
- Birthday party booking flow (details → waiver → Stripe payment → confirmation, with double-booking protection on a `[partyDate, timeSlot]` unique constraint)
- Walk-in / open-play registration via QR code at the front desk
- Membership signups with recurring Stripe billing and check-in tracking
- Gift card purchase, balance check, and redemption
- Event/camp registration (Easter, summer camp, spring break, field trips)
- Pre-party pizza ordering via tokenized link to a partner restaurant (Old Chicago)
- Post-party survey with sentiment analysis
- Standalone waiver signing with reusable email-keyed records (1-year expiry)

**Operations**
- Admin dashboard with KPI cards, revenue streams, 12-month trends, projections
- Weekly schedule grid (click-to-edit bookings)
- Customer database with full booking history
- Subscription management and churn dashboards
- Stripe Terminal (in-person card readers) with diagnostics page
- Cash drawer open/close flow with denomination counts and reconciliation
- Employee scheduling, time-off requests, push-notification shift reminders
- Marketing campaign builder with HTML templates and customer segmentation
- Inventory, gallery, signage slides, cleaning checklists

## Tech stack

- **Next.js 16** — App Router, server/client components, edge runtime where it matters
- **TypeScript** end to end
- **Postgres via Supabase** with **Prisma** (50 models, CUID IDs)
- **Stripe** — Checkout, Payment Intents, Terminal SDK, Connect, webhooks (API v2025-02-24)
- **Resend** for transactional email; HTML templates with PDF attachments via **pdfkit**
- **Twilio** for SMS (payment links, shift reminders, marketing campaigns)
- **Web Push** for staff notifications (Service Worker + VAPID keys)
- **Upstash Redis** for rate limiting at the edge
- **Tailwind CSS 3.4** + **shadcn/ui** + **radix-ui** primitives
- **Zustand** for client state, **React Hook Form + Zod** for form validation
- **Vercel** for hosting (with custom domain, CSP headers, function tracing for pdfkit)

## Notable engineering decisions

**Edge-compatible HMAC session auth.** I didn't want to ship a heavyweight auth library for what is fundamentally a single-tenant admin login. Implemented signed session tokens using only the Web Crypto API (no Node `crypto` module), so the auth check runs in `middleware.ts` at the edge with sub-millisecond overhead. ~120 lines, no dependencies, four subjects (`admin`, `reports`, `staff`, `cfo`).

**Stripe Terminal for in-person sales.** Every guide on the internet shows Stripe Checkout. Almost no one writes about Stripe Terminal — the SDK for physical card readers. Built the connection-handling, payment-intent-creation, and reader-discovery flows from the SDK docs and the source. The diagnostics page surfaces reader status, last error, and a "send test charge" button so staff can self-troubleshoot.

**13 cron jobs for the back-of-house.** The platform runs without me. Every cron is a small route under `/api/cron/*` with a single responsibility: expire pending unpaid bookings after 7 days, send post-party surveys, sync Stripe subscription status, send pizza-order reminders 2 days before parties, "we miss you" emails after 2 weeks of membership inactivity, repeat-visitor membership pitches on the 3rd / 6th / 9th visit, employee shift-reminder push notifications 1 hour pre-shift, scheduled marketing email delivery, Google review request emails, daily Supabase backup verification.

**pdfkit on Vercel.** Generating party contract PDFs server-side meant fighting Vercel's bundler. Required adding pdfkit to `serverExternalPackages`, configuring file tracing for the font assets, and reducing the function timeout headroom. Took an afternoon to get right; runs reliably now.

**Schema with reusable waivers.** Parents sign a digital waiver once per year, indexed by email — subsequent bookings for any of their children pull from `StoredWaiver` and `StoredWaiverChild` records. Cascade rules: deleting a Customer cascades through Bookings → Waiver, Payments, PizzaOrder. `Booking.partyDate` is a full DateTime so we can filter with `gte`/`lt`; `Availability.date` is `@db.Date` because availability is per-day. Decimal fields return `Prisma.Decimal` and require `.toNumber()` — bit me once, never again.

**Stripe webhook handler at 1,154 lines.** One route, six event types, all idempotent on Stripe's event ID. Handles `checkout.session.completed` (mark booking paid, activate gift cards), `payment_intent.succeeded` (update payment status), `customer.subscription.deleted` (mark canceled), `charge.refunded` (handle refunds), and a couple more. The length is the cost of correctness — every branch is doing real work.

## What's not in this repo

The application source is private. If you'd like a deeper look — code samples, an architecture call, a demo of a specific subsystem — reach out:

- Email: clint@technolo.co
- LinkedIn: [linkedin.com/in/clintonphillips](https://www.linkedin.com/in/clintonphillips)
- GitHub: [github.com/0xdeadd](https://github.com/0xdeadd)

Happy to walk through any part of it.
