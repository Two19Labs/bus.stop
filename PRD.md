# School Bus Tracking SaaS — MVP Product Requirements Document
**Document status:** Draft for implementation  
**Audience:** Founder, product/design, and Claude Code  
**Version:** 0.1  
**Last updated:** 10 October 2026  
**Working product name:** TBD

> This document defines the MVP to build. Where a technical choice is open, Claude Code should inspect the repository, select a sensible, well-supported approach, document the decision, and continue without inventing unavailable credentials or pretending an integration works.

---

## 1. Product summary

Build a mobile-first, responsive **web application** that lets a school manage bus routes and trips, lets authorized drivers share a bus's live location during an active trip, and lets authorized parents view the relevant bus and receive useful arrival/status alerts.

The product has three roles:
1. **School Admin** — configures the school, users, buses, routes, stops and schedules; monitors active trips and responds to operational alerts.
2. **Bus Driver** — sees assigned trips, starts/ends trips, shares location during an active trip, and reports an incident or delay.
3. **Parent** — sees only the bus/trip associated with their school-linked child or children, receives trip and proximity notifications, and can see tracking freshness and ETA.

All roles use the same responsive web app. The app's UI and permissions change by role. Do not build separate native mobile apps for the MVP.

### Product promise
Give parents a trustworthy answer to “Where is the bus, and when will it reach our stop?” while giving the school a simple view of its active fleet and exceptions.

### Non-goals
The MVP is not a full school ERP, payment system, CCTV system, driver-behaviour scoring product, hardware/GPS-device management platform, route-optimization engine, or student attendance system.

**Important scope choice:** The MVP tracks bus arrival only. It does **not** claim to verify that a particular child boarded or was dropped off. No student-level pickup/drop-off confirmation, QR/RFID scanning, or “child is safely on the bus” status in this release.

---

## 2. Problem and target users

### Problems
- Parents lack a reliable, timely view of their child's bus location and arrival estimate.
- Schools may receive repetitive calls/messages asking where a bus is.
- Admins need one view of active buses, delayed trips, stale GPS and reported incidents.
- Drivers need a very simple way to start a trip and share location without messaging every parent.

### Initial target customer
A school with a small-to-medium bus fleet that wants a straightforward, low-friction tracking experience. The MVP should be demonstrable to one pilot school; multi-tenant design is required even if the first deployment has only one school.

### Success metrics for a pilot
Instrument or manually measure:
- Time for an admin to configure one school, route, bus and users.
- Percentage of scheduled trips started and ended correctly.
- Percentage of active trip time with fresh GPS updates.
- Time between a geofence threshold being crossed and the alert being created.
- Duplicate/false proximity alert rate.
- Parent weekly active usage and usefulness feedback.
- Admin-reported reduction in “where is the bus?” calls.
- Number of unresolved GPS/route incidents.

Do not invent target results. Provide analytics/events so the pilot can establish a baseline.

---

## 3. Product principles

1. **School-controlled access:** A user cannot browse a directory of schools and join any school. School admins invite/onboard users and explicitly link parents to student records.
2. **Least privilege:** Every API query and mutation must enforce school and role access server-side. Hiding a button is not authorization.
3. **Freshness over false certainty:** Clearly show when a location is stale or unavailable. Never present an old coordinate as live.
4. **Events, not notification spam:** GPS updates update the map; only meaningful state changes produce push/in-app notifications.
5. **Server is authoritative:** Trip status, permissions, geofence transitions, deduplication and notification decisions are calculated/validated on the server.
6. **Privacy by default:** Share driver location only during an active trip; expose location only to authorized users who need it.
7. **Build for phone screens first:** Large controls, clear status labels, good contrast and minimal typing. The driver must not need to interact with the phone while driving.
8. **Graceful degradation:** GPS, browser permissions, network, push delivery and map tiles can fail. Explain the state and preserve last-known information with a timestamp.
9. **Honest product claims:** Bus location does not prove student boarding, child identity, or safety. The UI must not imply that it does.

---

## 4. Roles, authentication and onboarding

### 4.1 Authentication
- Use secure email/password authentication or the auth method already supported by the selected stack.
- If a one-time code or magic link is used, it must be implemented by a real provider and documented.
- Include sign out, password reset, session handling and a clear unauthorized/pending state.
- Never store plaintext passwords.
- Keep secrets server-side; never expose service-role keys or private API keys to the browser.

### 4.2 Role selection
A new user may choose a requested role: Parent, Bus Driver, or School Admin. Role choice is a **request**, not an entitlement.
- A person cannot become a school admin simply by selecting the role.
- The initial school admin account should be created through a controlled bootstrap process (environment variable/seed script/owner-only setup flow) or a secure invite generated by the platform owner.
- Admins onboard/invite drivers and parents.
- Unapproved users see a pending/invalid-invitation state and cannot access school data.
- A user cannot self-assign a school by entering a school name.

### 4.3 School-admin onboarding
Admin creates/configures:
- School name, timezone (default `Asia/Kolkata`), optional logo/contact details.
- Buses and optional registration/identifier.
- Drivers and their accounts.
- Routes and ordered stops, each stop having a name, address/description, latitude and longitude.
- Trip templates/schedules, including direction (pickup/drop-off), days, planned departure and expected arrival windows.
- Students (minimal data only) and their assigned route/stop.
- Parent accounts and parent-to-student links.
- Driver-to-bus/trip assignments.

