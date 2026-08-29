# Case Study 01 — Metups Zimbabwe
## Zimbabwe's First Dedicated Second-Hand Marketplace

**Role:** Sole Founder & Solo Developer  
**Type:** Progressive Web Application (PWA)  
**Stack:** HTML · CSS · Vanilla JavaScript · Supabase · PostgreSQL  
**Live:** [metups.com](https://metups.com)  
**Timeline:** 2025 — Present  
**Status:** Live in production

---

## The Problem

Zimbabwe has no dedicated second-hand marketplace.

When a university student wants to sell a laptop at the end of semester, their options are:
- Post in a WhatsApp group and hope the right person sees it
- Go to Roadport or a flea market and negotiate in person for hours
- List on Facebook Marketplace, which has low adoption and a poor mobile experience in Zimbabwe
- Let it sit unused

None of these work reliably. WhatsApp groups have no search, no categories, no trust signals, and no structure. A seller in Borrowdale cannot reach a buyer in Highfields unless they happen to be in the same group. Buyers cannot browse. Sellers cannot be verified. Scams are common and there is no recourse.

The informal second-hand economy in Zimbabwe is enormous. It runs entirely on trust-based personal networks. Metups is the infrastructure that makes that economy work at scale.

---

## The Solution

A mobile-first PWA that works on any smartphone, on slow mobile data, without requiring a download.

Sellers list items in under 2 minutes. Buyers search by category, location, and price. Messaging happens in-app — no phone number exchange required before trust is established. Sellers can earn a verified badge. Listings surface nearest buyers first.

The product is free to use. Revenue comes from featured listings, verification badges, and subscriptions — all designed to align platform incentives with seller success.

---

## My Role

I conceived, designed, built, and launched Metups entirely alone.

There is no co-founder. No development team. No agency. Every line of code, every SQL function, every design decision, every business document, and every marketing strategy came from one person — me — while studying full-time for a Computer Engineering degree.

---

## Technical Architecture

### Frontend

The frontend is intentionally framework-free. Plain HTML, CSS, and vanilla JavaScript.

This was a deliberate architectural decision, not a limitation. A React or Vue application would have added bundle overhead that directly impacts load times on Zimbabwe's mobile networks. A page that loads in 1.2 seconds on a 3G connection beats a React app that loads in 4 seconds on the same connection, regardless of how elegant the component structure is.

Key frontend systems built from scratch:

**Fuzzy Search Engine**  
Standard SQL `LIKE` queries do not handle misspellings, which are common in any user-generated content system. I implemented a client-side fuzzy search combining Levenshtein distance (edit distance algorithm) and trigram matching. A search for "labtop" surfaces "laptop" listings. A search for "samsung galxy" returns Samsung Galaxy results. This required implementing both algorithms in vanilla JavaScript and tuning the scoring weights for the specific distribution of listing titles in the Metups database.

**PersonaEngine**  
A session-level affinity engine that adjusts listing rankings based on inferred user preferences without requiring login. It tracks category interactions, price range engagement, and location signals within the session and applies a weighted score boost to listings that match the emerging profile. This is not a recommendation system in the ML sense — it is a deterministic scoring adjustment that produces meaningfully personalised results from the first interaction.

**SmartSuggest**  
An autocomplete system with debounced query processing, local caching of recent queries, and category-aware suggestions. Built entirely without a library.

**PWA Implementation**  
Full service worker with offline capability, app manifest for home screen installation, and a background sync strategy for listing drafts. The app is installable on Android without going through the Play Store.

**Layout System**  
A custom sidebar navigation with a filter chip bar and advanced filter drawer. Debugging this required diagnosing a subtle bug where `position: sticky` was reserving invisible vertical space and causing the chip rail to overflow incorrectly — traced to missing flex rules on the `#chipRail` container.

---

### Backend & Database

The backend is Supabase (PostgreSQL) with no intermediary API layer. The frontend calls Supabase directly via the JavaScript client for standard CRUD operations, and calls custom PostgreSQL RPCs for complex operations.

**Database Design**  
15+ relational tables with proper normalisation, foreign key constraints, and cascading rules. Core tables: profiles, products, product_images, categories, messages, conversations, wishlists, notifications, referrals, admin_users, admin_sessions, audit_log, support_payments, featured_listings.

**Row Level Security**  
Every table has RLS policies. Users can only read and write their own data. Admin operations run through SECURITY DEFINER functions that bypass RLS with controlled, audited access.

**Custom SQL Functions**  
All complex operations are encapsulated in PostgreSQL functions rather than application code. This keeps business logic close to the data and reduces round trips. Key functions include: product scoring with featured boost, city-based listing ranking, analytics aggregations, cohort retention calculations, and admin session management.

**A Real Bug I Diagnosed and Fixed**  
The admin authentication system uses pgcrypto for password hashing and token generation. After implementing session management, logins were returning `ok: true` with a token, but subsequent verification calls were returning `session not found`. The bug had two layers:

1. pgcrypto functions in Supabase must be called with the explicit `extensions.` schema prefix (`extensions.crypt()`, `extensions.digest()`, `extensions.gen_random_bytes()`). Calling them without the prefix causes silent failures in certain Supabase configurations.

2. The token hash was being computed differently in the login function versus the verification function — one used `encode(digest(token, 'sha256'), 'hex')` and the other used a slightly different expression. The hashes never matched.

3. The session INSERT was being silently blocked by an RLS policy — the SECURITY DEFINER function was running as a role that did not have INSERT privileges on the `admin_sessions` table.

All three issues had the same symptom (session not found) but completely different root causes. Diagnosing them required reading PostgreSQL execution logs, testing each function in isolation, and understanding how Supabase's extension schema interacts with SECURITY DEFINER execution context.

---

### Admin System

The admin dashboard is a custom-built system, not an off-the-shelf solution. It has four access tiers:

| Tier | Role | Access |
|---|---|---|
| 0 | Super Admin | Everything |
| 1 | Admin | Full analytics, user and listing management |
| 2 | Moderator | Listings and moderation only |
| 3 | Analyst | Read-only analytics |

Ten custom PostgreSQL RPCs handle all admin operations: `admin_login`, `admin_verify`, `admin_logout`, `admin_get_analytics`, `admin_get_users`, `admin_get_listings`, `admin_action`, `admin_get_flags`, `admin_get_audit`, `admin_manage_admins`.

Session tokens are generated using `gen_random_bytes(32)`, stored as SHA-256 hashes, and verified on every request. The login page clears stale session tokens on load to prevent authentication loops.

---

### Commercial Analytics Dashboard

Six tabs, each pulling from custom PostgreSQL views:

**Overview** — 12 KPI cards with sparklines, growth chart, category distribution donut, listing age analysis, conversion funnel, platform health score, top listings

**Growth** — User acquisition bars with rolling 7-day average, listings velocity chart, DAU trend, cohort retention heatmap

**Marketplace** — Supply vs demand table by category, price distribution histogram, sell-through rate by category

**Engagement** — Message activity over time, seller response time distribution, conversation-to-sale funnel

**Geography** — City-level bar chart, city performance table with GMV, listing density by region

**Revenue** — GMV line chart, projected fee revenue, GMV by category, revenue summary table

The analytics cohort view required fixing a PostgreSQL date subtraction bug — subtracting two `DATE` values returns an `INTEGER` in PostgreSQL, not an `INTERVAL`, so extracting weeks required integer division rather than `EXTRACT(WEEK FROM interval)`.

---

### Monetisation Infrastructure

All revenue infrastructure is built and ready to activate:

- **Featured listings** — `is_featured` boolean and `featured_until` timestamp on products table. Frontend scoring gives featured listings a `+0.3` weight boost in the ranking algorithm. Payable via EcoCash or Paynow.
- **Verification badges** — `is_verified_seller` on profiles. Manual admin approval flow for Phase 1, automated for Phase 2.
- **Support Metups** — Voluntary tip system with `support_payments` table, EcoCash manual flow, reference number verification, supporter badge award, and a full admin supporters tab.
- **Subscriptions** — `subscription_tier` and `subscription_until` on profiles. Listing count enforcement at the point of listing creation. Upgrade prompt UI built.
- **Sale commission** — 3% honour-based system with UI prompt on mark-as-sold flow.

---

### Business Infrastructure

Building Metups required going far beyond code.

**Legal:** Full legal document library — Terms of Service, Privacy Policy, Cookie Policy, Acceptable Use Policy, GDPR and POPIA compliance statements, Data Processing Agreement, Refund Policy, Subscription Policy, Copyright Notice, Accessibility Statement. Articles of Incorporation prepared under Zimbabwe's Companies and Other Business Entities Act [Chapter 24:31] for PBC registration with CIPAZ.

**HR:** Employment Contract, Freelancer Agreement, Equity & Vesting Agreement, NDA, Community Manager Agreement — all written to Zimbabwe's Labour Act [Chapter 28:01].

**Financial:** 3-year financial projections, break-even analysis at 60 paying users, seed funding model ($6,500 ask), Revenue & Expense Tracker with EcoCash reconciliation log.

**Marketing:** 12-week university outreach campaign targeting Student Representative Councils across Zimbabwe. TikTok campaign ("It Won't Fit") — 50-video script series built on Jonah Berger's STEPPS framework, designed for production at scale using Sora AI.

---

## Problems I Solved

### The WhatsApp Group Problem
WhatsApp group selling has no discoverability, no structure, and no permanence. The architecture of Metups — location-based ranking, category filtering, fuzzy search — directly addresses each of these. A seller in Harare does not need to be in the same group as a buyer. They just need to list on Metups.

### The Trust Problem
Zimbabwe's informal economy operates on trust-based personal networks. Extending those networks digitally requires trust signals. The verified seller badge, in-app messaging (which keeps contact details private until trust is established), the moderation system, and the buyer safety guide all address the trust problem systematically.

### The Data Problem
No one knew what was actually being traded in Zimbabwe's second-hand market, at what prices, in which cities, with what velocity. The Metups analytics system is the first attempt to answer these questions with real data. The geography tab and the supply/demand analysis are genuinely novel infrastructure for Zimbabwe's digital economy.

---

## What I Learned

**Technical:**
- PostgreSQL RLS behaves differently inside SECURITY DEFINER functions depending on the executing role — a subtlety that is not well documented and took significant debugging to resolve
- Fuzzy search tuning is an empirical problem, not a theoretical one — the right Levenshtein threshold depends on the actual distribution of your data, not on what algorithms papers suggest
- PWA service worker caching strategies have significant trade-offs between freshness and offline capability that must be tuned per content type
- `position: sticky` interacts with flex containers in ways that are non-obvious and poorly covered in documentation

**Product:**
- The supply side (sellers) is harder to acquire than the demand side (buyers) in a marketplace — every product decision should prioritise reducing friction for sellers first
- Trust is not a feature. It is the product. Every component of Metups is ultimately a trust system.
- Listing quality (photo quality, description quality) directly determines whether a marketplace feels alive or dead — which is why the Seller Onboarding Guide and content creator role exist

**Business:**
- Solving one problem (connecting buyers and sellers) requires solving ten adjacent problems (legal, payments, trust, marketing, support, moderation, analytics, monetisation, hiring, incorporation) — and you have to solve all of them alone before you can hire anyone to help
- A product that is free to use and genuinely solves a problem will market itself eventually — but "eventually" requires a structured 12-week plan and daily consistent action to accelerate

---

## Outcomes

- Live at metups.com — accessible to every Zimbabwean with a smartphone
- Complete legal, financial, and business infrastructure for a registerable Zimbabwe company
- Production-grade admin system with commercial analytics
- Full monetisation infrastructure ready to activate
- 12-week marketing campaign in execution targeting Zimbabwe's university population

---

## Stack Summary

| Layer | Technology |
|---|---|
| Frontend | HTML5, CSS3, Vanilla JavaScript |
| PWA | Service Worker, Web App Manifest |
| Search | Custom Levenshtein + Trigram (vanilla JS) |
| Personalisation | PersonaEngine (custom session affinity) |
| Database | PostgreSQL via Supabase |
| Auth | Supabase Auth + custom admin RPCs |
| Security | Row Level Security, pgcrypto |
| Analytics | Custom PostgreSQL views + Chart.js |
| Payments | EcoCash + Paynow |
| Hosting | Netlify |
| Domain | metups.com |

