# ContractKit — Contracts & e-Signatures for Freelancers, in Minutes

**One-line pitch:** ContractKit turns a freelance agreement into a signed, milestone-tracked contract in under five minutes — no lawyer, no PDF juggling, no chasing clients for a signature.

## Problem & Target Users

Freelancers lose money and sleep to informal agreements: work agreed over chat, scope creep with no paper trail, late payments with nothing enforceable to point at, and clients who vanish after delivery. Existing e-signature tools (DocuSign, PandaDoc) are built for enterprises — expensive, heavy, and full of features a solo freelancer never touches.

**Target users:**
- Solo freelancers and small studios (designers, developers, writers, consultants) who close 2–10 projects a month.
- Micro-agencies (2–5 people) that need a repeatable, branded contracting flow without hiring legal help.
- Clients of freelancers, who get a clean, mobile-friendly signing experience with no account required.

## Tech Stack

- **Next.js 15 (App Router, TypeScript)** — one codebase for the marketing site, dashboard, and signing pages; great SEO for the template gallery.
- **Tailwind CSS + shadcn/ui** — fast, consistent UI; the contract editor and signing flow need polish, not custom CSS.
- **PostgreSQL + Prisma** — relational data (contracts, clauses, signatures, milestones) with type-safe queries.
- **NextAuth (email magic links)** — freelancers sign in without passwords; clients never need an account at all.
- **Stripe** — paid plans plus optional escrow-style milestone payments (client funds a milestone, released on delivery).
- **Resend** — transactional email: signature requests, reminders, and signed-copy delivery.
- **Vercel** — hosting with preview deployments; fits the Next.js stack perfectly.
- **PDF-lib + html-to-pdf** — generate the final signed PDF with an embedded signature certificate page.

## Architecture Overview

```
Client (browser)
  ├─ Marketing site + template gallery (static, ISR)
  ├─ Dashboard (authenticated, freelancer workspace)
  └─ Public signing page /sign/[token] (no auth, token-gated)
        │
        ▼
Next.js API routes (server actions + route handlers)
  ├─ Auth (NextAuth, magic link)
  ├─ Contracts service (CRUD, clause engine, versioning)
  ├─ Signing service (token generation, audit trail, PDF render)
  ├─ Payments service (Stripe: subscriptions + milestone escrow)
  └─ Email service (Resend: requests, reminders, delivery)
        │
        ▼
PostgreSQL (Prisma) ── Stripe ── Resend ── S3-compatible storage (signed PDFs)
```

Key design decisions:
- **Token-based signing, not accounts:** a client opens a unique signed URL; the token is single-use per signing step and expires after 14 days. Zero friction for the person paying you.
- **Immutable contract versions:** every edit after sending creates a new version; the audit trail records who viewed, signed, and when (IP + timestamp), so the signed PDF is defensible.
- **Clause library as data, not text:** contracts are assembled from typed clauses (scope, payment, revision, IP, termination), so templates are composable and the editor can offer plain-language explanations per clause.

## MVP Feature List (Prioritized)

**P0 — must ship:**
1. Magic-link signup and freelancer workspace.
2. Template gallery: 6 starter templates (fixed-price project, hourly, retainer, NDA, design services, consulting).
3. Contract builder: pick a template, fill a guided form (parties, scope, fee, milestones, timeline), live preview.
4. Send for signature: email the client a token link; client signs by typing/drawing a signature on any device.
5. Signed PDF generation with audit-trail certificate page; both parties get a copy by email.
6. Contract list with statuses (draft / sent / viewed / signed / completed).

**P1 — first month after launch:**
7. Milestone payments via Stripe: client funds a milestone, freelancer marks it delivered, funds release.
8. Automatic reminders for unsigned contracts (day 3, day 7).
9. Clause library: add, edit, and reuse custom clauses; plain-language explainer per clause.
10. Branding: freelancer logo, colors, and custom signing-page header on paid plans.