Admin may invite a parent/driver by a secure, single-use, expiring invitation link/code. For a pilot, a manually provisioned user is acceptable if the flow is secure and documented. Invitations must be scoped to one school and one intended role; parent invitations should be linked to the intended student(s) where possible. Do not reveal whether a child exists to an unauthenticated person.

### 4.4 Parent onboarding
- Parent follows a school-issued invitation and authenticates.
- Parent sees only the child/children explicitly linked by the admin.
- Parent may not search all students or routes.
- Parent sees assigned stop and relevant active/upcoming trips.
- If a parent has multiple linked children, allow switching between them.
- Parent access can be revoked by the admin; revoked users lose access immediately on subsequent requests/session refresh.

### 4.5 Driver onboarding
- Driver follows a school-issued invitation or admin-created account.
- Driver sees only assigned trips and the minimum necessary route/stop details.
- Driver can start a trip only if assigned and the trip is in a valid state.
- Admin can deactivate/reassign a driver.

### 4.6 Multi-tenancy
Every school-owned record must include `school_id` (or have a guaranteed parent relation that scopes it). Every read/write must verify that the authenticated user belongs to the relevant school and has permission. Add tests for cross-school data leakage.

---

## 5. Core user journeys

### 5.1 Admin setup
1. Admin signs in.
2. Admin creates/edits school profile.
3. Admin adds bus and driver.
4. Admin creates route and ordered stops with map pin placement/search or manual coordinate entry.
5. Admin creates scheduled trips and assigns bus/driver.
6. Admin creates student records and assigns each to a route and stop.
7. Admin invites parents and links each parent to student record(s).
8. Admin sees setup completeness and can launch a test trip.

### 5.2 Driver trip
1. Driver signs in on a phone browser.
2. Driver sees “Today's assigned trips”.
3. Driver opens a trip and sees route, stops and current status.
4. Before starting, app checks secure context (HTTPS in production), geolocation support/permission, and assignment.
5. Driver taps **Start trip** while safely stopped.
6. Server transitions trip from `scheduled` to `active`, records actual start, and starts accepting location updates for that trip.
7. App requests browser geolocation updates and posts valid coordinates to the backend.
8. Driver sees tracking state: Sharing location / Weak signal / Permission needed / Network unavailable.
9. Driver can report a breakdown, delay or emergency while safely stopped. Emergency UI should clearly instruct the driver to stop safely and contact local emergency services where appropriate; this app is not an emergency-response service.
10. Driver taps **End trip** after stopping safely. Server closes the trip and tracking stops.

### 5.3 Parent tracking
1. Parent signs in and selects linked child (if multiple).
2. Parent sees upcoming/active trip, assigned stop, bus status and last-updated time.
3. If active and location fresh, parent sees bus on a map and an ETA if enough route/location data exists.
4. When bus crosses the configured proximity threshold to that child's assigned stop, server generates one alert for that stop/trip.
5. Parent sees trip updates, delay/cancellation notices, and “location temporarily unavailable” when applicable.
6. Parent can open notification history and contact the school using configured contact details.

### 5.4 Admin live operations
1. Admin opens dashboard.
2. Dashboard shows active trips and bus status (active/fresh, active/stale, offline, delayed, incident).
3. Admin opens a bus/trip to see route, last coordinate, timestamp, accuracy, current status and event timeline.
4. Admin acknowledges operational alerts and records a note/action.
5. Admin can send a school-authored notice to affected parents, with a preview and confirmation step.
6. Admin can cancel a trip or mark it delayed with a reason; parents assigned to the trip receive an appropriate notification.

---

## 6. Functional requirements by role

### 6.1 Shared
- Responsive layout for mobile, tablet and desktop.
- Authenticated routes and role-specific navigation.
- School branding/name visible so users know which school account they are in.
- Loading, empty, error, permission-denied, offline and stale-data states.
- In-app notification centre with read/unread state.
- Notification preferences where feasible, but safety/operational notices must not be silently disabled if the product defines them as mandatory.
- Timestamps shown in the school's configured timezone.
- Accessible labels, keyboard navigation for admin pages, sensible contrast and screen-reader-friendly status text.

### 6.2 Admin
- School settings and contact information.
- CRUD for buses, drivers, students, parent accounts, routes, stops and trip schedules.
- Assign bus and driver to a trip.
- Assign student to a route/stop; link parent(s) to student.
- Invite, resend invite, revoke invite and deactivate user.
- Live map of active buses, with status and last update time.
- Filter by route/status and open trip details.
- Alerts list with severity, timestamp, source, status, acknowledgement and resolution note.
- Trip history and event timeline.
- Manual delay/cancellation notice to affected parents.
- Audit log for security- and safety-relevant actions.
- Prevent deletion of referenced records that would break trip history; use archive/deactivate where appropriate.

