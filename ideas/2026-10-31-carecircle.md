# CareCircle — Medication & Appointment Companion for Elderly Patients and Their Families

**One-line pitch:** A simple, calm app that helps elderly patients take the right medicine at the right time — and lets their family see, at a glance, that they did.

## Problem & target users

Medication mistakes are one of the most common preventable health problems for older adults: multiple prescriptions, confusing schedules ("twice daily, with food"), refills that run out unnoticed, and appointments that slip through the cracks. The patient often struggles with small text and complicated apps, while adult children worry from another city and only find out about a missed dose after a crisis.

**Target users:**

- **Primary — elderly patients (65+):** people managing 2–10 daily medications who need large, clear reminders and a one-tap "I took it" action, not another complicated health dashboard.
- **Secondary — family caregivers:** adult children or relatives who want a gentle, non-intrusive view of adherence, refill alerts, and upcoming appointments — without calling every day to check.
- **Tertiary — small clinics and pharmacies (later phase):** practices that want to share a patient's schedule and receive adherence summaries instead of phone-tagging families.

## Tech stack

- **Next.js (App Router, TypeScript) —** one codebase for the marketing site, patient web app, and caregiver dashboard, with server rendering for fast loads on older phones.
- **Tailwind CSS + shadcn/ui —** rapid, accessible UI work with large touch targets and high-contrast themes; the design system is the product's main differentiator for elderly users.
- **PostgreSQL via Supabase —** managed Postgres with row-level security, so each family's health data is isolated by default and the solo developer never runs a database server.
- **Supabase Auth —** email + magic-link sign-in (no passwords to forget) and a "caregiver invite" flow built on its invitation model.
- **Prisma —** type-safe data access that keeps the medication-schedule logic readable and testable as rules grow.
- **Web Push API + Twilio SMS (fallback) —** push reminders for smartphones, SMS for patients on basic phones — reminders must reach the person, not just the app.
- **Vercel —** zero-config hosting, preview deploys, and cron jobs for the reminder scheduler.
- **Vitest + Playwright —** unit tests for the scheduling engine (the safety-critical part) and end-to-end tests for the "mark as taken" flow.

## Architecture overview

The app is split into three surfaces on one Next.js codebase:

1. **Patient app (mobile-first web app):** a single "Today" screen — big cards for each dose due, one large "Taken" button per card, and a "taken / missed / upcoming" strip. No menus more than one tap deep.
2. **Caregiver dashboard (web):** a weekly adherence view per patient, refill warnings, appointment list, and a quiet alert feed (only real exceptions: missed critical dose, refill below 7 days, appointment tomorrow).
3. **Reminder service (scheduled worker):** a cron job that runs every 5 minutes, computes due doses from each patient's schedule (including "with food" windows and variable cycles like "every other day"), and dispatches push or SMS reminders. Delivery attempts and confirmations are logged so a missed reminder is visible, not silent.

Data flows one way for safety: the scheduling engine is the single source of truth for "what is due when"; the patient app only records intake events, and the dashboard only reads aggregates. A separate audit log records every medication change (who changed what, when) so families and clinicians can trust the history.

## MVP feature list (prioritized)

