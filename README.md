<div align="center">

<img src="assets/logo-wordmark-dark.png" alt="Shindigs" width="260"/>

```text
 ███████╗██╗  ██╗██╗███╗   ██╗██████╗ ██╗ ██████╗ ███████╗
 ██╔════╝██║  ██║██║████╗  ██║██╔══██╗██║██╔════╝ ██╔════╝
 ███████╗███████║██║██╔██╗ ██║██║  ██║██║██║  ███╗███████╗
 ╚════██║██╔══██║██║██║╚██╗██║██║  ██║██║██║   ██║╚════██║
 ███████║██║  ██║██║██║ ╚████║██████╔╝██║╚██████╔╝███████║
 ╚══════╝╚═╝  ╚═╝╚═╝╚═╝  ╚═══╝╚═════╝ ╚═╝ ╚═════╝ ╚══════╝
   discover  ·  reserve  ·  ticket  ·  scan   —  live acts of India
```

**Event ticketing, reservation and discovery for independent artists, comics and promoters.**
<br/>One database. Five surfaces. Zero oversold seats.

<br/>

[![Next.js](https://img.shields.io/badge/Next.js-16-000000?style=for-the-badge&logo=nextdotjs&logoColor=white)](https://nextjs.org)
[![React](https://img.shields.io/badge/React-19-149ECA?style=for-the-badge&logo=react&logoColor=white)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![Tailwind](https://img.shields.io/badge/Tailwind-v4-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white)](https://tailwindcss.com)
[![Supabase](https://img.shields.io/badge/Supabase-Postgres%20%2B%20Auth-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white)](https://supabase.com)
[![Vercel](https://img.shields.io/badge/Vercel-deployed-000000?style=for-the-badge&logo=vercel&logoColor=white)](https://vercel.com)

[![Payments](https://img.shields.io/badge/payments-Cashfree-6D28D9?style=flat-square)](#-money-is-integers-and-gst-is-extracted)
[![KYC](https://img.shields.io/badge/KYC-Didit-0EA5E9?style=flat-square)](#-everything-else-that-ships)
[![Migrations](https://img.shields.io/badge/migrations-160%2B-F59E0B?style=flat-square&logo=postgresql&logoColor=white)](#-the-one-rule-that-never-bends)
[![Checks](https://img.shields.io/badge/checks-85%20node%3Aassert-22C55E?style=flat-square)](#-checks-not-tests)
[![No overselling](https://img.shields.io/badge/overselling-impossible-EF4444?style=flat-square)](#-the-one-rule-that-never-bends)
[![Source](https://img.shields.io/badge/source-private-lightgrey?style=flat-square)](LICENSE)

<sub>[The rule](#-the-one-rule-that-never-bends) · [Surfaces](#-five-surfaces-one-deploy) · [Architecture](#-architecture) · [The sale](#-the-sale-hold--pay--fulfil) · [Money](#-money-is-integers-and-gst-is-extracted) · [Door](#-the-door)</sub>

</div>

<img src="assets/hero-still.webp" alt="Shindigs" width="100%"/>

> **View only.** This is a showcase. The source code is private and all rights are reserved. See [LICENSE](LICENSE).

```text
$ npm run check
— tests/bento.check.mts                  ok
— tests/didit-webhook.check.mts          ok   ← webhook signature verification
— tests/door-code.check.mts              ok
— tests/email-escaping.check.mts         ok   ← 24 templates, nothing typed reaches HTML raw
— tests/hosts.check.mts                  ok   ← five hosts, three sessions
— tests/pricing.check.mts                ok   ← GST extraction, commission taken once
  … 85 files, no framework, no fixtures, no runner
$ npm run build
✓ Compiled successfully   ✓ Type-checked   ✓ 88 pages   ✓ 40 route handlers
```

---

## ▍ What it is

Shindigs treats a live show as **three verbs, not one**, and each is a surface:

| Verb | What it means | Where |
|---|---|---|
| **Discovery** | Public site, SEO-first event pages, and *Dighead*, a mood-based AI search | public site |
| **Reservation** | The *hold* a seat takes while checkout runs: spoken for, not yet sold | Postgres |
| **Ticketing** | The sale, the ticket (screen + PDF) and the door that scans it | attendee · organizer · door |

Organizers publish; the public finds and buys. **Publishing makes an event public, and a sale decrements the organizer's inventory live.** The audience is deliberately narrow: independent artists, comics and promoters. Not venues-at-large, not enterprise.

---

## ▍ The one rule that never bends

> **No overselling. Ever.**

```text
  buyer A ─┐                                   ┌─ ticket_types row   (FOR UPDATE)
           ├─► place_hold() ─► BEGIN ─────────►│    sold + held + qty  ≤  capacity
  buyer B ─┘                    │              └─ CHECK (…)          ← backstop
                                │
        second buyer WAITS on the row lock,     if the app is wrong, the
        re-reads the true count, then commits   database is not.
        or is refused.
```

Inventory commits inside one Postgres transaction with `SELECT … FOR UPDATE` on the ticket-type row, **plus a `CHECK` constraint as a backstop**. Ticket *phases* (locked tiers, "opens after the first tier sells out", sales windows on a clock or as an offset from doors) are enforced in the same function, so no client can skip a rung.

---

## ▍ Five surfaces, one deploy

One Next.js deploy, one Supabase project, one user per person, **three separately scoped sessions**. The host of the request decides which surface you are on.

```text
                          ┌───────────────────────────────────────────────┐
                          │             ONE Next.js 16 deploy             │
                          │        edge proxy  ─►  host → surface         │
                          └──┬─────────┬──────────┬─────────┬──────────┬──┘
                             │         │          │         │          │
                       shindigs.cc   app.       events.    qr.       admin.
                             │         │          │         │          │
                        ┌────▼───┐ ┌───▼────┐ ┌───▼────┐ ┌──▼───┐ ┌────▼───┐
                        │ PUBLIC │ │ATTENDEE│ │ORGANIZ.│ │ DOOR │ │ ADMIN  │
                        │  site  │ │console │ │console │ │ staff│ │console │
                        └────┬───┘ └───┬────┘ └───┬────┘ └──┬───┘ └────┬───┘
                        guest or    attendee    organizer  signed     admin
                        attendee    session     session    door cookie session
```

| Surface | Canvas | Sign-in |
|---|---|---|
| **Public site** | dark violet | attendee session *or* guest checkout |
| **Attendee console** | blue/mint, toggleable | Google · WhatsApp OTP |
| **Organizer console** | blue/mint, toggleable | email + password |
| **Staff door** | blue/mint dark | organizer's email + a door code |
| **Admin** | blue/mint dark | email + password, and an allow-listed admin row |

The staff door carries **no account session at all**, only a signed door cookie, so a lost phone at a gate can never become a console login.

---

## ▍ Architecture

```mermaid
flowchart LR
  subgraph Edge["Vercel · Fluid Compute"]
    P[proxy<br/>host → surface<br/>CSP nonce · cookie flags]
    RSC[React 19 Server Components<br/>+ Server Actions]
    API[Route handlers<br/>checkout · webhook · cron · PDF]
  end

  subgraph SB["Supabase"]
    PG[(Postgres<br/>RLS · row locks · CHECK<br/>pg_cron)]
    AU[Auth]
    ST[Storage]
  end

  CF[Cashfree<br/>payments + webhook]
  DI[Didit<br/>organizer KYC]
  EM[ZeptoMail · SES<br/>transactional · invites]
  WA[WhatsApp<br/>tickets · OTP]
  AI[Dighead<br/>AI discovery]

  Browser --> P --> RSC --> PG
  P --> API
  RSC --> AU
  API <--> CF
  API --> DI
  API --> EM
  API --> WA
  RSC --> AI
  RSC --> ST
```

**Stance.** Server Components own the reads, Server Actions own the writes, and Postgres owns the invariants. Anything that must hold under concurrency (inventory, idempotent fulfilment, refund state) lives in SQL, behind row-level security and explicit function grants.

| Layer | Choice |
|---|---|
| Framework | Next.js 16 App Router, React 19 |
| Data | Supabase Postgres, 160+ additive migrations |
| Auth | Supabase Auth, three cookie-scoped sessions |
| Payments | Cashfree, integer paise everywhere |
| Mail | ZeptoMail (transactional), SES (invite campaigns) |
| Messaging | WhatsApp for ticket delivery and phone proof-of-ownership |
| KYC | Didit, gating payouts only |
| Tickets | `pdf-lib` + `qrcode`; screen stub and PDF are one design in two orientations |
| Door scan | `@zxing/browser`, lazy-loaded so it never ships to people who never open a door |
| Motion | GSAP, registered in exactly one place |
| UI | shadcn/ui on Base UI, three palettes, one token system |

---

## ▍ The sale: hold → pay → fulfil

```mermaid
sequenceDiagram
  autonumber
  actor B as Buyer
  participant A as App (server action)
  participant DB as Postgres
  participant C as Cashfree
  participant M as Mail · WhatsApp

  B->>A: choose tiers, qty (guest or signed-in)
  A->>A: validate · rate-limit · phone proof
  A->>DB: place_hold() — row lock on ticket_type
  DB-->>A: hold + expiry   (or: sold out / phase locked)
  A->>C: create order (integer paise, GST already extracted)
  C-->>B: hosted checkout
  B->>C: pay
  C->>A: signed webhook
  A->>A: verify signature · idempotency
  A->>DB: commit order — hold → sale, tickets minted
  A->>M: PDF + QR by email, WhatsApp, on-screen stub
  Note over DB,A: cron sweeps expired holds and orders,<br/>a reconcile route heals a missed webhook
```

A declined card **retires** the attempt without **terminating** the event for that buyer. An early version locked people out of a whole event after one bad card.

---

## ▍ Money is integers, and GST is extracted

Money is **integer minor units (paise)** end to end, and the rupee↔paise and signature code is kept import-free so it can be asserted in isolation.

```text
  sticker price  ₹ 1,180   ─┐
                             ├─ GST extracted (never added on top)      ₹ 180
  organizer net  ₹ 1,000   ─┘
        └─ commission taken ONCE, on the pre-GST base
  ─────────────────────────────────────────────────────────────────────────
  refunded order  →  stops earning commission,  KEEPS its platform fee
  sub-₹500 ticket →  zero GST, for every organizer
```

*Illustrative numbers.* The invariants are pinned by checks, and a conservation sweep asserts that what the buyer paid equals what is split between organizer, platform and tax.

---

## ▍ The door

```text
  ┌──────────────────────────┐        ┌──────────────────────────┐
  │  qr.shindigs.cc          │        │  attendee phone          │
  │                          │        │                          │
  │   [ organizer email ]    │        │   ┌──────────────────┐   │
  │   [ door code       ]    │  scan  │   │ ▄▄▄▄▄ ▄▄  ▄▄▄▄▄  │   │
  │   [ ▶ open camera   ]◄───┼────────┼───│ █   █ ▀▄▀ █   █  │   │
  │                          │        │   │ █▄▄▄█ ▄▀▄ █▄▄▄█  │   │
  │  ✔ checked in · 19:42    │        │   └──────────────────┘   │
  └──────────────────────────┘        │   tap to enlarge · wake  │
     signed cookie, no account        │   lock keeps it lit      │
                                      └──────────────────────────┘
```

- Staff sign in with the organizer's email and a door code. No console access.
- Check-in has a database-fault path so a **real attendee is never turned away by a hiccup**.
- The phone screen *is* the scanning surface: tap-to-enlarge QR plus a wake lock.
- The camera reader loads only when a door is opened.
- Offline-capable check-in for rooms with bad signal.

---

## ▍ Everything else that ships

<table>
<tr><td width="50%" valign="top">

**For the public**
- SEO-first event pages (JSON-LD, sitemap, social cards)
- **Dighead**: describe a mood, get picks
- Guest checkout with WhatsApp phone proof
- Ticket transfer, waitlist, follows, calendar links, reminders
- Recurring events collapsed to one card per series

**For organizers**
- 3-step create wizard; live-event edits walk the same wizard
- Ticket tiers, phases, sales windows, discount codes, cover charge
- Guest lists with CSV/XLSX import (xlsx parsed with zero dependencies)
- Contact book, block-based email composer, WhatsApp invites
- Teams, table bookings, per-event readiness checklist
- Payouts gated on Didit identity verification
- Dighead for organizers: a console assistant with tools

</td><td width="50%" valign="top">

**For operators**
- Approvals, refunds, payouts, earnings, people, admins
- Guest delivery ledger (email + WhatsApp, per ticket)
- System health, free-tier meters, server error log
- Admin unpublish / publish / delete with cascade

**For the platform**
- One atomic rate limiter across every public entry point
- Email outbox with retries; cron lease so sweeps never overlap
- In-database schedules via `pg_cron`
- Nightly DB backups
- Strict CSP with per-request nonce, hardened cookies and headers
- Column-level and function-level grants

</td></tr>
</table>

---

## ▍ Checks, not tests

85 plain `node:assert` scripts. No framework, no fixtures, no runner. They import the pure modules by relative path, so every module a check touches stays free of framework imports. The modules you least want unchecked are checked hardest: money, the webhook signature, GST extraction, host routing, and email escaping across all 24 templates.

## ▍ Conventions that are load-bearing

| Rule | Reason |
|---|---|
| **A server action never throws a message meant for a person; it returns `{ error }`** | Next strips thrown messages in production, so a useful refusal became a generic digest. |
| **Success is a toast, not an inline tick** | An inline tick under a sticky bar was never seen. |
| **A saved record opens locked** | Nobody is deliberately *in* a screen of live fields. |
| **Animated hero is served through a raw `<img>`** | Next's optimizer re-encodes animated sources to a still frame. |
| **Time is UTC in the DB, rendered in `Asia/Kolkata`** | A UTC host would show a late-night IST event on the previous day. |
| **`grid-cols-1` on any grid wider than a phone** | The implicit grid track stretched a 390px screen to 1082px. |

## ▍ Design

Three palettes, one system: violet for the marketing surface, blue/mint for both consoles, and a light mode as the reader's opt-in. The theme is a cookie read on the server, so there is no flash and no client theme script.

---

## ▍ Contact

arcxx1995@gmail.com

<div align="center">

```text
   ┌──────────────────────────────────────────────┐
   │  made for the ones who play the small rooms  │
   └──────────────────────────────────────────────┘
        ▂▃▅▇█▇▅▃▂  doors at 7 · first act at 8  ▂▃▅▇█▇▅▃▂
```

</div>