### 6.3 Driver
- Today's and upcoming assigned trips.
- Trip details, route and stop list.
- Start trip, end trip, and report incident.
- Current location-sharing status and latest successful server acknowledgement.
- Visible warning if permission is denied, GPS is unavailable, location accuracy is poor, or updates cannot be uploaded.
- No manual editing of GPS coordinates.
- No student-level pickup/drop-off buttons in the MVP.
- No interaction that requires typing while moving. Add a clear reminder to use the app only while safely stopped.

### 6.4 Parent
- Linked children list and child switcher.
- Upcoming and active trip status.
- Live bus map, last update time and freshness badge.
- Assigned stop and ETA/arrival estimate when calculable.
- Arrival/proximity alert history.
- Trip started, delay, cancellation, location unavailable and trip ended notices.
- School contact details and admin-issued notices.
- No access to other parents, students, routes, drivers' personal details, or buses unrelated to the linked child.

---

## 7. GPS and location architecture — implementation requirements

Claude Code must research and implement the most reliable approach supported by the chosen web stack. Do not hand-wave GPS as “add a map.” Location capture, server ingestion, geofencing and notification logic are core functionality.

### 7.1 Browser geolocation limitations
This is a responsive web app, not a native app. Use the browser Geolocation API (`navigator.geolocation.watchPosition`) while a driver has an active trip and the app is open/active. Production must use HTTPS (localhost is usually allowed for development).

**Important limitation:** Mobile browsers can suspend JavaScript and geolocation when the tab is backgrounded, the screen is locked, the browser is killed, or the operating system applies battery restrictions. A web app cannot guarantee uninterrupted, native-app-like background tracking on every device. The MVP must:
- Test on actual Android phones and the target browsers.
- Keep a visible tracking state.
- Detect stale location and report it.
- Explain that the driver should keep the trip screen active and phone powered/connected during the pilot.
- Never claim tracking is guaranteed when the browser is suspended.
- Document the limitation and a later native-app/managed-device upgrade path if schools require dependable locked-screen tracking.

Do not attempt to bypass OS permissions or browser restrictions.

### 7.2 Capture flow
When an authorized driver starts an assigned trip:
1. Validate trip assignment and state on the server.
2. Request geolocation permission.
3. Use `watchPosition` with high accuracy where supported; use a configurable `maximumAge` and timeout.
4. Capture latitude, longitude, accuracy in metres, browser timestamp and trip ID.
5. Validate payload shape and plausible coordinate bounds on the server.
6. Reject or flag stale, impossible or low-quality points; do not let a client choose another driver's trip ID.
7. Store the latest valid point and append a location-history point at a sensible interval.
8. Broadcast or make the latest location available to authorized admin/parent clients.
9. Server evaluates geofence transitions and stale-location state.
10. Stop the watch and reject further updates when the trip ends, is cancelled, or driver assignment is revoked.

Initial tuning targets (must be configurable and tested):
- Location request target: about every 5–10 seconds during an active trip, subject to browser/OS scheduling and battery.
- Fresh location: updated within 30 seconds.
- Stale warning: no valid update for 30 seconds.
- Offline/degraded status: no valid update for 90 seconds.
These are starting values, not guarantees. Clearly distinguish `stale` from `offline/degraded`.

### 7.3 Storage and transport
Choose a reliable transport supported by the selected backend:
- Prefer managed realtime subscriptions/WebSockets for live map updates if already available in the stack.
- Polling at a sensible interval is acceptable for an MVP if it is robust and documented.
- Never expose privileged database keys in the browser.
- Validate every location write against authenticated user, driver assignment, active trip and school.
- Add rate limits and payload validation.
- Use server timestamps for received time; retain client timestamp separately for diagnosing delays.
- Keep location history retention configurable and minimal.

### 7.4 Map display
- Use a real map provider/library that is appropriate for the selected stack and deployment.
- Read its current terms, API-key restrictions and billing limits; document setup in `.env.example` and README.
- Never commit API secrets. Browser-restricted public map keys may be used only as intended by the provider.
- Show the bus marker, assigned stop(s), route line if available, and a “last updated” label.
- If map tiles or coordinates fail, show a useful fallback status and timestamp rather than a blank page.
- Do not expose all school routes or all buses to parents.

### 7.5 Geofencing and 500-metre alert
The proximity alert is based on the bus's distance from the **child's assigned stop**, not the parent's current phone location.

For each active trip and eligible parent/student-stop link:
1. Calculate distance server-side between the latest valid bus coordinate and the assigned stop coordinate using a geospatial distance calculation (Haversine is acceptable for the MVP).
2. Only evaluate if location is fresh enough and accuracy is within a configured reasonable limit. If accuracy is too poor, delay the trigger or mark it uncertain instead of asserting exact proximity.
3. Trigger when distance crosses from outside to inside the configured radius (default 500 metres).
4. Create one idempotent event keyed by trip + stop + event type (and student/assignment where needed).
5. Notify only parents linked to students assigned to that stop and that trip.
6. Do not send repeated alerts while the bus remains inside the radius.
7. Reset the geofence only after the bus is outside a larger exit radius or a new trip begins, to avoid GPS jitter. Use hysteresis, for example entry at 500 m and exit at 600 m, configurable.
8. Do not fire an alert for old/stale points, invalid coordinates or a trip that is not active.
9. Store event timestamp, measured distance, reported accuracy and location point reference for debugging.

