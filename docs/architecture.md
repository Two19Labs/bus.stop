# 🏛️ System Architecture & Engineering Blueprint

## 1. System Overview

**BusStop** is designed as a unified, mobile-first responsive web application serving three distinct user roles under strict multi-tenant school isolation:
1. **School Admin** (Fleet management, user onboarding, live operations command center)
2. **Bus Driver** (Mobile console, real-time GPS broadcast, incident reporting)
3. **Parent** (Linked child tracking, bus stop proximity alerts, live map)

---

## 2. Technology Stack & Architectural Decisions

| Layer | Selected Technology | Rationale |
|---|---|---|
| **Framework** | **Next.js (App Router) + TypeScript** | Full-stack React framework with SSR, API route handlers, strong typing, and built-in edge/node runtime flexibility. |
| **Styling & UI** | **Vanilla CSS Design System / Modern CSS Variables** | High-contrast, mobile-first, zero runtime bloat, accessible touch targets for mobile driving use. |
| **Database** | **PostgreSQL** | ACID-compliant relational database with strong foreign key enforcement, indexing, and geospatial capabilities. |
| **ORM / Data Access** | **Prisma / Kysely or Typed SQL Client** | Type-safe queries, migration history tracking, and strict tenant-scoping abstractions. |
| **Realtime Updates** | **Server-Sent Events (SSE) / WebSockets** with **Smart Polling Fallback** | Low battery consumption on mobile devices, instant map marker updates, resilient reconnects. |
| **Map Rendering** | **Leaflet / OpenStreetMap / Mapbox** | Lightweight, performant vector/raster map tiles with custom vehicle, route, and stop markers. |
| **Geolocation API** | **`navigator.geolocation.watchPosition`** | Native browser GPS without app store friction, paired with `navigator.wakeLock` to prevent screen sleep. |
| **Validation** | **Zod** | Runtime contract verification for all API requests, GPS payloads, and environment variables. |

---

## 3. Multi-Tenant Isolation & Least Privilege Model

Every school-owned record in the database is bound to a `school_id`.

```
                    ┌─────────────────────────┐
                    │      Authenticated      │
                    │       User Session      │
                    └────────────┬────────────┘
                                 │
             ┌───────────────────┴───────────────────┐
             ▼                                       ▼
    ┌─────────────────┐                     ┌─────────────────┐
    │  School Member  │                     │   Target Entity │
    │    School ID    │                     │    School ID    │
    └────────┬────────┘                     └────────┬────────┘
             │                                       │
             └───────────────► MATCH? ◄──────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
                [ ALLOW ]                   [ DENY 403 ]
```

### Authorization Rules:
- **School Admins:** Restricted to reading and mutating records belonging strictly to their verified `school_id`.
- **Bus Drivers:** Can only view trips assigned to their `user_id` and can only upload GPS points to active trips where they are the designated driver.
- **Parents:** Can only read locations and alerts for trips associated with their explicitly linked `student_id` and assigned stop.

---

## 4. GPS Tracking Pipeline & Hysteresis Geofencing

```
[Driver Phone Browser]
       │
       │ navigator.geolocation.watchPosition (5-10s interval)
       │ + navigator.wakeLock.request('screen')
       ▼
[POST /api/trips/:id/locations]
       │
       ├──► 1. Verify Driver Assignment & Active Trip Status
       ├──► 2. Validate Coordinate Bounds & Accuracy (<50m)
       ├──► 3. Append to `trip_locations` & Update Latest Cache
       │
       ▼
[Server-Side Geofence Evaluation Engine]
       │
       ├── For each stop on the route:
       │     Calculate Haversine Distance: d(Bus, Stop)
       │
       ├── If d <= 500m AND state == OUTSIDE:
       │     ├── State := INSIDE
       │     └── Trigger Proximity Event (Idempotent: trip_id + stop_id)
       │           └── Dispatch In-App Notification to linked parents
       │
       └── If d >= 600m AND state == INSIDE:
             └── State := OUTSIDE (Hysteresis reset for jitter prevention)
```

---

## 5. Mobile Browser GPS Behavior & Limitation Mitigation

| Constraint | Physical Behavior | Software Countermeasure |
|---|---|---|
| **Tab Backgrounding** | Mobile OS throttles JS execution & location polling. | UI displays prominent warning: *"Keep this screen active while driving."* |
| **Screen Lock** | Browser may freeze `watchPosition` updates. | Implements Screen Wake Lock API (`navigator.wakeLock`). |
| **GPS Signal Loss** | Driver enters tunnel or underground area. | Server detects update gap $>30\text{s}$ and updates status to `Stale` (amber badge). If $>90\text{s}$, transitions to `Degraded/Offline` (red badge). |
| **Battery Saver Mode** | OS delays network transmissions. | Batched coordinate queuing with client-recorded timestamps to reconstruct paths accurately upon reconnection. |

---

## 6. Directory Structure Blueprint

```
bus-stop/
├── docs/
│   ├── architecture.md
│   ├── gps-and-location.md
│   └── security-and-privacy.md
├── src/
│   ├── app/                      # Next.js App Router
│   │   ├── (auth)/               # Sign in, accept invite, password reset
│   │   ├── (dashboard)/
│   │   │   ├── admin/            # Fleet, routes, stops, students, invites, live map
│   │   │   ├── driver/           # Assigned trips, active GPS broadcast console
│   │   │   └── parent/           # Child switcher, live bus map, stop proximity
│   │   └── api/                  # REST & SSE endpoints
│   ├── components/               # Reusable UI & design system
│   │   ├── ui/                   # Buttons, badges, modals, cards
│   │   ├── maps/                 # Leaflet map container, bus marker, stop radius
│   │   └── layout/               # Role-based headers, navigation
│   ├── lib/                      # Core utilities
│   │   ├── auth/                 # Session & membership resolution
│   │   ├── db/                   # Database client & connection pool
│   │   ├── geofencing/           # Haversine distance, hysteresis, idempotency
│   │   └── validation/           # Zod schemas
│   └── types/                    # Shared TypeScript interfaces
├── prisma/ or migrations/        # Database schema & migrations
├── PRD.md                        # Master Product Requirements Document
└── README.md                     # Comprehensive Project Guide & Setup
```
