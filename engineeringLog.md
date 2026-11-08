# Fleet OverWatch — Engineering Log

_Development history, release log and time record, v1.0 to v4.5.13_

| | |
|---|---|
| **Project** | Fleet OverWatch (fleetoverwatch.com) |
| **Author** | Franko (jBravo) |
| **Period** | September 30 – November 6, 2026 (6 weeks) |
| **Current release** | Server 4.5.13 · Mobile app 1.3.0 |
| **Releases logged** | 67 server releases, 4 mobile app releases |
| **Time recorded** | 192 hours (estimated; see Time Record) |

## Contents

1. [Summary](#summary)
2. [Technology stack](#technology-stack)
3. [Time record](#time-record)
4. [Release log](#release-log)
5. [Incident log](#incident-log)
6. [Engineering practices](#engineering-practices)

## Summary

Fleet OverWatch started as a desktop dashboard that merged two vendor feeds, Samsara telematics and Fleetio maintenance, into one live view for dispatch and the shop. Over six weeks it grew into a self-hosted, multi-tenant fleet management platform: maintenance, dispatch, inventory, DOT compliance, driver scoring and a live map, served from a LAMP server with Android and iOS clients.

This log records each iteration: what changed, why, the defects found and how they were fixed. It also records the infrastructure and release work that kept a live system updateable without downtime or data loss. All work was done outside a regular 12-hour work day, on weekday evenings and longer weekend sessions.

## Technology stack

| Area | Choice |
|---|---|
| Runtime | PHP 8.3 (php-fpm) on Ubuntu, Apache 2.4, OPcache |
| Storage | SQLite 3 in WAL mode, one database per company account; MySQL supported |
| Front end | Vanilla JavaScript (no framework), Leaflet for maps, CSS custom properties for theming |
| Background work | Long-running PHP worker (sync, health checks, alerts, photo capture) under cron supervision |
| Integrations | Samsara REST API, Fleetio REST API and CSV exports, TomTom, NHTSA recalls, Comdata, SMTP |
| Public API | REST API at /api/v1 with token auth, 50+ webhook events, signed inbound hooks |
| Mobile | Android (Kotlin, WebView) and iOS (Swift, WKWebView) thin clients, built on GitHub Actions |
| Quality | Acceptance suite (127 automated checks), Playwright device emulation, secret scan, install tests |

## Time record

Hours are estimates, recorded per week and per release, against the actual release dates. Work was done outside a regular 12-hour work day: about 4 hours on weekday evenings and 8 hours on Saturdays and Sundays, so a full week is 36 hours and no week exceeds 50.

| Week | Dates | Focus | Hours |
|---|---|---|---:|
| 1 | Sep 30 – Oct 4 (Wed–Sun) | Python prototype 1.0 and 1.1 | 28 |
| 2 | Oct 5 – Oct 11 | LAMP port (2.0), production server, 3.0 sections, Windows build | 36 |
| 3 | Oct 12 – Oct 18 | Traffic analytics, 4.0 platform rebuild, 4.0.x fixes, 4.1 users and DOT | 36 |
| 4 | Oct 19 – Oct 25 | 4.2.1 – 4.2.11 multi-tenant hosting, Fleetio import; 4.3 build | 36 |
| 5 | Oct 26 – Nov 1 | 4.3.0 – 4.3.24 beta requirements, public API, inventory; 4.4 build | 36 |
| 6 | Nov 2 – Nov 6 (Mon–Fri) | 4.4.0 – 4.5.13, mobile apps 1.0 – 1.3 | 20 |
| | | **Total** | **192** |

## Release log

Entries use the [Keep a Changelog](https://keepachangelog.com) verbs: **Added**, **Changed**, **Fixed**. Hours are per release or release group.

### Phase 1 — Desktop prototype (v1.x)

Goal: prove that Samsara and Fleetio data could be joined into one useful operational view before investing in a server product.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Sep 30 – Oct 2 | `1.0` | **Added:** Python desktop dashboard combining Samsara GPS and Fleetio issues, work orders, inventory and DVIRs.<br>**Added:** simulated-data mode so the UI could be built and demoed before production API credentials arrived; a settings switch moves it to live keys.<br>**Added:** live map with ETA estimates, destination highlighting of nearby trucks, truck detail panel with engine fault codes and camera links, filters, weather and traffic overlays. | 12 |
| Oct 3 – Oct 4 | `1.1` | **Changed:** layout rebuilt to the approved mockup (left nav, map, right-side cards, bottom vehicle panels); 3D globe replaced with a flat map; TomTom integrated.<br>**Added:** dispatch assistant that filters by vehicle class and excludes shop equipment; Fleetio "Driver Damage" work orders under Maintenance.<br>**Added:** trailer tracking with the same alerting and colour coding as trucks; idle rules (parked within 10 mi of HQ, idle timer after 20 min beyond 25 mi, adjustable); purple highlight for idle trucks outside the yard geofence.<br>**Changed:** demo data removed; live data only. Renamed Fleet Command to Fleet OverWatch. | 16 |
| | | **Phase total** | **28** |

### Phase 2 — Web edition and production server (v2.0 – v3.0)

Goal: move from a single desktop to a server everyone in the company can reach, and stand up the production host.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 5 – Oct 6 | `2.0` | **Changed:** ported 1.1 to a LAMP web application (HTML, CSS, JavaScript, PHP) so it runs in any browser.<br>**Added:** server-side API proxy so vendor keys never reach the browser; configuration file outside the web root. | 8 |
| Oct 7 | `Infra` | Provisioned the production host: Ubuntu VM on DreamCompute, Apache, PHP, firewall, SSH key auth, TLS for fleetoverwatch.com.<br>Evaluated DigitalOcean as an alternative; stayed on DreamCompute on cost, with headroom to scale. | 4 |
| Oct 8 – Oct 10 | `3.0` | **Changed:** split the app into independently maintainable sections: Dashboard hub (API keys, connection status), Dispatch, Shop, Safety, Finance, Drivers and Overwatch (managers and owners).<br>**Added:** theme editor where every UI element has its own colour (no grouped tokens).<br>**Added:** demo and live editions on the same server. | 16 |
| Oct 11 | `3.0 (Windows)` | **Added:** Windows desktop build of the 3.0 web edition in Python, with an installer project that lets a customer drop in their own logo.png and build an executable. | 8 |
| | | **Phase total** | **36** |

### Phase 3 — Platform rebuild (v4.0 – v4.1)

Goal: stop depending on Fleetio for core records. 4.0 is a standalone maintenance, dispatch and asset-tracking system that can import Fleetio exports or sync via its API.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 12 | `Infra` | **Added:** site traffic tracker with an install script: who visits, which edition (demo or live), what they do, and errors.<br>**Added:** bot and probe detection with automatic blocking, plus an analytics dashboard in the admin area. | 4 |
| Oct 13 – Oct 15 | `4.0.0` | **Changed:** own data model and database tables for vehicles, contacts, issues, work orders, parts, inspections and fuel; Fleetio becomes an optional import and sync source.<br>**Added:** one-file installer and updater (update-x.y.z.sh) with integrity check, automatic backup to /root/fot-backups and --rollback.<br>**Added:** admin-only Admin section with server, application and network monitoring, logs and graphs. | 12 |
| Oct 16 | `4.0.2 – 4.0.5` | **Added:** DVIR rework (full report view, defect sign-off, one work order for many defects), maintenance timers by days/miles/engine hours, 21 standard reports, assignments, NHTSA recall checks.<br>**Added:** inspection photos from Samsara's newer DVIR stream API, copied to the server because vendor photo links expire after 24 hours.<br>**Fixed:** Samsara HTTP 400 on long DVIR ranges (now requested in 30/7/1-day windows); admin graphs stopped recording (missing worker function); server status failing on hardened PHP builds. | 4 |
| Oct 17 – Oct 18 | `4.1.0` | **Added:** users and roles (custom roles, per-record permissions, password policy, lockout, sign-in history, active sessions, My account).<br>**Added:** DOT compliance module: driver qualification files (49 CFR 391.51), drug and alcohol testing with reproducible random draws, accident register (390.15), roadside inspections, IFTA worksheets, DOT vehicle file (396.3), document library. | 8 |
| | | **Phase total** | **28** |

### Phase 4 — Multi-tenant hosting and data migration (v4.2.x)

Goal: let other companies run on the same server, and make migration from Fleetio lossless.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 18 – Oct 20 | `4.2.1` | **Added:** public landing page, user manual, command dashboard, per-visitor sandboxed demo with a guided tour.<br>**Added:** company accounts (self sign-up, manual approval, company-code sign-in, per-company branding and subdomains), each with an isolated database and worker.<br>**Added:** host scrub (worker.php scrub-host) so the platform hosts only customer accounts. | 16 |
| Oct 21 | `4.2.2 – 4.2.3` | **Added:** Fleetio work-order export import, part photos, per-role field-level data hiding (costs, pay, contacts), bulk tire stock.<br>**Added:** Basic / Advanced view switch, guided tour updates.<br>**Fixed:** company live data stopped updating; a watchdog now restarts a stalled per-company worker every minute. | 4 |
| Oct 22 | `4.2.4 – 4.2.11` | **Added:** driver card on the live map, custom driver avatars, phone alert stack, themed landing page; copy-from-platform transfer worker (4.2.7).<br>**Fixed:** "server helper not answering" after an in-app update (4.2.6); Fleetio CSV headers with time-zone suffixes were skipped (4.2.8); "\|"-separated task names split on commas; archived records imported as active (4.2.9); voided fuel entries and meter recounts on first sync (4.2.10).<br>**Added:** stable FleetOverWatch IDs on every record (FO-WO-00012) alongside the Fleetio ID, searchable and round-trippable through exports (4.2.11). | 4 |
| | | **Phase total** | **24** |

### Phase 5 — Beta requirements, public API and inventory (v4.3.x)

Goal: close out the 4.3 beta specification and the fleet management system reference, and match Fleetio's work-order and inventory behaviour exactly.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 23 – Oct 26 | `4.3.0` | **Added:** beta specification sections 29–36: consolidated alert card, calendar and time pickers everywhere, suggestions and bug reports, company forum by department. | 14 |
| Oct 25 – Oct 26 | `4.3.1` | **Changed:** parts leave stock when saved on a work order; every edit, cancel and reopen adjusts the count; negative stock allowed and flagged, as in Fleetio. | 3 |
| Oct 25 – Oct 26 | `4.3.2` | **Added:** REST API at /api/v1 (token plus account token), 50+ webhook events with retry outbox, signed inbound hooks processed asynchronously and idempotently, integration health and field ownership. | 7 |
| Oct 27 | `4.3.3 – 4.3.5` | **Added:** parts from inventory inside the work-order form; issues and inspection defects attached to work orders (completion resolves them); parts list with Fleetio's exact columns and a lossless part export import. | 1.5 |
| Oct 27 | `4.3.6 – 4.3.10` | **Added:** employee evaluations per department with admin-built forms, schedules and countdowns (results visible to managers only); photos and PDFs on work orders; site-wide forum with moderation.<br>**Fixed:** company bar overlapping page content; pages cut off at the bottom; company accounts no longer labelled "demo". | 1.5 |
| Oct 27 | `4.3.11 – 4.3.15` | **Added:** vehicle status drop-down with reasons, grouped colour-coded side menu, theme pack, guarded "delete all inventory".<br>**Fixed:** demo redirect loop on hosting servers (4.3.14). Changed: demo runs as a normal account with sample data (4.3.15). | 1 |
| Oct 28 | `4.3.16 – 4.3.18` | **Added:** driver roster (one line per truck), website contact form, admin record eraser, bulk tire orders with photos and invoices, host administrator tools. | 2 |
| Oct 28 | `4.3.19 – 4.3.22` | **Added:** Comdata fuel cards, vehicle photos as list icons, one-step company sign-in reset for host admins, more themes, vehicle fuel type and grade, seasonal theme. | 2 |
| Oct 29 | `4.3.23` | **Added:** evaluations for any typed name; PDF and Word export of evaluations; printable blank forms. | 2 |
| Oct 29 | `4.3.24` | **Added:** SMTP email servers (site-wide and per company) with provider presets, masked passwords and a test; "Forgot password" with single-use, one-hour reset links. | 2 |
| | | **Phase total** | **36** |

### Phase 6 — Operations, scoring, live map and mobile (v4.4 – v4.5)

Goal: harden the platform for daily use, add driver performance tooling, and make every screen work on phones and tablets.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 30 – Nov 2 | `4.4.0` | **Added:** server administration console for host admins: health checks with self-repair, issue tracker with history, admin alerts by category and severity, IP access control with automatic blocking.<br>**Fixed:** corrections list from the 4.3 review. | 22 |
| Nov 2 | `4.4.1` | **Added:** Automation and AI feature switches, inventory locations and counts, part lookup with pictures, server control panel. | 2 |
| Nov 3 | `4.4.2 – 4.4.5` | **Added:** Fleetio service reminders and part export, full Fleetio transfer, barcode scan-to-fill for parts and work orders; vehicle status follows Fleetio; reconciliation report proving every Fleetio record is accounted for. | 1.5 |
| Nov 3 | `4.5.0` | **Added:** shipment board, driver scores and report cards, 50 themes, interface pass. | 1 |
| Nov 3 | `4.5.1 – 4.5.2` | **Added:** grading settings and score resets, truck scores, vehicle classes and shipments on the live map, driver scores in every department, achievement board, 100 themes. | 1.5 |
| Nov 4 | `4.5.3` | **Fixed:** CPU pinned at 70–100%. Profiled the admin screens: directory-size scans ran on every poll. Added 15-minute cached disk sizing with a time budget, shared 2-second cache for live readings, SQLite tuning (cache_size, mmap, temp_store), and an OPcache profile written by the installer.<br>**Fixed:** "database is locked" in the demo account register (schema check once per version, WAL, busy timeout); worker_age() undefined; SplFileInfo::getSize stat failures on files deleted mid-scan.<br>**Added:** "Clear list" for the issues panel; new default home location for demo and blank accounts. | 2 |
| Nov 4 | `4.5.4 – 4.5.5` | **Changed:** 4.5.4 was rolled back at the product owner's request; 4.5.5 is 4.5.3 plus the driver grade curve only (shop-only line items that adjust a driver's score, shown on report cards). | 1 |
| Nov 4 | `4.5.6 – 4.5.7` | **Added:** grade curve for shop and admins; SVG vehicle icons by class (tractor, box truck, pickup, trailer); Trailers panel with geofence locations, colour by trailer style and a user-editable Top 5; admin icon size control. | 1 |
| Nov 5 | `4.5.8` | **Changed:** full mobile pass on iOS Safari and Android Chrome, audited with Playwright device emulation (iPhone SE, iPhone 13, Pixel 7) across every page and role: 16 px inputs to stop iOS zoom, scrollable tab strips, sticky modal footers, phone fleet sheet, no horizontal overflow at 320 px.<br>**Added:** per-asset icon and colour override with 20 shapes (flatbed, dry van, RGN, port chassis, tanker…). | 1.5 |
| Nov 5 | `4.5.9` | **Added:** icon style (pictures, classic pins, class badges, dots, heading arrows) and size per user in Themes, with an admin company default.<br>**Changed:** on phones the vehicle details open as a draggable bottom sheet (peek and full); the map recentres the selected truck in the visible area and lifts its controls above the sheet. | 1 |
| Nov 5 | `4.5.10` | **Fixed:** Gmail SMTP 535 with valid credentials. Root causes: app passwords pasted with Google's display spaces, and browser autofill writing the site login into the SMTP fields. Normalised the password, locked the fields until focused, added a Gmail-specific diagnosis. | 0.5 |
| Nov 6 | `4.5.11` | **Added:** Share location and Directions (Google Maps, Apple Maps, Waze, Android geo:) for trucks and trailers, using the native share sheet when available.<br>**Changed:** alerts moved out of the content area (side-menu bell, bottom-left cards; on phones a bottom line suppressed while details are open); solid details panel; larger touch targets; Android download link on the landing page. | 1.5 |
| Nov 6 | `4.5.12` | **Changed:** on phones the Basic / Advanced switch moved from the top bar to the top of the menu. | 0.5 |
| Nov 6 | `4.5.13` | **Fixed:** admin alerts not arriving by email. Delivery failures were swallowed. The alert path now records the outcome (no site mail server, no recipients, or the SMTP server's reply) and the admin page and test button report it. | 1 |
| | | **Phase total** | **38** |

### Mobile clients (Android and iOS)

Thin native shells around the hosted web app, so every server release reaches phones without an app update. Native code is limited to what a browser cannot do.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Nov 5 | `App 1.0 – 1.1` | **Added:** Android (Kotlin, minSdk 24) and iOS (Swift, iOS 15, XcodeGen) projects: camera and photo upload, geolocation, downloads, printing, external links to native apps, offline screen, MDM-managed server address, app shortcuts per department.<br>**Added:** GitHub Actions pipeline (Android APK/AAB, iOS archive and IPA with signing from repository secrets).<br>**Changed:** scope widened from drivers to all employees (admin and every department). | 1 |
| Nov 6 | `App 1.2 – 1.3` | **Fixed:** CI failures (starter workflow looking for gradlew at the repo root; setup-android installing the removed "tools" package).<br>**Changed:** server address built in; the app opens straight to sign-in.<br>**Added:** native share sheet, haptics for alerts, keep-screen-on, biometric app lock (BiometricPrompt / LocalAuthentication). | 1 |
| | | **Phase total** | **2** |

## Incident log

Production and build defects worth recording, with root cause and fix.

| ID | Date | Version | Symptom | Root cause | Resolution |
|---|---|---|---|---|---|
| INC-01 | Oct 16 | 4.0.2 | Samsara DVIR requests returned HTTP 400 | API rejects long date ranges | Adaptive windowing (30/7/1 days) and the newer stream endpoints |
| INC-02 | Oct 16 | 4.0.5 | DVIR photos missing after a day | Vendor photo URLs expire after 24 h | Worker copies photos to local storage within seconds |
| INC-03 | Oct 21 | 4.2.3 | Company live data stopped updating | Per-company worker stalled without exiting | Watchdog restarts stalled workers every minute |
| INC-04 | Oct 22 | 4.2.6 | "Server helper not answering" after in-app update | Worker started by the installer under the wrong user | Installer restarts the worker as the web user |
| INC-05 | Oct 22 | 4.2.8 – 4.2.10 | Fleetio CSV imports dropped data | Header suffixes, comma splitting, archived and voided rows | Header normalisation and row-state mapping; lossless re-import |
| INC-06 | Oct 27 | 4.3.14 | Demo "redirected you too many times" | Section routing on hosting servers | Routing fixed; demo runs as a normal sandboxed account |
| INC-07 | Nov 4 | 4.5.3 | CPU pinned at 70–100% | Directory scans on every admin poll; no OPcache tuning | Cached sizing with time budget, shared reading cache, SQLite and OPcache tuning |
| INC-08 | Nov 4 | 4.5.3 | "database is locked" | Schema check on every request against a busy register | Version-gated schema check, WAL, busy timeout |
| INC-09 | Nov 4 | 4.5.3 | Debug log line found in admin code | Leftover diagnostic write to /tmp | Removed; build now rejects any package containing debug markers |
| INC-10 | Nov 5 | 4.5.10 | Gmail SMTP 535 with correct credentials | App-password spaces; browser autofill | Password normalisation, autofill lock, provider-specific diagnosis |
| INC-11 | Nov 6 | App 1.2 | Android CI build failed | Wrong workflow; deprecated SDK package | Project-aware workflow using the preinstalled SDK |
| INC-12 | Nov 6 | 4.5.13 | Admin alert emails not arriving | Delivery errors swallowed silently | Outcome recorded and surfaced on the admin page and test |

## Engineering practices

- **Release pipeline.** Each release is a single self-extracting update-x.y.z.sh (plus a full installer) carrying the whole program, a checksum, an automatic backup and --rollback. The build refuses to package configuration, customer data, analytics logs or debug markers.
- **Testing.** An acceptance suite (127 automated checks at 4.5.13, one block per requirement) runs on every build against a throwaway server with mock SMTP and vendor endpoints. UI work is checked with Playwright on desktop and emulated phones.
- **Install verification.** Every release is installed over the previous release and onto a clean directory before it ships; both must pass.
- **Security.** Server-side authorization on every route; bcrypt password hashes; secrets never sent to the browser or logged; masked credential fields; SSRF guard on outbound fetches; card numbers stored as last four only; one-time display of temporary passwords; a secret scan on every package.
- **Documentation.** Each release ships upgrade notes, in-app release notes, user manual updates, an API route list, a data dictionary and a requirements traceability matrix.
- **Change control.** Requirements are tracked row by row in the traceability matrix. A change the product owner rejects is rolled back as a new release (4.5.4 to 4.5.5), never patched over.

---

Prepared by Franko (jBravo) · November 7, 2026