1. **Medication schedule setup** — add a medicine (name, dose, form, times per day, days of week, start/end date, "with food" flag) with a large-type, step-by-step form.
2. **Today view with one-tap intake logging** — the patient sees only today's doses, taps "Taken," and can undo within 10 minutes.
3. **Smart reminders** — push notification at dose time, one gentle repeat after 30 minutes if unconfirmed, then a quiet alert to the caregiver (no alarm spam).
4. **Caregiver invite & dashboard** — invite by email or link; dashboard shows this week's adherence percentage, missed doses, and next refill dates.
5. **Refill tracking** — quantity on hand per medicine, auto-computed "runs out on" date, alert at 7 days remaining.
6. **Appointment list** — doctor, date/time, clinic, notes; a reminder the day before and the morning of.
7. **Weekly adherence report** — a simple emailed summary every Sunday ("Mum took 19 of 20 doses this week") that families actually read.
8. **Accessibility mode** — large-text theme, high contrast, and Persian/English language toggle (the developer's own bilingual audience is a natural first market).

## Data model

- **User** — id, name, email, phone, role (patient | caregiver | clinician), timezone, language, largeTextEnabled, createdAt.
- **CareCircle (family group)** — id, name (e.g. "Mum's care"), createdBy, createdAt. Links patients and their caregivers.
- **Membership** — id, circleId, userId, role (patient | caregiver), notificationPreference (push | sms | both), alertThresholdMinutes.
- **Medication** — id, circleId, patientId, name, strength (e.g. "10 mg"), form (tablet | capsule | liquid | injection | other), instructions, withFood (boolean), colorCode, active (boolean), createdBy, createdAt.
- **Schedule** — id, medicationId, timesOfDay (array, e.g. ["08:00", "20:00"]), daysOfWeek (array, 0–6, or null for daily), startDate, endDate (nullable), intervalDays (nullable, for "every other day").
- **IntakeEvent** — id, scheduleId, medicationId, patientId, scheduledAt, status (taken | missed | skipped), loggedAt, loggedBy (patient | caregiver), note (nullable).
- **Inventory** — id, medicationId, quantityOnHand, refillThresholdDays, lastRefillDate, pharmacyName (nullable).
- **Appointment** — id, circleId, patientId, doctorName, clinicName, address (nullable), appointmentAt, reason, notes, reminderSent (boolean).
- **ReminderLog** — id, intakeEventId (nullable), channel (push | sms), sentAt, delivered (boolean), failureReason (nullable).
- **AuditLog** — id, circleId, actorId, entityType, entityId, action (created | updated | deactivated), changesJson, createdAt.

## Seed / initial data

A demo family, "The Maleki family circle":

- **Users:** Maryam (patient, 72, timezone Asia/Tehran, large text on); Sara (caregiver, daughter, SMS + push); Dr. Ahmadi (clinician, read-only).
- **Medications:**
  - Amlodipine 5 mg, tablet, 1× daily at 08:00, with food, 28 tablets on hand, pharmacy "Daroukhaneh Shafa".
  - Metformin 500 mg, tablet, 2× daily at 08:00 and 20:00, with food, 54 tablets on hand.
  - Vitamin D3 1000 IU, capsule, every other day at 09:00, 12 capsules on hand (triggers a refill warning in ~24 days).
- **Schedules:** Amlodipine daily 08:00 from 2026-10-01, no end date; Metformin daily 08:00/20:00; Vitamin D3 intervalDays = 2 from 2026-10-30.
- **Appointments:** Dr. Ahmadi, cardiology check-up, 2026-11-12 10:30, "Vali-e Asr Clinic", note "Bring blood pressure log".
- **Intake history (past week):** 19 taken, 1 missed (Metformin, evening 2026-10-28 — logged as missed after the 30-minute repeat went unconfirmed, caregiver alert sent and acknowledged by Sara).

## Build roadmap (4 weeks)

- **Week 1 — Foundation & patient app:** project setup (Next.js, Supabase, Prisma), auth with magic links, medication CRUD, and the "Today" view with intake logging. End of week: a patient can add a medicine and log a dose on their phone.
- **Week 2 — Scheduling engine & reminders:** schedule computation (times, weekdays, intervals), cron worker, push notifications, SMS fallback, reminder log, and the 30-minute repeat + caregiver alert rule. End of week: reminders fire on time in staging, with delivery logged.
- **Week 3 — Caregiver dashboard & refills:** invites, circle memberships, adherence aggregates, refill math and warnings, appointment list with reminders, weekly email report. End of week: a caregiver can invite, watch, and receive the Sunday report.
- **Week 4 — Polish, accessibility & launch prep:** large-text/high-contrast themes, Persian + English UI, undo window, audit log UI, Playwright end-to-end tests, landing page, and a private beta with 5 real families. End of week: beta-ready, with feedback collection built in.

## Monetization angle

Freemium with a family-priced Pro tier: the patient app and one caregiver are free forever (this is health, and the free tier builds trust and word of mouth), while CareCircle Pro ($4.99/month per family) adds unlimited caregivers, SMS reminders, refill auto-tracking, appointment sync, and the weekly PDF report for the doctor. Clinics and pharmacies are the second revenue line later: a $29/month practice seat to manage schedules for their own patients and receive adherence summaries, sold only after the consumer product has real families using it daily.
