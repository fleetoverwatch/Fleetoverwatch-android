# Fleet OverWatch — Engineering Log

_How I built it: development history, release log and time record, v1.0 to v4.5.13_

| | |
|---|---|
| **Project** | Fleet OverWatch (fleetoverwatch.com) |
| **Author** | jBravo |
| **Period** | September 30 – November 6, 2026 (6 weeks) |
| **Current release** | Server 4.5.13 · Mobile app 1.3.0 |
| **Releases logged** | 67 server releases, 4 mobile app releases |
| **Time recorded** | 192 hours (estimated; see Time record) |

## Contents

1. [Summary](#summary)
2. [Technology stack](#technology-stack)
3. [Time record](#time-record)
4. [Release log](#release-log)
5. [Incident log](#incident-log)
6. [Engineering practices](#engineering-practices)

## Summary

I started Fleet OverWatch to solve a problem I saw every day at work: our dispatch and shop teams were bouncing between Samsara for telematics and Fleetio for maintenance, and neither tool showed the whole picture. My first version was a desktop dashboard that merged the two feeds into one live view. Six weeks later it is a self-hosted, multi-tenant fleet management platform. It covers maintenance, dispatch, inventory, DOT compliance, driver scoring and a live map, runs on a LAMP server I manage, and has Android and iOS clients.

I am sharing this log so other developers can see how the project evolved: what I changed in each iteration and why, the bugs I hit and how I fixed them, and the release and infrastructure work it took to keep a live system updateable without downtime or lost data. I did all of it outside my regular 12-hour work days, on weekday evenings and longer weekend sessions.

## Technology stack

I kept the stack deliberately boring so that a single developer could own every layer and anyone after me could read and change it.

| Area | Choice |
|---|---|
| Runtime | PHP 8.3 (php-fpm) on Ubuntu, Apache 2.4, OPcache |
| Storage | SQLite 3 in WAL mode, one database per company account; MySQL supported |
| Front end | Vanilla JavaScript (no framework), Leaflet for maps, CSS custom properties for theming |
| Background work | A long-running PHP worker (syncs, health checks, alerts, photo capture) supervised by cron |
| Integrations | Samsara REST API, Fleetio REST API and CSV exports, TomTom, NHTSA recalls, Comdata, SMTP |
| Public API | REST API at /api/v1 with token auth, 50+ webhook events, signed inbound hooks |
| Mobile | Android (Kotlin, WebView) and iOS (Swift, WKWebView) thin clients, built on GitHub Actions |
| Quality | Acceptance suite (127 automated checks), Playwright device emulation, secret scan, install tests |

## Time record

These hours are my estimates, recorded per week and per release against the actual release dates. I fit the work around a regular 12-hour work day: about 4 hours on weekday evenings and 8 hours on Saturdays and Sundays. That makes a full week 36 hours, and no week went past 50.

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

Each entry is tagged with the [Keep a Changelog](https://keepachangelog.com) verbs Added, Changed and Fixed. Hours are per release or release group and add up to the time record.

### Phase 1 — Desktop prototype (v1.x)

My goal here was to prove that joining Samsara and Fleetio data was worth it before I committed to building a server product.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Sep 30 – Oct 2 | `1.0` | **Added:** I built a Python desktop dashboard that combines Samsara GPS with Fleetio issues, work orders, inventory and DVIRs.<br>**Added:** I wrote a simulated-data mode so I could build and demo the UI before our production API credentials arrived; a settings switch moves it onto live keys.<br>**Added:** I put in a live map with ETA estimates, destination highlighting of nearby trucks, a truck detail panel with engine fault codes and camera links, filters, and weather and traffic overlays. | 12 |
| Oct 3 – Oct 4 | `1.1` | **Changed:** I rebuilt the layout to match the approved mockup (left nav, map, right-side cards, bottom vehicle panels), replaced the 3D globe with a flat map and wired in TomTom.<br>**Added:** I wrote a dispatch assistant that filters by vehicle class and leaves out shop equipment, and surfaced Fleetio "Driver Damage" work orders under Maintenance.<br>**Added:** I brought trailers onto the map with the same alerting and colour coding as trucks, and encoded our idle rules (parked within 10 mi of HQ, idle timer after 20 minutes beyond 25 mi, both adjustable), with a purple highlight for trucks idling outside the yard geofence.<br>**Changed:** I removed the demo data so it runs on live data only, and renamed the project from Fleet Command to Fleet OverWatch. | 16 |
| | | **Phase total** | **28** |

### Phase 2 — Web edition and production server (v2.0 – v3.0)

Next I needed everyone in the company to reach it from a browser, which meant standing up a production host.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 5 – Oct 6 | `2.0` | **Changed:** I ported 1.1 to a LAMP web application (HTML, CSS, JavaScript, PHP) so it runs in any browser.<br>**Added:** I put a server-side proxy in front of the vendor APIs so keys never reach the browser, and moved configuration outside the web root. | 8 |
| Oct 7 | `Infra` | I provisioned the production host: an Ubuntu VM on DreamCompute with Apache and PHP, a firewall, SSH key auth and TLS for fleetoverwatch.com.<br>I priced DigitalOcean as an alternative and stayed on DreamCompute for cost, with room to scale later. | 4 |
| Oct 8 – Oct 10 | `3.0` | **Changed:** I split the app into sections I can maintain independently: a Dashboard hub (API keys, connection status), Dispatch, Shop, Safety, Finance, Drivers and Overwatch for managers and owners.<br>**Added:** I built a theme editor where every UI element has its own colour, with no grouped tokens.<br>**Added:** I set up demo and live editions side by side on the same server. | 16 |
| Oct 11 | `3.0 (Windows)` | **Added:** I packaged a Windows desktop build of the 3.0 web edition in Python, with an installer project that lets a customer drop in their own logo.png and build an executable. | 8 |
| | | **Phase total** | **36** |

### Phase 3 — Platform rebuild (v4.0 – v4.1)

This is where I stopped depending on Fleetio for core records. I rebuilt 4.0 as a standalone maintenance, dispatch and asset-tracking system that can import Fleetio exports or sync through its API.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 12 | `Infra` | **Added:** I wrote a site traffic tracker with its own install script. It logs who visits, which edition (demo or live) they use, what they do, and any errors.<br>**Added:** I added bot and probe detection with automatic blocking, and an analytics dashboard in the admin area. | 4 |
| Oct 13 – Oct 15 | `4.0.0` | **Changed:** I designed my own data model and tables for vehicles, contacts, issues, work orders, parts, inspections and fuel; Fleetio became an optional import and sync source.<br>**Added:** I wrote a one-file installer and updater (update-x.y.z.sh) with an integrity check, an automatic backup to /root/fot-backups and a --rollback switch.<br>**Added:** I built an admin-only Admin section with server, application and network monitoring, logs and graphs. | 12 |
| Oct 16 | `4.0.2 – 4.0.5` | **Added:** I reworked DVIRs (full report view, defect sign-off, one work order for many defects) and added maintenance timers by days, miles and engine hours, 21 standard reports, assignments and NHTSA recall checks.<br>**Added:** I switched inspection photos to Samsara's newer DVIR stream API and had the worker copy them to the server, because the vendor's photo links expire after 24 hours.<br>**Fixed:** Samsara returned HTTP 400 on long DVIR ranges, so I now request them in 30, 7 and 1-day windows. I also fixed admin graphs that had stopped recording (a missing worker function) and server status failing on hardened PHP builds. | 4 |
| Oct 17 – Oct 18 | `4.1.0` | **Added:** I built users and roles: custom roles, per-record permissions, password policy, lockout, sign-in history, active sessions and a My account page.<br>**Added:** I wrote the DOT compliance module: driver qualification files (49 CFR 391.51), drug and alcohol testing with reproducible random draws, the accident register (390.15), roadside inspections, IFTA worksheets, the DOT vehicle file (396.3) and a document library. | 8 |
| | | **Phase total** | **28** |

### Phase 4 — Multi-tenant hosting and data migration (v4.2.x)

Here I opened the platform to other companies on the same server and made migration from Fleetio lossless.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 18 – Oct 20 | `4.2.1` | **Added:** I built the public landing page, the user manual, a command dashboard and a per-visitor sandboxed demo with a guided tour.<br>**Added:** I implemented company accounts (self sign-up, manual approval, company-code sign-in, per-company branding and subdomains), each with an isolated database and worker.<br>**Added:** I wrote a host scrub (worker.php scrub-host) so the platform hosts only customer accounts. | 16 |
| Oct 21 | `4.2.2 – 4.2.3` | **Added:** I added Fleetio work-order export import, part photos, per-role field-level data hiding (costs, pay, contacts) and bulk tire stock.<br>**Added:** I added a Basic / Advanced view switch and updated the guided tour.<br>**Fixed:** company live data stopped updating when a per-company worker stalled without exiting. I added a watchdog that restarts stalled workers every minute. | 4 |
| Oct 22 | `4.2.4 – 4.2.11` | **Added:** I added a driver card on the live map, custom driver avatars, a phone alert stack, a themed landing page and a copy-from-platform transfer worker (4.2.7).<br>**Fixed:** I fixed "server helper not answering" after an in-app update (4.2.6), Fleetio CSV headers with time-zone suffixes being skipped (4.2.8), "\|"-separated task names splitting on commas and archived records importing as active (4.2.9), and voided fuel entries and meter recounts on first sync (4.2.10).<br>**Added:** I gave every record a stable FleetOverWatch ID (FO-WO-00012) alongside its Fleetio ID, searchable and round-trippable through exports (4.2.11). | 4 |
| | | **Phase total** | **24** |

### Phase 5 — Beta requirements, public API and inventory (v4.3.x)

In this phase I worked through the 4.3 beta specification and the fleet management system reference, and matched Fleetio's work-order and inventory behaviour exactly.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 23 – Oct 26 | `4.3.0` | **Added:** I delivered beta specification sections 29–36: a consolidated alert card, calendar and time pickers everywhere, suggestions and bug reports, and a company forum by department. | 14 |
| Oct 25 – Oct 26 | `4.3.1` | **Changed:** I made parts leave stock when saved on a work order, with every edit, cancel and reopen adjusting the count; negative stock is allowed and flagged, as in Fleetio. | 3 |
| Oct 25 – Oct 26 | `4.3.2` | **Added:** I built the public REST API at /api/v1 (token plus account token), 50+ webhook events with a retry outbox, signed inbound hooks processed asynchronously and idempotently, and an integration health and field-ownership view. | 7 |
| Oct 27 | `4.3.3 – 4.3.5` | **Added:** I put parts from inventory inside the work-order form, attached issues and inspection defects to work orders (completing one resolves them), and matched the parts list to Fleetio's exact columns with a lossless part-export import. | 1.5 |
| Oct 27 | `4.3.6 – 4.3.10` | **Added:** I built employee evaluations per department with admin-built forms, schedules and countdowns (results visible to managers only), photos and PDFs on work orders, and a moderated site-wide forum.<br>**Fixed:** I fixed the company bar overlapping page content, pages cut off at the bottom, and company accounts being labelled "demo". | 1.5 |
| Oct 27 | `4.3.11 – 4.3.15` | **Added:** I added a vehicle status drop-down with reasons, a grouped and colour-coded side menu, a theme pack and a guarded "delete all inventory".<br>**Fixed:** I fixed a demo redirect loop on hosting servers (4.3.14), then changed the demo to run as a normal account with sample data (4.3.15). | 1 |
| Oct 28 | `4.3.16 – 4.3.18` | **Added:** I added a driver roster (one line per truck), a website contact form, an admin record eraser, bulk tire orders with photos and invoices, and host administrator tools. | 2 |
| Oct 28 | `4.3.19 – 4.3.22` | **Added:** I integrated Comdata fuel cards and added vehicle photos as list icons, a one-step company sign-in reset for host admins, more themes, vehicle fuel type and grade, and a seasonal theme. | 2 |
| Oct 29 | `4.3.23` | **Added:** I extended evaluations to any typed name and added PDF and Word export and printable blank forms. | 2 |
| Oct 29 | `4.3.24` | **Added:** I added SMTP email servers (site-wide and per company) with provider presets, masked passwords and a test, plus "Forgot password" with single-use, one-hour reset links. | 2 |
| | | **Phase total** | **36** |

### Phase 6 — Operations, scoring, live map and mobile (v4.4 – v4.5)

My focus shifted to hardening the platform for daily use, adding driver performance tooling and making every screen work on phones and tablets.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Oct 30 – Nov 2 | `4.4.0` | **Added:** I built a server administration console for host admins: health checks with self-repair, an issue tracker with history, admin alerts by category and severity, and IP access control with automatic blocking.<br>**Fixed:** I worked through the corrections list from my 4.3 review. | 22 |
| Nov 2 | `4.4.1` | **Added:** I added Automation and AI feature switches, inventory locations and counts, part lookup with pictures and a server control panel. | 2 |
| Nov 3 | `4.4.2 – 4.4.5` | **Added:** I added Fleetio service reminders and part export, a full Fleetio transfer and barcode scan-to-fill for parts and work orders, made vehicle status follow Fleetio, and wrote a reconciliation report proving every Fleetio record is accounted for. | 1.5 |
| Nov 3 | `4.5.0` | **Added:** I built a shipment board, driver scores and report cards, 50 themes and an interface pass. | 1 |
| Nov 3 | `4.5.1 – 4.5.2` | **Added:** I added grading settings and score resets, truck scores, vehicle classes and shipments on the live map, driver scores in every department, an achievement board and 100 themes. | 1.5 |
| Nov 4 | `4.5.3` | **Fixed:** the CPU was pinned at 70–100%. Profiling showed the admin screens re-scanning directory sizes on every poll. I cached disk sizing for 15 minutes with a time budget, shared a 2-second cache for live readings, tuned SQLite (cache_size, mmap, temp_store) and had the installer write an OPcache profile.<br>**Fixed:** I fixed "database is locked" in the demo account register (schema check once per version, WAL, busy timeout), an undefined worker_age() and SplFileInfo::getSize failures on files deleted mid-scan.<br>**Added:** I added "Clear list" to the issues panel and a new default home location for demo and blank accounts. | 2 |
| Nov 4 | `4.5.4 – 4.5.5` | **Changed:** the product owner rejected 4.5.4, so I rolled it back. 4.5.5 is 4.5.3 plus only the driver grade curve (shop-only line items that adjust a driver's score and appear on report cards). | 1 |
| Nov 4 | `4.5.6 – 4.5.7` | **Added:** I opened the grade curve to shop and admins, drew SVG vehicle icons by class (tractor, box truck, pickup, trailer), built a Trailers panel with geofence locations, colour by trailer style and a user-editable Top 5, and added an admin icon size control. | 1 |
| Nov 5 | `4.5.8` | **Changed:** I did a full mobile pass on iOS Safari and Android Chrome, auditing every page and role with Playwright device emulation (iPhone SE, iPhone 13, Pixel 7): 16 px inputs to stop iOS zoom, scrollable tab strips, sticky modal footers, a phone fleet sheet and no horizontal overflow at 320 px.<br>**Added:** I added per-asset icon and colour overrides with 20 shapes (flatbed, dry van, RGN, port chassis, tanker and more). | 1.5 |
| Nov 5 | `4.5.9` | **Added:** I added a per-user icon style (pictures, classic pins, class badges, dots, heading arrows) and size in Themes, with an admin company default.<br>**Changed:** on phones I turned the vehicle details into a draggable bottom sheet (peek and full); the map recentres the selected truck in the visible area and lifts its controls above the sheet. | 1 |
| Nov 5 | `4.5.10` | **Fixed:** Gmail SMTP returned 535 with valid credentials. I traced it to app passwords pasted with Google's display spaces and to browser autofill writing the site login into the SMTP fields. I normalised the password, locked the fields until focused and added a Gmail-specific diagnosis. | 0.5 |
| Nov 6 | `4.5.11` | **Added:** I added Share location and Directions (Google Maps, Apple Maps, Waze, Android geo:) for trucks and trailers, using the native share sheet where available.<br>**Changed:** I moved alerts out of the content area (a side-menu bell and bottom-left cards; on phones a bottom line that stays quiet while details are open), made the details panel solid, enlarged touch targets and added an Android download link to the landing page. | 1.5 |
| Nov 6 | `4.5.12` | **Changed:** on phones I moved the Basic / Advanced switch from the top bar to the top of the menu. | 0.5 |
| Nov 6 | `4.5.13` | **Fixed:** admin alerts were not arriving by email because delivery failures were being swallowed. The alert path now records the outcome (no site mail server, no recipients, or the SMTP server's reply), and the admin page and test button report it. | 1 |
| | | **Phase total** | **38** |

### Mobile clients (Android and iOS)

I built the mobile apps as thin native shells around the hosted web app, so every server release reaches phones without an app update. I only wrote native code for what a browser cannot do.

| Date (2026) | Version | Changes | Hours |
|---|---|---|---:|
| Nov 5 | `App 1.0 – 1.1` | **Added:** I created the Android (Kotlin, minSdk 24) and iOS (Swift, iOS 15, XcodeGen) projects with camera and photo upload, geolocation, downloads, printing, external links to native apps, an offline screen, an MDM-managed server address and app shortcuts per department.<br>**Added:** I set up a GitHub Actions pipeline that builds the Android APK and AAB and the iOS archive and IPA, with signing from repository secrets.<br>**Changed:** I widened the scope from drivers to all employees, including admin and every department. | 1 |
| Nov 6 | `App 1.2 – 1.3` | **Fixed:** I fixed two CI failures: a starter workflow looking for gradlew at the repo root, and setup-android trying to install the removed "tools" package.<br>**Changed:** I built the server address into the app so it opens straight to sign-in.<br>**Added:** I added a native share sheet, haptics for alerts, keep-screen-on and a biometric app lock (BiometricPrompt on Android, LocalAuthentication on iOS). | 1 |
| | | **Phase total** | **2** |

## Incident log

These are the production and build defects I think are worth recording, with what I found and what I did about each.

| ID | Date | Version | Symptom | Root cause | Resolution |
|---|---|---|---|---|---|
| INC-01 | Oct 16 | 4.0.2 | Samsara DVIR requests returned HTTP 400 | The API rejects long date ranges | I added adaptive windowing (30/7/1 days) and moved to the newer stream endpoints |
| INC-02 | Oct 16 | 4.0.5 | DVIR photos missing after a day | Vendor photo URLs expire after 24 h | I had the worker copy photos to local storage within seconds |
| INC-03 | Oct 21 | 4.2.3 | Company live data stopped updating | A per-company worker stalled without exiting | I added a watchdog that restarts stalled workers every minute |
| INC-04 | Oct 22 | 4.2.6 | "Server helper not answering" after an in-app update | The installer started the worker under the wrong user | I made the installer restart the worker as the web user |
| INC-05 | Oct 22 | 4.2.8 – 4.2.10 | Fleetio CSV imports dropped data | Header suffixes, comma splitting, archived and voided rows | I normalised headers and mapped row states; re-imports are lossless |
| INC-06 | Oct 27 | 4.3.14 | Demo "redirected you too many times" | Section routing on hosting servers | I fixed the routing and moved the demo to a normal sandboxed account |
| INC-07 | Nov 4 | 4.5.3 | CPU pinned at 70–100% | Directory scans on every admin poll; no OPcache tuning | I added cached sizing with a time budget, a shared reading cache, and SQLite and OPcache tuning |
| INC-08 | Nov 4 | 4.5.3 | "database is locked" | A schema check on every request against a busy register | I gated the schema check by version and added WAL and a busy timeout |
| INC-09 | Nov 4 | 4.5.3 | Debug log line found in admin code | A leftover diagnostic write to /tmp | I removed it and made the build reject any package containing debug markers |
| INC-10 | Nov 5 | 4.5.10 | Gmail SMTP 535 with correct credentials | App-password spaces; browser autofill | I added password normalisation, an autofill lock and a provider-specific diagnosis |
| INC-11 | Nov 6 | App 1.2 | Android CI build failed | Wrong workflow; deprecated SDK package | I wrote a project-aware workflow that uses the preinstalled SDK |
| INC-12 | Nov 6 | 4.5.13 | Admin alert emails not arriving | Delivery errors swallowed silently | I now record the outcome and surface it on the admin page and in the test |

## Engineering practices

A few habits carried the project. If you are building something similar on your own, these are the ones I would keep.

- **Release pipeline.** I ship each release as a single self-extracting update-x.y.z.sh (plus a full installer) that carries the whole program, a checksum, an automatic backup and --rollback. My build refuses to package configuration, customer data, analytics logs or debug markers.
- **Testing.** I keep an acceptance suite (127 automated checks at 4.5.13, one block per requirement) that runs on every build against a throwaway server with mock SMTP and vendor endpoints. I check UI work with Playwright on desktop and emulated phones.
- **Install verification.** Before anything ships, I install it over the previous release and onto a clean directory, and both have to pass.
- **Security.** I enforce authorization on the server for every route, hash passwords with bcrypt, never send secrets to the browser or write them to logs, mask credential fields, guard outbound fetches against SSRF, store card numbers as the last four digits only, show temporary passwords once, and run a secret scan on every package.
- **Documentation.** Every release ships with upgrade notes, in-app release notes, user manual updates, an API route list, a data dictionary and a requirements traceability matrix.
- **Change control.** I track requirements row by row in the traceability matrix. When the product owner rejects a change, I roll it back as a new release (4.5.4 to 4.5.5) rather than patching over it.

Thanks for reading. If you have questions about any of these decisions, or ideas for doing them better, I would like to hear them.

---

jBravo · November 7, 2026