**P2 — later:**
11. Contract renewal and amendment flow (new version, re-sign).
12. Multi-language templates (starting with Persian + English — a real gap for Iranian freelancers working with foreign clients).
13. Analytics: average time-to-sign, late-payment rate, most-used templates.

## Data Model

**User** — `id, name, email, businessName, logoUrl, plan (free|pro), stripeCustomerId, createdAt`
**Client** — `id, userId, name, email, company, country, createdAt` (a freelancer's client roster)
**Template** — `id, name, category, description, isPublic, clausesJson, createdBy (system|userId)`
**Contract** — `id, userId, clientId, templateId, title, status (draft|sent|viewed|signed|completed|voided), currency, totalAmount, startDate, endDate, version, signingToken, tokenExpiresAt, createdAt, updatedAt`
**Clause** — `id, contractId, type (scope|payment|revision|ip|termination|custom), title, body, plainLanguageNote, order`
**Milestone** — `id, contractId, title, amount, dueDate, status (pending|funded|delivered|released), stripePaymentIntentId`
**Signature** — `id, contractId, signerRole (freelancer|client), signerName, signerEmail, signatureImageUrl, signedAt, ipAddress, userAgent`
**AuditEvent** — `id, contractId, eventType (created|sent|viewed|signed|reminded|voided), actor, metadataJson, createdAt`

## Seed / Initial Data

**Templates (6):**
1. Fixed-Price Project — scope, 2 revision rounds, 50% upfront / 50% on delivery, IP transfers on final payment.
2. Hourly Engagement — $45/hr, weekly invoicing, 14-day payment terms, 10h/week cap.
3. Monthly Retainer — $1,500/mo, 20h included, rollover up to 5h, 30-day notice to cancel.
4. NDA (Mutual) — 2-year confidentiality, standard exclusions, governing law selectable.
5. Design Services — 3 revision rounds, source files on final payment, portfolio-use clause.
6. Consulting — day rate $600, expenses reimbursed, deliverables defined per statement of work.

**Sample contract:** "Website Redesign for Bright Studio" — freelancer Sara Ahmadi → client Bright Studio Ltd., fixed-price $3,200, milestones: $1,600 upfront (funded), $1,600 on launch; 3 revision rounds; delivery 2026-12-15; status: signed.

**Sample client records:** Bright Studio Ltd. (hello@brightstudio.co), Northwind Café (owner@northwind.cafe), TechFlow Inc. (ops@techflow.io).

**Sample clauses:** scope ("Redesign of 8 pages: home, about, services ×4, contact, blog index"), payment ("50% upfront, 50% within 7 days of launch"), revision ("3 rounds included; extra rounds at $120 each"), IP ("All IP transfers to client upon final payment").

## Build Roadmap (4 Weeks)

**Week 1 — Foundation:** Next.js + Tailwind + shadcn scaffold; Prisma schema and migrations; NextAuth magic-link auth; dashboard shell; contract list + status badges.
**Week 2 — Builder:** template gallery with 6 templates; guided contract form; live preview; clause engine (typed clauses, add/remove/reorder); save as draft.
**Week 3 — Signing:** token-based public signing page; typed/drawn signatures; audit trail (view/sign events with IP + timestamp); signed PDF generation with certificate page; Resend emails (request, signed copy).
**Week 4 — Polish & Launch:** Stripe subscriptions (free: 3 contracts/mo; pro $12/mo unlimited); milestone payments (P1, behind a flag if time is short); reminders; landing page + SEO for the template gallery; deploy to Vercel; launch on X and Product Hunt.

## Monetization Angle

Freemium with a clear upgrade trigger: free users get 3 contracts per month with ContractKit branding on the signing page; Pro at $12/month removes the limit and branding, adds custom clauses, reminders, and milestone payments. The real revenue lever is milestone escrow — a 2.5% fee on milestone payments processed through Stripe turns ContractKit from a document tool into the place money actually moves, and freelancers who get paid through it rarely leave. A template marketplace (paid, lawyer-reviewed templates, revenue share with the author) is a natural second stream once the user base exists.