The UI wording should be “Bus is approximately 500 m from your stop” or “Bus is approaching your stop,” not an exact arrival guarantee.

### 7.6 ETA
- MVP ETA may use route progress and recent speed/distance only if enough valid points exist.
- If no credible ETA can be calculated, show “ETA unavailable” instead of inventing a number.
- Do not infer road travel time from straight-line distance alone and present it as precise.
- A routing provider may be added if the API key/terms/cost are configured. Keep the routing integration behind a small service/module so it can be swapped.
- Clearly label ETA as an estimate.

### 7.7 Route deviation
For MVP, use the planned route geometry if available, or a defined corridor/sequence of stops with a configurable tolerance. Do not flag every ordinary GPS drift as a deviation.
- Trigger only after a bus is outside the route tolerance for a sustained duration or multiple consecutive valid points.
- Ignore poor-accuracy points.
- Create one alert per deviation episode and suppress duplicates until the bus returns within the route corridor.
- Default alert severity should be “warning,” not “emergency.”
- Admin can acknowledge and resolve it.

### 7.8 GPS failure and stale states
- `fresh`: recent valid update inside freshness threshold.
- `stale`: last update exceeded freshness threshold.
- `offline/degraded`: update exceeds the longer outage threshold or the app reports upload failure.
- `permission_required`: browser permission is denied/not yet granted.
- `gps_unavailable`: browser reports position unavailable.
- `trip_ended`: tracking is not expected.
Show these states consistently to driver, admin and parent with role-appropriate wording.

---

## 8. Notification system

### 8.1 Delivery channels
Implement:
1. **In-app notifications** persisted in the database (required).
2. **Browser Web Push** if the selected stack/provider supports it cleanly and the app is served over HTTPS, with permission opt-in and service worker.
3. If push cannot be completed in the MVP, make in-app notifications fully functional and clearly document that background push is not enabled. Do not fake push delivery.
4. SMS/WhatsApp are out of scope unless explicitly configured later; do not claim these integrations exist.

### 8.2 Notification matrix

| Event | Trigger | Parent | Admin | Driver |
|---|---|---|---|---|
| Trip started | Driver successfully starts assigned trip | Notify parents on that trip | Fleet status updates; optional in-app event | Confirmation |
| Stop proximity | Valid bus point crosses stop geofence entry threshold | Notify linked parents for that stop/trip once | Visible event only if desired | No notification by default |
| Bus arrived at stop | Valid point enters tighter stop radius / stop arrival logic | In-app arrival update if enabled; avoid duplicate with proximity | Trip/stop status updates | Optional status |
| Trip delayed | Admin marks delay or server detects configured schedule threshold | Notify affected parents | Dashboard alert | Notify driver only if operationally useful |
| Trip cancelled | Admin cancels trip with reason | Notify all parents assigned to trip | Audit/event record | Notify assigned driver |
| GPS stale | No valid update for configured period | Show tracking unavailable; notify only if duration merits it | Alert | Driver sees tracking warning if app active |
| Route deviation | Sustained valid deviation from route | Notify only if school policy says it affects parents; default admin-only | Warning alert | Driver warning only while safe/stopped or non-distracting |
| Breakdown | Driver reports breakdown | Notify affected parents with neutral delay notice after admin action or configured policy | High-priority alert | Confirmation |
| Emergency | Driver reports emergency | Admin immediate high-priority alert; parent communication follows configured policy | High-priority alert and incident record | Confirmation |
| Trip ended | Driver ends trip | Trip completion notice | Trip marked complete | Confirmation |
| School notice | Admin sends message to selected trip/route | Selected parents | Audit record | If relevant |

### 8.3 Notification content and deduplication
- Every notification has type, recipient, school, related trip/event, created time, read time and delivery status.
- Notification delivery is asynchronous/retryable where possible.
- Use an idempotency key so retried GPS processing cannot send duplicate alerts.
- Keep copy concise and actionable.
- Do not include unnecessary student personal data on lock screens.
- Do not describe a GPS-derived event as confirmed child safety or boarding.
- For a driver-reported emergency, show the admin the incident, location timestamp and last known coordinates; do not promise dispatch or emergency-service response.

Suggested copy:
- Trip started: “Your school bus has started its trip.”
- Proximity: “Your bus is approximately 500 m from your stop.”
- Stale: “Live tracking is temporarily unavailable. Last updated at {time}.”
- Delay: “Your bus is delayed. The school will share updates when available.”
- Cancellation: “Today's bus trip has been cancelled. Reason: {reason}.”
- Trip ended: “The bus trip has ended.”
- Incident: “The school has reported a transport issue affecting this trip. Please check for updates.”

### 8.4 Notification preferences
Parents may control non-critical informational preferences if implemented. They must not be able to subscribe to unrelated routes or another student's alerts. Admin policy controls which operational alerts are sent.

---

## 9. Data model (suggested; adapt to the chosen database)

Use migrations/schema versioning. Add indexes for common filters and foreign keys. UUIDs or provider-generated opaque IDs are preferred.

