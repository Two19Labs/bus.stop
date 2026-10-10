# 🚌 BusStop — School Bus Live Tracking SaaS (MVP)

> **A mission-critical, mobile-first responsive web application for real-time school bus fleet tracking, role-based operations, and intelligent proximity alerts.**

[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue.svg?logo=typescript)](https://www.typescriptlang.org/)
[![Next.js](https://img.shields.io/badge/Framework-Next.js-black.svg?logo=next.js)](https://nextjs.org/)
[![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-336791.svg?logo=postgresql)](https://www.postgresql.org/)
[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Build%20Status-Phase%200%20(Planning)-orange.svg)]()

---

## 📑 Table of Contents

- [1. Executive Summary & Vision](#1-executive-summary--vision)
- [2. Problem Statement & Scope](#2-problem-statement--scope)
- [3. Core Product Principles](#3-core-product-principles)
- [4. Role Matrix & User Journeys](#4-role-matrix--user-journeys)
- [5. System Architecture & Tech Stack](#5-system-architecture--tech-stack)
- [6. GPS Ingestion & Geofencing Engine](#6-gps-ingestion--geofencing-engine)
- [7. Critical Browser GPS Limitations](#7-critical-browser-gps-limitations)
- [8. Notification System & Event Matrix](#8-notification-system--event-matrix)
- [9. Database Schema & Data Models](#9-database-schema--data-models)
- [10. API Specification & Realtime Channels](#10-api-specification--realtime-channels)
- [11. Security, Multi-Tenancy & Child Safety](#11-security-multi-tenancy--child-safety)
- [12. Comprehensive Phased Implementation Plan](#12-comprehensive-phased-implementation-plan)
- [13. Getting Started & Development Setup](#13-getting-started--development-setup)
- [14. Real-Device Testing Protocol](#14-real-device-testing-protocol)
- [15. Acceptance Criteria & Definition of Done](#15-acceptance-criteria--definition-of-done)

---

## 1. Executive Summary & Vision

**BusStop** is a lightweight, high-reliability web platform tailored for schools with small-to-medium bus fleets. It solves the perennial question asked by thousands of parents every morning: *"Where is the bus, and when will it reach our stop?"*

By running directly inside mobile and desktop web browsers:
- **No App Store Friction:** Drivers and parents do not need to download heavy native apps; authorized users simply sign in via a mobile-optimized PWA-ready web client.
- **Hardware-Free GPS:** Drivers broadcast coordinates using their existing smartphone browser during active scheduled trips.
- **Precision Geofencing:** Servers calculate real-time proximity against each student's designated stop, dispatching 500-metre advance alerts automatically.
- **Operational Oversight:** School administrators get a live control room displaying bus status, trip delays, GPS freshness, and safety alerts.

---

## 2. Problem Statement & Scope

### The Problem
1. **Parent Anxiety & Guesswork:** Parents wait at stops in extreme heat or rain without knowing if the bus has arrived, is stuck in traffic, or is cancelled.
2. **School Administrative Bottleneck:** Reception desks are flooded with urgent phone calls asking for updates on delayed routes.
3. **Driver Distraction:** Drivers are repeatedly called or messaged while driving, posing severe road safety risks.

### In Scope (MVP)
- **Single Responsive Web Application** serving all three user roles with dynamic, role-restricted UI views.
- **School-Controlled Multi-Tenant Onboarding:** Admins invite drivers and parents via secure, single-use, cryptographically hashed tokens.
- **Live Trip GPS Tracking:** Active driver broadcasts location via `navigator.geolocation.watchPosition` over HTTPS.
- **500m Server-Side Proximity Geofencing:** Intelligent entry/exit hysteresis and deduplication for assigned bus stops.
- **Admin Fleet Command Center:** Live map, route visualization, stale GPS detection, delay management, and incident logging.
- **In-App Notification Center:** Real-time persistence of trip lifecycle updates and proximity triggers.

### Non-Goals (Strict MVP Boundaries)
- ❌ **No Student Boarding / Attendance Confirmation:** GPS tracks the vehicle only; it **does not** verify if a specific child boarded or exited.
- ❌ **No Hardware Dependencies:** No OBD-II or dedicated IoT GPS devices required.
- ❌ **No School ERP / Fee Payment / CCTV Modules:** Focused strictly on live transport visibility.
- ❌ **No Public Self-Registration:** Users cannot browse or self-associate with schools without admin provisioning.

---

## 3. Core Product Principles

| Principle | Description |
|---|---|
| **1. School-Controlled Access** | Zero open sign-up. Admins invite parents/drivers. Roles requested at sign-up grant **no** automatic privileges. |
| **2. Server is Authoritative** | Geofence calculations, trip states, permissions, and alerts are computed strictly on the backend. |
| **3. Freshness Over False Certainty** | Never present stale GPS coordinates as "live". Clearly flag signal age and degradation. |
| **4. Events, Not Notification Spam** | GPS streams update map markers silently; only meaningful operational state transitions trigger alerts. |
| **5. Privacy & Least Privilege** | Parents view *only* their linked child's bus and stop. Drivers view *only* assigned trips. |
| **6. Driver Safety First** | Zero driver interaction required while vehicle is in motion. Large high-contrast touch targets. |
| **7. Honest Product Boundaries** | Vehicle location does not prove child safety or physical boarding. UI states explicitly reflect this reality. |

---

## 4. Role Matrix & User Journeys

```mermaid
flowchart TD
    subgraph School Admin
        A1[Log into Admin Portal] --> A2[Configure School, Fleet & Stops]
        A2 --> A3[Issue Secure Single-Use Invites]
        A3 --> A4[Monitor Fleet & Resolve Incidents]
    end

    subgraph Bus Driver
        D1[Accept Invite & Open Mobile Browser] --> D2[Select Assigned Trip]
        D2 --> D3[Tap Start Trip & Grant GPS]
        D3 --> D4[Drive Safely: Auto GPS Broadcast]
        D4 --> D5[End Trip at Final Destination]
    end

    subgraph Parent
        P1[Accept Invite & Sign In] --> P2[Select Linked Child]
        P2 --> P3[View Real-Time Bus Location & Stop ETA]
        P3 --> P4[Receive 500m Proximity Alert]
    end

    D3 -. Location Points .-> S[(Central Server)]
    S -. Live Coordinates .-> A4
    S -. Authorised Stream .-> P3
    S -. 500m Geofence Alert .-> P4
```

### 1. School Admin
- Configures school profile, timezone (default `Asia/Kolkata`), buses, routes, ordered stops, and students.
- Generates expiring invitation links linking parents to specific student profiles.
- Monitors live fleet status (Active/Fresh, Active/Stale, Offline, Delayed, Incident).
- Issues broadcast delay/cancellation notices with live preview.

### 2. Bus Driver
- Accesses simple, mobile-optimized interface with zero typing required while moving.
- Reviews "Today's Assigned Trips", verifies GPS permission, and taps **Start Trip** while parked.
- Background/foreground GPS watch stream sends coordinates to `/api/trips/[id]/locations`.
- Reports quick one-tap incident presets (Traffic Delay, Breakdown, Route Issue, Emergency).
- Taps **End Trip** upon safe completion, immediately terminating tracking.

### 3. Parent
- Securely logs in and toggles between multiple linked children (if applicable).
- Sees only the child's assigned route, assigned stop, vehicle marker, and tracking freshness badge.
- Receives in-app notifications when trip begins, bus enters 500m radius, delays occur, or trip ends.

---

## 5. System Architecture & Tech Stack

```
┌────────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER (Next.js PWA)                      │
│   ┌─────────────────────┬─────────────────────┬────────────────────┐   │
│   │   Admin Dashboard   │   Driver Console    │   Parent Portal    │   │
│   │   (Desktop/Tablet)  │   (Mobile Browser)  │   (Mobile/PWA)     │   │
│   └─────────────────────┴─────────────────────┴────────────────────┘   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ HTTPS / REST / WebSockets
┌───────────────────────────────────▼────────────────────────────────────┐
│                       SERVER LAYER (Next.js API)                       │
│   ┌─────────────────────┬─────────────────────┬────────────────────┐   │
│   │  Auth & Permissions │  GPS Ingestion Pipe │ Geofencing Engine  │   │
│   │  (School Tenant)    │  (Rate & Bounds)    │ (Haversine/Hyst)   │   │
│   ├─────────────────────┼─────────────────────┼────────────────────┤   │
│   │  Trip State Machine │ Notification Router │ Audit Event Logger │   │
│   └─────────────────────┴─────────────────────┴────────────────────┘   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Database Queries & RLS
┌───────────────────────────────────▼────────────────────────────────────┐
│                         DATA LAYER (PostgreSQL)                        │
│   ┌─────────────────────┬─────────────────────┬────────────────────┐   │
│   │  Multi-Tenant Core  │  Trip Locations     │  Notifications &   │   │
│   │  (Schools, Users)   │  (Time-Series GPS)  │  Trip Events       │   │
│   └─────────────────────┴─────────────────────┴────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

- **Runtime & Web Framework:** Next.js (App Router) + React + TypeScript (Strict Mode).
- **Database:** PostgreSQL (with Row-Level Security and indexed geospatial/time-series queries).
- **Map Integration:** Leaflet / OpenStreetMap or Mapbox/MapLibre (pluggable map provider interface).
- **Realtime Transport:** Server-Sent Events (SSE) or WebSockets with robust polling fallback.
- **Validation:** Zod schema enforcement across all client inputs, API payloads, and environment variables.
- **Testing:** Vitest / Jest (Unit & Integration) + Playwright (E2E flows).

---

## 6. GPS Ingestion & Geofencing Engine

### 1. Ingestion Pipeline
1. Driver taps `Start Trip` $\rightarrow$ Server marks trip `active` and opens location stream.
2. Driver phone calls `navigator.geolocation.watchPosition({ enableHighAccuracy: true, timeout: 10000, maximumAge: 0 })`.
3. Browser pushes location payload every ~5–10 seconds:
   ```json
   {
     "tripId": "uuid",
     "latitude": 28.6139,
     "longitude": 77.2090,
     "accuracy": 12.4,
     "timestamp": 1728555000000
   }
   ```
4. Server verifies:
   - Authenticated user is the **assigned driver** for this active trip.
   - Latitude/Longitude within valid geospatial ranges ($-90 \le \text{lat} \le 90$, $-180 \le \text{lon} \le 180$).
   - Accuracy threshold $< 50\text{ m}$ (reject or flag coarse/inaccurate updates).
5. Latest point is saved to `trip_locations` and broadcast to authorized clients.

### 2. 500m Proximity Geofencing with Hysteresis
To prevent flickering alerts due to GPS jitter, the system uses a **dual-boundary hysteresis model**:

$$\text{Distance } d = 2R \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta\phi}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\Delta\lambda}{2}\right)}\right)$$

```
        Outside (> 600m)
              │
              ▼  (Bus approaches)
     [ Entry Threshold: 500m ] ──► TRIGGER ALERT (Idempotent: trip_id + stop_id)
              │
        Inside (< 500m)
              │
              ▼  (Bus leaves)
     [ Exit Threshold: 600m ]  ──► RESET GEOFENCE STATE
              │
        Outside (> 600m)
```

- **Idempotency Guarantee:** A unique compound key `trip_id:stop_id:proximity_500m` ensures parents receive **at most one** proximity notification per trip.

### 3. GPS Freshness & Degradation Thresholds

| State Badge | Duration Since Last Valid Point | Visual Indicator | System Action |
|---|---|---|---|
| 🟢 **Fresh** | $< 30\text{ seconds}$ | Green Pulse | Normal live map display |
| 🟡 **Stale** | $30 - 90\text{ seconds}$ | Amber Badge | "Live tracking updating slowly..." |
| 🔴 **Offline / Degraded** | $> 90\text{ seconds}$ | Red Badge | "Tracking temporarily unavailable" |

---

## 7. Critical Browser GPS Limitations

> [!WARNING]
> **Mobile Browser Geolocation Behavior & Honest Product Disclosure:**
> Mobile web browsers (Chrome on Android, Safari on iOS) are subject to aggressive OS-level power management. When the browser tab is backgrounded, another app opened, or the phone screen locked:
> 1. JavaScript timers and network requests may be throttled or completely suspended.
> 2. `navigator.geolocation` events may pause until the driver wakes the phone and re-opens the active tab.

### Operational Countermeasures:
- **Driver Instructions:** The Driver UI explicitly reminds the driver to keep the phone screen active on a dashboard mount and connected to vehicle power.
- **Wake Lock API:** Implements `navigator.wakeLock.request('screen')` where supported to prevent phone sleep during active trips.
- **Honest Status Display:** The UI never freezes the last-known coordinate and pretends the vehicle is live. It shows:
  *"Last updated 4 minutes ago — Signal paused"* with clear visual cues.
- **Upgrade Path:** For institutions demanding guaranteed locked-screen tracking, a dedicated native wrapper or OBD-II hardware GPS integration is planned post-MVP.

---

## 8. Notification System & Event Matrix

| Event | Trigger Condition | Recipient(s) | Channel |
|---|---|---|---|
| **Trip Started** | Driver transitions trip to `active` | Parents on assigned trip | In-App + Push |
| **Stop Proximity (500m)** | Valid bus coordinate crosses $500\text{m}$ to stop | Linked parents for that stop | In-App + Push (High Priority) |
| **Stop Arrival** | Bus enters stop arrival radius ($<100\text{m}$) | Linked parents for that stop | In-App Status |
| **Trip Delayed** | Admin flags delay or schedule threshold exceeded | Affected parents & Admin | In-App + Banner |
| **Trip Cancelled** | Admin cancels trip with reason | All assigned parents & Driver | In-App + Push |
| **GPS Signal Stale** | No valid update for $>30\text{ seconds}$ | Admin Dashboard Queue | Operational Alert |
| **Route Deviation** | Vehicle departs route corridor $>3\text{ consecutive points}$ | Admin Dashboard Queue | Warning Alert |
| **Vehicle Breakdown** | Driver reports breakdown from phone | Admin + Neutral Parent Notice | Immediate High-Priority |
| **Emergency Reported** | Driver presses Emergency button | School Admin Dashboard | Critical Incident Banner |
| **Trip Completed** | Driver taps End Trip | All assigned parents | Trip Summary Card |
| **School Notice** | Admin broadcasts message to route | Parents on specified route | In-App Notice |

---

## 9. Database Schema & Data Models

All primary tables enforce multi-tenant separation with `school_id` foreign keys and strict database constraints.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                               RELATIONAL OVERVIEW                                │
│                                                                                  │
│   schools ──┬── profiles ── school_memberships                                   │
│             ├── invitations                                                      │
│             ├── buses                                                            │
│             ├── drivers                                                          │
│             ├── students ──┬── parent_student_links                              │
│             │              └── student_route_assignments                         │
│             ├── routes ────┬── stops                                             │
│             │              └── trip_templates                                    │
│             └── trips ─────┬── trip_locations                                    │
│                            ├── trip_events                                       │
│                            └── notifications                                     │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Core Schema Highlights:
- **`schools`**: Organization profile, timezone, contact details.
- **`profiles`** & **`school_memberships`**: RBAC with roles (`admin`, `driver`, `parent`) and statuses (`active`, `pending`, `revoked`).
- **`invitations`**: Single-use cryptographic tokens with expiration timestamps.
- **`routes`** & **`stops`**: Ordered sequence of coordinates with configurable `arrival_radius_m`.
- **`students`** & **`parent_student_links`**: Explicit mapping preventing cross-child visibility.
- **`trips`**: State machine (`scheduled` $\rightarrow$ `active` $\rightarrow$ `completed` / `cancelled` / `delayed`).
- **`trip_locations`**: Time-stamped GPS coordinates with accuracy, speed, and heading.
- **`trip_events`**: Idempotent state transitions and audit logs.
- **`notifications`**: Recipient-scoped notification delivery queue with read/unread tracking.

---

## 10. API Specification & Realtime Channels

### Authentication & Memberships
- `GET /api/me`: Returns current user identity, active school profile, and granted role.
- `POST /api/invitations/accept`: Validates token, links user to school and intended students.

### Fleet & Route Management (Admin Only)
- `POST /api/admin/schools/settings`: Update timezone, branding, and contact details.
- `GET|POST|PUT|DELETE /api/admin/buses`: Manage fleet vehicles.
- `GET|POST|PUT|DELETE /api/admin/routes`: Manage routes and ordered stop pins.
- `GET|POST|PUT|DELETE /api/admin/students`: Manage student directory and stop assignments.
- `POST /api/admin/invitations`: Dispatch parent and driver invites.

### Trip Execution & GPS (Driver)
- `GET /api/driver/trips/today`: Retrieve assigned schedules.
- `POST /api/trips/[id]/start`: Activate trip and initiate GPS ingestion window.
- `POST /api/trips/[id]/locations`: High-frequency endpoint for GPS coordinate ingestion.
- `POST /api/trips/[id]/incidents`: Log quick delay, breakdown, or safety incidents.
- `POST /api/trips/[id]/end`: Safely terminate trip and lock location ingestion.

### Live Tracking & Notifications (Parent & Admin)
- `GET /api/trips/[id]/live`: Stream or poll latest coordinate, accuracy, and freshness status.
- `GET /api/notifications`: Retrieve user's in-app notification history.
- `POST /api/notifications/[id]/read`: Acknowledge notification item.

---

## 11. Security, Multi-Tenancy & Child Safety

1. **Multi-Tenant Isolation:** Every table query is scoped by `school_id`. Users from School A can never query records from School B, even with direct ID access (IDOR protection).
2. **Strict Parental Privacy:** Parents have zero access to the broader student directory, other parents, or buses unassigned to their registered children.
3. **No Driver Stalking:** Driver location is queried and broadcast **strictly** during active scheduled trips. No background tracking exists outside trip windows.
4. **Data Minimization:** Student records require only minimal identifiers (e.g., First Name, Grade). No unnecessary personal data is stored or logged.
5. **No False Safety Assertions:** The application UI includes an explicit, prominent safety disclaimer:
   > *"Vehicle proximity does not certify student boarding or physical attendance."*

---

## 12. Comprehensive Phased Implementation Plan

```mermaid
gantt
    title Master Build Plan (Phases 0 - 7)
    dateFormat  YYYY-MM-DD
    section Phase 0
    System Blueprint & Setup Checklist :active, p0, 2026-10-10, 1d
    section Phase 1
    Foundation, Multi-Tenant Auth & DB  :p1, after p0, 2d
    section Phase 2
    Admin Fleet, Routes & Invitations  :p2, after p1, 2d
    section Phase 3
    Trip Management & Driver Console   :p3, after p2, 2d
    section Phase 4
    Real GPS Ingestion & Live Map Pipe :p4, after p3, 2d
    section Phase 5
    500m Geofencing & Alert Dispatcher :p5, after p4, 2d
    section Phase 6
    Parent Portal & Live Control Room  :p6, after p5, 2d
    section Phase 7
    Hardening, Pilot Testing & Polish  :p7, after p6, 2d
```

### Detailed Phase Milestones:

- [ ] **Phase 0 — Repository Inspection & Architecture Blueprint**
  - Verify environment, tools, and repo state.
  - Finalize `PRD.md`, `README.md`, and technical specifications.
  - Define folder hierarchy and schema migrations strategy.

- [ ] **Phase 1 — Core Foundation, Database & Multi-Tenant Auth**
  - Initialize Next.js project with TypeScript strict mode.
  - Configure PostgreSQL database schema, migrations, and seeds.
  - Implement role-based access control and tenant verification middleware.
  - Prevent unauthorized admin self-assignment.

- [ ] **Phase 2 — School Onboarding, Fleet Management & Invites**
  - Admin CRUD for buses, drivers, routes, stops, and students.
  - Implement cryptographic single-use invitation system.
  - Validate parent-to-student relationship integrity.

- [ ] **Phase 3 — Trip Lifecycle & Driver Mobile Experience**
  - Recurring trip templates and daily scheduled trip instances.
  - Driver mobile trip console with touch-optimized controls.
  - Trip state machine (`scheduled` $\rightarrow$ `active` $\rightarrow$ `completed`).

- [ ] **Phase 4 — Real-World GPS Pipeline & Live Map Stream**
  - Driver `navigator.geolocation.watchPosition` client with Wake Lock.
  - Rate-limited coordinate ingestion API with bounds validation.
  - Real-time location stream and dynamic freshness badges.

- [ ] **Phase 5 — Geofencing Engine & Notification Dispatcher**
  - Server-side Haversine distance calculator.
  - 500m entry / 600m exit hysteresis geofence logic.
  - Idempotent in-app notification creation and broadcast.

- [ ] **Phase 6 — Parent Tracking Portal & Admin Control Room**
  - Mobile parent view: child switcher, live bus map, stop ETA.
  - Admin fleet live dashboard: active trips, stale GPS alerts, delay management.
  - Incident handling and resolution workflow.

- [ ] **Phase 7 — Hardening, Pilot Simulation & Real-Device Testing**
  - Automated unit and integration test coverage.
  - Real Android & iOS mobile browser testing over HTTPS.
  - Seed fictional pilot school with realistic routes and simulated bus trips.
  - Security audit and operational documentation.

---

## 13. Getting Started & Development Setup

### Prerequisites
- Node.js `20.x` or `22.x` (LTS)
- PostgreSQL `15+`
- Package manager: `npm` or `pnpm`

### 1. Clone & Install
```bash
git clone https://github.com/Two19Labs/bus.stop.git
cd "bus stop"
npm install
```

### 2. Environment Configuration
Copy `.env.example` to `.env.local`:
```bash
cp .env.example .env.local
```
Configure your database and app keys:
```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/bus_stop"
NEXTAUTH_SECRET="your-generated-super-secret-key"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXT_PUBLIC_DEFAULT_TIMEZONE="Asia/Kolkata"
```

### 3. Database Migrations & Fictional Seed
```bash
npm run db:migrate
npm run db:seed
```

### 4. Launch Development Server
```bash
npm run dev
```
Open [http://localhost:3000](http://localhost:3000) in your browser.

---

## 14. Real-Device Testing Protocol

Because this project relies on **browser-level mobile GPS**, testing solely in desktop browser emulators is insufficient.

1. **HTTPS or Localhost Tunnel:**
   - Geolocation APIs require a secure origin. In local development, use tools like `ngrok` or Cloudflare Tunnels to serve the application over HTTPS to your physical test phone.
2. **Android Chrome & iOS Safari Testing:**
   - Test foreground active trip tracking on real mobile hardware.
   - Verify permission prompts and location accuracy handling.
   - Verify screen-dimming and screen-lock behavior; document OS throttling honestly.
3. **Network Resilience Testing:**
   - Toggle phone to 3G / Airplane mode during an active trip.
   - Ensure the driver app queues and retries coordinates upon reconnect without crashing.
   - Ensure parent and admin maps display the amber **Stale** badge after 30 seconds of signal loss.

---

## 15. Acceptance Criteria & Definition of Done

The BusStop MVP is considered complete when:
- [x] Comprehensive PRD and Architecture documentation committed to source control.
- [ ] Admin can onboard a school, fleet, route, and stops without open school discovery.
- [ ] Parent invitations securely bind parents exclusively to their linked children.
- [ ] Driver starts trip from a mobile browser; coordinates stream live over HTTPS.
- [ ] Parents see their bus and assigned stop with honest freshness indicators.
- [ ] Server calculates 500m proximity and triggers exactly one alert per stop per trip.
- [ ] GPS signal degradation is accurately detected and visually flagged.
- [ ] Cross-school data leakage tests pass with 100% isolation.
- [ ] All automated unit, integration, and build checks pass without warnings.

---

*Engineered by [Two19 Labs](https://github.com/Two19Labs) — Built for reliable school transit visibility.*
