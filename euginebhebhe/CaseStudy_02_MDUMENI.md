# Case Study 02 — MDUMENI
## AI Agronomist for Zimbabwean Smallholder Farmers

**Role:** Lead Developer  
**Type:** Cross-Platform Mobile Application + REST API  
**Stack:** React Native · TypeScript · FastAPI · Python · Supabase · PostgreSQL  
**Organisation:** INTELLI-Farming, University of Zimbabwe  
**API:** [mdumeni-api.onrender.com](https://mdumeni-api.onrender.com)  
**Timeline:** 2025 — Present  
**Status:** Deployed and active  
**Team:** Eugine Bhebhe (Lead Developer), Ethel Kuvirima, Mathew Madutsa, Anotidaishe Muhamba, Watson Mashavire

---

## The Problem

Zimbabwe's agricultural sector employs over 60% of the population but most smallholder farmers have no reliable access to agronomic expertise.

A professional agronomist costs money, travels slowly, and cannot be in multiple places at once. Government extension workers are stretched thin across thousands of farms. When a farmer sees an unfamiliar disease on their maize crop, or needs to know the optimal planting window for their region, or wants to understand current market prices before deciding what to grow — they mostly guess.

Bad guesses destroy yields. Destroyed yields destroy livelihoods.

The INTELLI-Farming project at the University of Zimbabwe set out to change this: put an expert agronomist in every farmer's pocket, available any time, for any crop, in any of Zimbabwe's agricultural regions.

That is MDUMENI. The name means advisor or counsellor in Ndebele.

---

## My Role

I joined the INTELLI-Farming project as lead developer.

The project had research direction and domain expertise from the team. My responsibility was to take that knowledge and turn it into working, deployed software — the mobile application farmers would actually use and the API that powers it.

In a team of five, I owned all technical architecture decisions, all full-stack implementation, all deployment configuration, all debugging, and delivery of working features against research milestones.

---

## What I Built

### Mobile Application

A cross-platform mobile application built in React Native with Expo SDK 54, written entirely in TypeScript.

The choice of React Native over Flutter or native Android was deliberate: the team needed a single codebase that would run on both Android (primary target, given Zimbabwe's device market) and iOS, and React Native's JavaScript bridge allowed faster iteration on business logic with the team while maintaining native performance for UI rendering.

TypeScript was non-negotiable. Agricultural data has complex types — crop variants, regional classifications, growth stage enumerations, yield metrics — and runtime type errors in a production agricultural advisory app have real consequences for farmers acting on that advice.

**Architecture**  
Clean architecture with strict separation of concerns across three layers:
- **Presentation** — React Native screens and components, Expo navigation
- **Domain** — Business logic, use cases, entity definitions
- **Data** — Repository pattern, API client, local cache

This separation means that changing the backend API does not require touching screen code, and changing the UI does not affect business logic. For a research project where both the data model and the user interface evolve rapidly, this architecture prevented cascading changes from breaking unrelated features.

**Key Screens and Features**  
- Crop advisor with 60-crop database — disease diagnosis, treatment recommendations, planting guidance
- Pfumvudza basin planting calculator for Regions 4 and 5 (detailed below)
- MarketScreen with four real-time data tabs: current prices, active buyers, active sellers, price trends
- Regional agricultural calendar
- Offline-first data access for commonly used crop information

---

### Pfumvudza Basin Logic

Pfumvudza is a conservation farming technique formally adopted by the Government of Zimbabwe as part of the Presidential Input Scheme. It uses precise basin spacing and planting geometry to maximise water retention in low-rainfall areas.

Regions 4 and 5 are Zimbabwe's driest agricultural regions — Matabeleland South, parts of Masvingo, Beitbridge — where rainfall is insufficient for conventional row planting. Pfumvudza is not optional in these regions; it is often the difference between a viable harvest and crop failure.

I implemented Pfumvudza basin calculation logic for 13 crops across these regions. The logic accounts for:
- Basin dimensions (length × width × depth) varying by crop type and soil classification
- Inter-basin spacing calculated from plant spacing requirements and water catchment geometry
- Seed rate adjustment for basin vs row planting
- Expected yield differentials under Pfumvudza vs conventional methods

This was not a UI feature. It required understanding the actual agronomy, working with the domain knowledge from the research team, and translating it into precise calculations that give farmers actionable basin layouts for their specific crop and plot size.

---

### Backend API

A FastAPI backend deployed on Render at `mdumeni-api.onrender.com`.

FastAPI was chosen over Django REST Framework or Flask for three reasons: automatic OpenAPI documentation (essential for a research project where the API spec needs to be shared with supervisors), native async support for handling concurrent advisory requests, and Pydantic's schema validation which enforces data integrity at the API boundary.

**A Full-Stack Debugging Sprint**  
When I conducted a systematic audit of the codebase, I found and fixed four critical bugs that were causing silent failures in production:

**Bug 1 — Duplicate endpoint routing**  
Two routers were registering the same endpoint paths. FastAPI resolves duplicate routes by using the first registered handler and silently ignoring the second. This meant that certain crop endpoints were returning data from the wrong handler — the symptom was intermittent incorrect responses with no error logged.

The fix required auditing every router registration in `main.py`, identifying the duplicate paths, consolidating the handlers, and restructuring the router hierarchy so each endpoint path was unique across the application.

**Bug 2 — Hardcoded crop ID mismatches**  
Crop IDs were hardcoded as integer literals in multiple places across the codebase. When the crop database was updated (new varieties added, IDs resequenced), the hardcoded values became stale. A query for "maize disease profile" was returning data for the wrong crop because the hardcoded ID 14 now pointed to a different crop than it did when the code was written.

The fix was to replace all hardcoded IDs with named lookups against the crop database, and to add a validation step on startup that verifies all referenced crop IDs exist in the current database state.

**Bug 3 — Personal data in default field values**  
Several FastAPI endpoint definitions had personal data — likely from development testing — embedded directly in the `default=` parameter of Pydantic fields. This meant the API documentation (auto-generated by FastAPI) was publicly exposing this data. It also meant any client that did not send a value for these fields would receive responses populated with someone's personal information.

All default values were replaced with appropriate null or empty defaults, and a code review checklist was added to the documentation to prevent recurrence.

**Bug 4 — Duplicate marketplace.py in two unlinked locations**  
A `marketplace.py` file existed in two separate directories of the project. One was imported by the main application. The other was not imported anywhere — it was a stranded file from an earlier refactoring. When changes were made to the marketplace logic, they were being made to the unimported file, which meant the changes had no effect on the running application.

The fix was to delete the stranded file, ensure the correct file was the single source of truth, and wire the marketplace router properly into `main.py`.

---

### Marketplace Feature

After fixing the existing bugs, I implemented the full marketplace feature:

**`marketplace_router`** — A FastAPI router handling CRUD operations for marketplace listings: create listing, read listings (with filtering by crop, region, price range), update listing, delete listing, search listings.

**`marketplace_tables.sql`** — PostgreSQL schema for Supabase including: `market_listings` (seller, crop, quantity, price, region, contact, created_at), `market_prices` (historical price data by crop and region), `market_buyers` (buyer intent signals by crop and region). Full RLS policies on each table.

**MarketScreen** — The mobile screen with four tabs:
- **Prices** — Current market prices for major crops, updated from the database
- **Buyers** — Active buyers with crop needs, quantity, and contact information
- **Sellers** — Active sellers with produce, quantity, and asking price
- **Trends** — Price trend visualisation over time for selected crops

This gave farmers, for the first time, a single place to see both agronomic guidance and market intelligence for the same crop — so they could make growing decisions with both the cultivation risk and the commercial opportunity visible simultaneously.

---

### Documentation Suite

I produced a 10-document technical documentation suite:

1. Architecture Overview — system design, component relationships, data flow
2. API Reference — all endpoints, request/response schemas, authentication
3. Database Schema — table definitions, relationships, RLS policy descriptions
4. Mobile App Structure — screen hierarchy, navigation flow, state management
5. Deployment Guide — Render configuration, Supabase setup, Expo EAS build
6. Crop Data Model — the 60-crop database structure and update procedures
7. Pfumvudza Logic Reference — the basin calculation formulas and regional parameters
8. Testing Guide — unit test coverage, integration test patterns, manual test scripts
9. Contribution Guidelines — branching strategy, code review process, naming conventions
10. Changelog — version history with feature and fix descriptions

Documentation of this quality is what separates a research project that can be handed off and continued from one that dies when the original developer leaves.

---

## Technical Challenges

### Cross-Platform Consistency
React Native's promise of "write once, run anywhere" meets reality at platform-specific APIs. Camera access, file system operations, and notification scheduling all behave differently on Android and iOS. Managing these differences cleanly required abstracting platform-specific code behind a consistent interface that the rest of the application could call without knowing which platform it was on.

### Low-Connectivity Scenarios
Farmers in Zimbabwe's agricultural regions — particularly Regions 4 and 5 — often have intermittent or no mobile data access when they are in their fields. The app needed to function usefully without a network connection for the most critical features (crop disease identification, Pfumvudza calculations) while gracefully degrading for features that require real-time data (market prices).

This required a cache-first data strategy for crop data and a clear UI pattern for communicating data freshness to the user.

### Type Safety Across the Stack
TypeScript on the frontend and Python with Pydantic on the backend meant two separate type systems that needed to remain in sync. A schema change in the FastAPI Pydantic models required a corresponding change in the TypeScript interfaces. I implemented a manual sync process with a checklist step in the contribution guidelines, and documented the planned migration to auto-generated TypeScript types from the OpenAPI spec.

---

## What I Learned

**Technical:**
- FastAPI's silent route shadowing for duplicate paths is a significant footgun in larger applications — router organisation and naming conventions need to be enforced from the start
- React Native's `FlatList` performance degrades significantly with complex item renderers on low-end Android devices — optimising for the actual target hardware (not a simulator) requires testing on physical devices representative of the user base
- Pydantic's default value handling in FastAPI endpoint definitions is a security concern that is not obvious from the documentation — any default that could contain real data needs to be `None` or an empty value

**Domain:**
- Agricultural advisory software needs to be conservative. A misdiagnosed plant disease with a recommended treatment that is wrong does real damage to real farms. Building appropriate uncertainty signals into the AI responses — "consult a local extension worker if symptoms persist" — is not hedging, it is responsible design.
- Pfumvudza is not just a planting technique. It is a social and political programme in Zimbabwe. Building the calculator required understanding that context, not just the geometry.

**Team:**
- Leading technical work in a research team means translating between two languages: the language of academic research (hypothesis, methodology, variables) and the language of software engineering (inputs, functions, outputs, side effects). Getting good at that translation is what makes a research project ship.

---

## Outcomes

- Deployed API serving agricultural advisory data across 60 crop profiles
- Cross-platform mobile application with Pfumvudza basin calculations for 13 crops across Regions 4 and 5
- Full marketplace feature connecting farmers with buyers and sellers
- 4 critical bugs diagnosed and fixed that were causing silent production failures
- 10-document technical documentation suite enabling project continuity
- Research project advancing toward formal publication through INTELLI-Farming at the University of Zimbabwe

---

## Stack Summary

| Layer | Technology |
|---|---|
| Mobile | React Native, Expo SDK 54 |
| Language | TypeScript (mobile), Python (backend) |
| Architecture | Clean Architecture, MVVM |
| Backend | FastAPI |
| Database | PostgreSQL via Supabase |
| API Documentation | OpenAPI (auto-generated by FastAPI) |
| Deployment | Render (API), Expo EAS (mobile) |
| Data Validation | Pydantic (backend), TypeScript interfaces (mobile) |