### `schools`
- `id`
- `name`
- `timezone` (default `Asia/Kolkata`)
- `logo_url` nullable
- `contact_phone` nullable
- `contact_email` nullable
- `created_at`, `updated_at`, `status`

### `profiles`
- `id` (same ID as auth user where practical)
- `full_name`
- `email`
- `phone` nullable
- `requested_role`
- `created_at`, `updated_at`
- Do not treat `requested_role` as sufficient authorization.

### `school_memberships`
- `id`, `school_id`, `user_id`
- `role` enum: `admin`, `driver`, `parent`
- `status`: `pending`, `active`, `suspended`, `revoked`
- `created_at`, `created_by`
- Unique constraint suitable for user + school + role.

### `invitations`
- `id`, `school_id`, `email_or_phone` (minimize/secure)
- `intended_role`
- `student_ids` or a join table for intended parent links
- hashed token/code, `expires_at`, `used_at`, `revoked_at`, `created_by`
- Never store reusable raw invite secrets in logs.

### `buses`
- `id`, `school_id`, `label`/fleet number, optional registration identifier
- `status`: active/inactive
- `created_at`, `updated_at`, `archived_at`

### `drivers`
Prefer a profile + membership relationship; optional driver-specific table:
- `id`, `school_id`, `user_id`, optional staff code
- status and created timestamps.

### `students`
- `id`, `school_id`, display name or minimal identifier, optional class/grade
- `status`, timestamps
- Collect only the minimum data needed for the pilot.

### `parent_student_links`
- `id`, `school_id`, `parent_user_id`, `student_id`
- `status`, `created_by`, `created_at`
- Ensure parent belongs to the same school.

### `routes`
- `id`, `school_id`, `name`, `direction`/route type, status
- Optional encoded route geometry
- timestamps.

### `stops`
- `id`, `school_id`, `route_id`, `name`, description/address
- `latitude`, `longitude`, `sequence_number`
- `arrival_radius_m` (default 500 for proximity, if configured at stop level)
- timestamps.

### `student_route_assignments`
- `id`, `school_id`, `student_id`, `route_id`, `stop_id`
- effective dates/status
- Validate stop belongs to route and all records belong to same school.

### `trip_templates` (scheduled recurring trip)
- `id`, `school_id`, `route_id`, direction, scheduled departure time, weekdays, expected duration or arrival window, status.

### `trips` (individual occurrence)
- `id`, `school_id`, `trip_template_id` nullable, `route_id`, `bus_id`, `driver_user_id`
- `scheduled_start_at`, `actual_start_at`, `actual_end_at`
- `status`: `scheduled`, `active`, `delayed`, `cancelled`, `completed`, `aborted`
- `delay_reason` nullable, `created_at`, `updated_at`
- Use timezone-aware timestamps and store in UTC, render in school timezone.

### `trip_locations`
- `id`, `school_id`, `trip_id`, `driver_user_id`
- `latitude`, `longitude`, `accuracy_m`
- `client_recorded_at`, `server_received_at`
- optional `speed_mps`, `heading_degrees`
- indexes by trip and received time; apply retention policy.

### `trip_events`
- `id`, `school_id`, `trip_id`, optional `stop_id`
- `event_type`, `severity`, `status` (open/acknowledged/resolved)
- `payload` JSON for bounded metadata, not arbitrary sensitive data
- `occurred_at`, `created_at`, `acknowledged_by`, `acknowledged_at`, `resolved_by`, `resolved_at`, resolution note
- `idempotency_key` unique.

### `notifications`
- `id`, `school_id`, `recipient_user_id`, `trip_id` nullable, `event_id` nullable
- `type`, `title`, `body`, `created_at`, `read_at`
- `delivery_channel`, `delivery_status`, `attempt_count`, `last_error` sanitized
- dedupe/idempotency key.

### `push_subscriptions` (if web push is implemented)
- `id`, `user_id`, `endpoint`, encrypted/secured subscription keys, timestamps, revoked status.
- Do not expose subscription secrets to other users.

### `audit_logs`
- `id`, `school_id`, `actor_user_id`, `action`, `entity_type`, `entity_id`
- safe metadata, timestamp, request/correlation ID.
- Do not log passwords, invite tokens, secret keys, or excessive child location details.

### Relationships and constraints
- All school-owned objects must be scoped to a school.
- Route stop belongs to route; bus, driver, route and trip belong to same school.
- Parent-student link, student assignment, trip, stop and route must be consistent.
- Driver can upload coordinates only for their own currently active assigned trip.
- Parents can read only relevant trips/stops/locations associated with their linked child.
- Admin can read/manage only their school.
- Add database constraints where possible and enforce additional invariants server-side.

---

## 10. Backend/API behaviour

Use server-side service functions or API routes for business-critical actions. Names below are conceptual; adapt to framework conventions.

### Authentication/membership
- `GET /me` — profile, memberships, active school and role.
- `POST /invitations/accept` — validate token, expiry, intended role and school.
- Admin endpoints for creating/revoking invitations and memberships.

