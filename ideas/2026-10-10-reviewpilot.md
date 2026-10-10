# ReviewPilot — Automated review collection for small businesses

**One-line pitch:** Turn every happy customer into a 5-star Google review, automatically.

## Problem & target users
Small businesses (restaurants, salons, repair shops, clinics) live or die by Google reviews, but most satisfied customers never leave one — not because they're unhappy, but because nobody asks at the right moment. Target: local service businesses with 5–50 employees, starting with restaurants and beauty salons.

## Tech stack
- **Next.js 14 (App Router)** — full-stack React, fast to ship, great SEO for the marketing site.
- **Supabase** — Postgres + auth + realtime in one; row-level security keeps business data isolated.
- **Tailwind CSS + shadcn/ui** — professional dashboard UI without a designer.
- **Resend** — transactional email for review requests.
- **Twilio** — SMS requests (highest conversion channel).
- **Stripe** — subscriptions.

## Architecture overview
Multi-tenant SaaS. Each business gets: a dashboard (requests, reviews, analytics), a public review landing page (/r/{business-slug}) with smart routing (happy customers go to the Google review link, unhappy ones to a private feedback form), a QR code generator for print, and webhooks for POS integrations. A background worker (Supabase Edge Functions + pg_cron) sends timed follow-ups.

## MVP feature list (prioritized)
1. Business onboarding + Google review link setup
2. Review requests via SMS/email with templates
3. Smart-routing landing page (5-star → Google, below 4 → private feedback)
4. QR code generation for in-store display
5. Dashboard: sent/opened/reviewed funnel + review feed
6. Private feedback inbox with response templates
7. Basic analytics (review velocity, rating trend)

## Data model
- **businesses**: id, name, slug, google_review_url, plan, created_at
- **customers**: id, business_id, name, phone, email, source
- **review_requests**: id, business_id, customer_id, channel (sms/email/qr), status (sent/opened/completed), sent_at
- **reviews**: id, business_id, customer_id, rating (1–5), text, is_public, created_at
- **feedback**: id, business_id, customer_id, message, sentiment, resolved
- **templates**: id, business_id, channel, body

## Seed / initial data
- Business: "Café Aroma" (café), google_review_url https://g.page/cafe-aroma/review, plan: pro
- Business: "Glow Studio" (beauty salon), plan: starter
- 25 customers across both (names, phones in +1-555-01XX format)
- 40 review_requests in mixed statuses; 18 reviews (avg 4.7); 6 private feedback items (2 resolved)
- 3 SMS + 3 email templates (English)

## Build roadmap (4 weeks)
- **Week 1:** Auth, business onboarding, DB schema, landing page skeleton
- **Week 2:** Request sending (SMS/email), smart-routing page, QR codes
- **Week 3:** Dashboard (funnel, feed, analytics), feedback inbox
- **Week 4:** Stripe billing, templates editor, polish, launch

## Monetization
Freemium: free tier (50 requests/mo, 1 location) → Starter $19/mo (500 requests, 3 locations) → Pro $49/mo (unlimited requests, 10 locations, API access). SMS costs passed through with margin. Target: 100 paying businesses in 6 months ≈ $3k MRR.