### Admin configuration
- CRUD endpoints for school settings, buses, routes, stops, students, parent links, driver assignments and trip templates.
- `POST /trips/{id}/start` should normally be driver-only and validate assignment/state.
- `POST /trips/{id}/end` validates active assignment/state.
- `POST /trips/{id}/cancel` admin-only with reason.
- `POST /trips/{id}/delay` admin-only with reason/optional ETA.
- `POST /trips/{id}/incidents` driver/admin with incident type and details.
- `POST /events/{id}/acknowledge` and `/resolve` admin-only.

### Location
- `POST /trips/{id}/locations` authenticated driver only.
- Validate active trip, assignment, school, coordinate ranges, accuracy and rate.
- Store server receive time.
- Process geofence/route/stale events idempotently.
- `GET /trips/{id}/latest-location` only for authorized roles.
- `GET /trips/{id}/locations` restricted history endpoint for authorized admin and relevant parent only; avoid exposing full historical traces unnecessarily.
- Realtime channel or subscription must enforce row/channel authorization. A random trip ID is not authorization.

### Notifications
- `GET /notifications`
- `POST /notifications/{id}/read`
- Internal event-to-recipient resolver; clients must not be able to nominate arbitrary recipients for system events.
- `POST /admin/notices` for admin-authored, scoped messages.

Use typed validation schemas, predictable error responses and server logs with request IDs. Return no secrets or raw stack traces to clients.

---

## 11. UI and screen inventory

### Public/auth screens
- Landing / product overview (can be minimal for pilot).
- Sign in.
- Sign up/request access with role choice.
- Accept invitation.
- Forgot/reset password.
- Pending approval / invalid or expired invite.
- Privacy and terms placeholders that must be completed before real school/child data is used.

### Parent screens
1. Home: selected child, trip status, bus status, ETA and last updated.
2. Live tracking: map, bus marker, assigned stop, status/freshness.
3. Trip details: schedule, route/stop and relevant notices.
4. Notifications: list, read/unread and event details.
5. Children switcher.
6. Profile, notification settings, school contact and sign out.

### Driver screens
1. Today's trips.
2. Trip details and route stops.
3. Pre-start permission/location check.
4. Active trip screen with simple controls and clear tracking state.
5. Report issue: delay, breakdown, emergency, other.
6. Trip completion confirmation.
7. Profile/sign out.

### Admin screens
1. Dashboard/fleet overview.
2. Live map and active trip detail.
3. Alerts/incidents queue.
4. Schools/settings (single school tenant settings).
5. Buses list/create/edit.
6. Drivers list/invite/assign/deactivate.
7. Routes list/editor and stop map picker.
8. Students list/create/edit/import if feasible.
9. Parent invitations and parent-student links.
10. Trip templates/schedule and individual trips.
11. Trip history.
12. Notifications/notices.
13. Audit log.

### UI design guidance
- Modern, clean, trustworthy and operational; avoid clutter and excessive animations.
- Prioritize mobile layout for parent and driver, desktop/tablet layout for admin.
- Use consistent status badges/text: Active, Delayed, Location stale, Location unavailable, Completed, Cancelled, Incident open.
- Don't rely on color alone to communicate severity.
- Use confirmation dialogs for trip end/cancel and high-impact admin actions.
- Include empty states that explain the next action (e.g., “No bus assigned to this route yet”).
- Build reusable components rather than duplicating role-specific logic.

---

## 12. Technical implementation guidance

Claude Code should first inspect the existing repository and use its current framework/conventions if sound. If this is a blank repository, propose and implement a modern, maintainable stack with TypeScript.

Suggested default, only if repository is blank:
- Next.js/React with TypeScript for the responsive web app and server routes.
- A managed PostgreSQL database and authentication service (Supabase is a reasonable candidate if already configured; otherwise choose a well-supported equivalent).
- Database row-level security where available, plus server-side authorization.
- A map library/provider chosen after checking terms, API limits and key handling.
- Server-side geofence/notification processing.
- Web Push with a service worker only if implemented end-to-end; in-app notifications are mandatory regardless.
- Zod or equivalent runtime validation.
- Unit and integration tests.
- Deployment target compatible with HTTPS and background/service-worker needs.

Do not assume any third-party service is configured. Provide `.env.example` with placeholder variable names and explain where each key comes from. Never invent API keys.

### Required engineering practices
- TypeScript strict mode (or equivalent type safety).
- Migrations checked into version control.
- Seed/demo data isolated from production.
- No production credentials in repository.
- Lint, typecheck, test and build scripts.
- Server-side access control and database policies.
- Error/loading/empty states.
- No fake buttons: every visible action must work or be clearly marked unavailable.
- No mock GPS data in production paths. Demo/simulation mode, if included, must be explicitly labelled and impossible to confuse with real tracking.
- Keep business logic in testable modules: permissions, trip transitions, geofence evaluation, ETA, notifications and recipient resolution.

### Suggested project structure
Adapt to the chosen framework, but maintain separation similar to:
```
src/
  app/ or routes/
  components/
  features/
    auth/
    admin/
    driver/
    parent/
    trips/
    routes/
    tracking/
    notifications/
    incidents/
  lib/
    auth/
    db/
    permissions/
    validation/
    maps/
    geofencing/
    notifications/
    time/
  server/
    services/
    repositories/
  types/
  tests/
database/
  migrations/
  seed/
docs/
  architecture.md
  gps-and-location.md
  security-and-privacy.md
```

---

## 13. Security, privacy and child safety

This product processes sensitive location data and information relating to children. Treat privacy/security as core requirements.

- Use HTTPS in production.
- Enforce role and school authorization server-side on every protected read/write.
- Use database RLS or equivalent defense-in-depth where supported.
- Restrict parent access to linked children and their assigned trip only.
- Restrict drivers to their own assigned trips.
- Restrict admin access to their school.
- Keep location collection off outside active trips.
- Minimize data collected and retained.
- Establish configurable retention/deletion for raw location points and incident records.
- Protect invitations against brute force, replay and leakage; hash tokens and set expiry.
- Rate-limit authentication, invitation acceptance, location writes and incident creation.
- Sanitize/validate all input; guard against IDOR, XSS, CSRF where applicable, injection and insecure direct object references.
- Avoid sensitive child names/locations in analytics, URLs, logs and push payloads.
- Log security-relevant admin actions.
- Provide account deactivation and parent-link revocation.
- Document how to report an incident and how to export/delete data.
- Do not claim legal compliance without review. Before a real pilot, review applicable Indian privacy, child-data, school transport and state-specific requirements with qualified counsel and the school.
- Do not treat this product as a replacement for emergency services or mandated vehicle safety equipment.

---

## 14. Error handling and edge cases

Implement and test at least:
- Driver denies location permission.
- Geolocation API unsupported.
- GPS reports unavailable or poor accuracy.
- Browser/tab suspended; updates stop.
- Phone battery dies.
- Mobile network drops and returns.
- Duplicate/out-of-order location points.
- Driver tries to start a trip assigned to someone else.
- Driver attempts to start an already active/completed trip.
- Parent invitation is expired, revoked, already used or intended for another role.
- Parent is unlinked from a student while signed in.
- Bus/route/driver is deactivated while future trips exist.
- Trip is cancelled while active.
- GPS jitters around geofence boundary.
- Bus passes a stop without stopping.
- Multiple parents/children share a stop.
- No valid ETA can be calculated.
- Map provider fails or API quota is exceeded.
- Notification provider is unavailable.
- Admin accidentally creates inconsistent route/stop assignments.
- User from School A attempts to access School B's object ID.
- Daylight-saving/timezone edge cases (even though default timezone is India, store UTC and render correctly).
- Network retries cause duplicate trip events or notifications.

---

## 15. Acceptance criteria

The MVP is not complete until the following are demonstrably working:

### Access and onboarding
- A user cannot gain admin access by choosing “School Admin” on sign-up.
- Admin can create/invite a driver and parent.
- Parent invitation links to the intended student(s).
- Uninvited user cannot view school data.
- Cross-school access tests pass.

### Configuration
- Admin can create a bus, route, at least two ordered stops, a scheduled trip, a driver assignment and student/parent links.
- Invalid assignments (e.g., stop from another route or school) are rejected.

### Trip and GPS
- Assigned driver can start a scheduled trip.
- Starting a trip changes status and creates an audit/event record.
- Driver's phone obtains permission and sends coordinates over HTTPS.
- Admin sees fresh location and a last-updated timestamp.
- Authorized parent sees only the relevant bus/stop.
- Denied permission and stale location are clearly represented.
- Ending/cancelling a trip stops accepting new location points for that trip.
- Driver cannot upload points to another driver's trip.
- Production UI never mislabels stale coordinates as live.

### Geofence and alerts
- A test route with known coordinates triggers one proximity event when entering the configured 500 m radius.
- Remaining inside the radius does not create duplicates.
- GPS jitter does not repeatedly trigger alerts because hysteresis/idempotency is applied.
- Only parents linked to the affected stop/trip receive the alert.
- Stale/invalid GPS points do not trigger proximity alerts.
- Notification events are persisted and visible in-app.
- GPS loss, delay, cancellation and incident states are reflected in the admin UI.

### Usability and delivery
- All key screens work at common mobile widths and desktop width.
- No primary action is a dead button.
- Empty, loading, error, permission and offline states exist.
- Lint, typecheck, automated tests and production build pass.
- README documents setup, migrations, environment variables, seed/demo flow, GPS testing, known browser limitations and deployment.

---

## 16. Testing plan

### Unit tests
- Role and school permission helpers.
- Trip state transition rules.
- Coordinate validation.
- Distance calculation.
- Geofence entry/exit/hysteresis.
- Stale location calculation.
- ETA fallback behaviour.
- Notification recipient resolution and idempotency.
- Timezone/schedule conversion.

### Integration tests
- Invite acceptance and role assignment.
- Admin creates route/stop/trip.
- Driver starts trip and uploads location.
- Parent reads authorized live location.
- Parent from another school is denied.
- Driver cannot upload to another trip.
- Geofence creates one event and the correct notifications.
- Trip cancellation and ending revoke further tracking.
- Notification failure does not corrupt trip state.

### Manual device tests
Use at least one real Android phone with the intended mobile browser:
- HTTPS permission prompt.
- App foreground and background behaviour.
- Screen lock behaviour (document results honestly).
- GPS indoors/outdoors and weak-signal behaviour.
- Mobile data off/on.
- Battery optimization.
- Geofence threshold around 500 m.
- Parent notification delivery while app is closed, if web push is implemented.
- iOS Safari smoke test if iOS is intended for the pilot.

### Demo mode
A simulator can help development, but it must be visually labelled “DEMO/SIMULATED LOCATION,” must not share simulated points with production trips, and must never be represented as a real bus location.

---

## 17. Phased build plan for Claude Code

Do not attempt to build everything in one pass. Work in phases, keeping the application runnable at each stage.

### Phase 0 — Repository inspection and plan
- Inspect files, framework, dependencies, scripts and current state.
- Summarize existing architecture and identify missing setup.
- Create a task checklist and `docs/architecture.md`.
- If the repo is blank, propose a stack and explain why briefly, then proceed.
- Do not overwrite existing user work without understanding it.

### Phase 1 — Foundation
- Configure app, types, lint, tests and environment template.
- Implement authentication and profile/membership model.
- Implement protected routes and server-side role/school checks.
- Create migrations and safe seed/demo flow.
- Verify admin cannot be self-assigned.

### Phase 2 — School setup and invitations
- School settings, buses, drivers, routes, stops, students and parent links.
- Invitations and revocation.
- Admin CRUD pages and validation.
- Cross-school authorization tests.

### Phase 3 — Trips and driver experience
- Trip templates/instances, assignments and valid state transitions.
- Driver's assigned-trip list and trip detail.
- Start/end trip and incident report.
- Audit/event records.

### Phase 4 — GPS pipeline
- Browser Geolocation API integration.
- Secure location ingestion endpoint.
- Latest location and history storage.
- Live update mechanism (realtime or polling).
- Map view and freshness indicators.
- GPS failure states.
- Device tests and documentation.

### Phase 5 — Geofence and notifications
- Server-side 500 m stop proximity detection with hysteresis.
- Idempotent trip/stop events.
- In-app notifications and read state.
- Trip start/end, delay, cancellation, GPS stale, route deviation and breakdown events.
- Web Push only if it can be implemented reliably with real credentials/configuration.
- Notification recipient and cross-school tests.

### Phase 6 — Parent and admin live operations
- Parent home/map/ETA/notifications.
- Admin live fleet dashboard and alerts.
- Acknowledge/resolve incident.
- Parent-facing delay/cancellation notices.
- Responsive polish and accessibility.

### Phase 7 — Hardening and pilot readiness
- Full test suite, lint, typecheck and production build.
- Security and privacy review.
- Real-device GPS tests.
- `.env.example` and README completion.
- Seed a fictional demo school with fictional students and clearly simulated data.
- Write a known-limitations list and pilot checklist.

After each phase:
1. Run relevant tests.
2. Run lint/typecheck/build as available.
3. Fix regressions before moving on.
4. Update the checklist and docs.
5. Summarize files changed, tests run, remaining blockers and any credentials required.

---

## 18. Definition of done

The MVP is ready for a controlled school demonstration when:
- One admin can configure a fictional or pilot school.
- Admin can onboard a driver and parent without open school discovery.
- Admin can create a route, stop, bus and trip.
- Driver can start a trip from a phone browser and share actual GPS coordinates while the app is active.
- Admin sees the bus update on a map.
- Parent sees only the assigned bus and stop.
- The 500 m proximity event is calculated on the server and delivered once to the correct parent.
- Trip start/end and operational status notifications work in-app.
- GPS loss is shown honestly and no stale location is portrayed as live.
- Cross-school and unauthorized-access tests pass.
- Known web background-tracking limitations are clearly documented.
- All setup, environment, migration, testing and deployment steps are documented.

A demo-ready MVP is not yet proof of production-grade safety or uninterrupted tracking. Pilot with explicit school consent, monitor closely, and validate the browser-tracking limitations before relying on it operationally.

---

## 19. Instructions to Claude Code

You are the implementation agent for this repository. Treat this PRD as the product contract.

1. Read the entire PRD and inspect the repository before coding.
2. Do not ask the founder to make every minor technical decision. Make conventional, maintainable choices and document them.
3. Ask a question only if a genuinely blocking decision cannot be resolved safely from the PRD or repository.
4. Do not assume secrets, API keys, domains, email providers, or third-party services exist. Add placeholders and setup instructions.
5. Do not fake GPS, maps, push notifications, authentication, or persistence. If an integration is not configured, show an honest setup/error state.
6. Implement real role-based access and school-level data isolation from the beginning.
7. Use migrations and test data; never use real child data in seeds.
8. Keep each phase small and runnable. Do not dump a giant untested codebase in one pass.
9. After each phase, run tests, lint, typecheck and build where supported; fix failures before proceeding.
10. Keep the README, `.env.example`, architecture docs, GPS docs and checklist up to date.
11. Be explicit about browser geolocation/background limitations and test on real mobile devices.
12. When finished, report what works, what needs credentials, tests run, known limitations and the exact steps to run the app locally.

**Start with Phase 0, then proceed through the phases in order.**
