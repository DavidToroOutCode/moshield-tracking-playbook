Mosquito Shield — Internal Team Handoff Report
> Date: 2026-04-19 (rewritten for Plug & Play handoff)
> Author: David Toro (Analytics / GA4 — outgoing)
> Audience: Diego Tarazona, Josue Farfan (primary), future squad members
> Purpose: Single self-contained master document. Everything needed to maintain, extend, or diagnose the Mosquito Shield tracking system lives here. No external links required.
> Status: PRODUCTION LIVE — Analytics Engine v2.2 deployed. Funnel tracking active on PA, MI, NJ. 12 franchises pre-configured in code.
***0. TL;DR — 60-second read
If something breaks: start with Moziquito.debug() in browser console, then Section 11 Troubleshooting Runbook; if still stuck, see Appendix B — Escalation Guide.
***🧭 Table of Contents
Introduction & Context — Why this exists and who owns it.
Systems & Access — GA4, GTM, WP Engine, and CRM metadata.
Technical Architecture — Diagrams, layers, and data models.
Implementation Checklist — Events, dimensions, and ecommerce rules.
The Code Map — Where every line of tracking lives.
The Deployment Workflow — Staging to Production flow.
Change Management — Historical context and recent fixes.
Status & Backlog — Blockers and future roadmap.
Troubleshooting Runner — Diagnostic tools and common fixes.
THE RUNBOOK: Add a Franchise — START HERE for scaling.
Operating Principles — Coding and safety rules.
Onboarding Guide — Day-one setup.
Appendices — Comms history, Escalations, and Gotchas.
***1. Why this document exists
In March 2026 the project lost two key people (Carlos Lenon went on vacation, Peter Martinez left with no handoff). When David joined on March 31, he had to reconstruct the tracking state from scratch with zero documentation. That cost roughly two weeks of avoidable work.
This report exists so that it does not happen again. If someone new joins the squad tomorrow, just by reading this document they should be able to:
Understand the full architecture.
Request access to every system they need (and know who to ask).
Deploy a change without breaking anything.
Add a new franchise on their own (Section 12).
Diagnose 90% of common problems.
The document is deliberately self-contained: no external wiki, no required secondary reads. Everything is here.
***2. Squad and contacts
Person	Role	Responsibility	Status
**Josue Farfan**	Backend Dev / SFTP	WordPress theme edits, SFTP to production, Pocomos integration, checkout.js	**Primary owner after handoff**
**Diego Tarazona**	Project Lead	Client-facing, access management, escalation, WP Engine coordination	**Primary owner after handoff**
**Mark (WP Engine)**	DevOps	Purge All Caches (WP Engine + NitroPack), server config	Through Diego
**Juan Oliveira**	Client (Five Star Franchising)	Business owner, uses GA4 reports, approves changes	Technical client — monitors dashboards actively
**David Blinn (FSF)**	Corporate Ops	Franchisee comms, Pocomos admin (office_id, tax_code_id), Inspectlet vendor liaison	Escalation for franchisee setup
**David Toro**	Analytics / GA4 (outgoing)	Handoff author. Available for technical questions during transition.	Outgoing
Peter Martinez	Original GA4 implementer	Out — no handoff	—
Carlos Lenon	Previous GA4 lead	On leave since April 1	—
Golden rule: before touching production, announce it in the Teams channel "Five Star Franch. Chat Interno en Espanol". Post-deploy always requires a Purge All Caches (Diego → Mark).
***3. System architecture — overview
The system has four layers:
Identification layer (Plug & Play engine in header-ecommerce.php) — reads window.Moziquito.Config.franchises, detects the URL, and pushes region_identified to the dataLayer. Also conditionally activates Inspectlet.
Behavior layer (checkout.js + GTM variables) — tracks user interaction inside the checkout form.
Enrichment layer (GTM tag CJS Remapper) — intercepts dataLayer.push, adds Elite parameters, re-maps events.
Reporting layer (GA4 + Inspectlet + Pocomos) — final destinations for the data.
3.1 Sequence diagram — flow of a sale (zip → purchase)
sequenceDiagram
    autonumber
    participant User
    participant Browser as Browser (Moziquito Engine v2.2)
    participant Inspectlet as Inspectlet (Session Recording)
    participant GTM as GTM Container (Events & Dedup Guard)
    participant GA4 as GA4 Master (Property 321414391)
    participant Pocomos as Pocomos (Franchisee API)
    Note over User,Pocomos: 1. Identification Phase (Digital Fingerprint)
    User->>Browser: Lands on /southwest-michigan/
    Browser->>Browser: Match slug in Moziquito.Config.franchises
    Browser->>GTM: push(region_identified, moshield_region, is_test)
    Browser->>Inspectlet: Activate (if enabled in config)
    Browser->>GA4: Set franchise_id & state (event-scoped)
    Note over User,Pocomos: 2. Behavior Phase (Interactive Tracking)
    User->>Browser: Interacts with zipcode/agreements
    Browser->>GTM: push(form_field_complete: field_name)
    Browser->>Inspectlet: Tag session with user_email & region
    Note over User,Pocomos: 3. Elite Conversion Phase (Revenue Capture)
    User->>Browser: Click 'Complete Checkout'
    Browser->>Pocomos: Process Payment (use office_id & tax_code_id)
    Pocomos-->>Browser: Response: SUCCESS (transaction_id: TX-123)
    
    par Dual-Stack Reporting
        Browser->>GTM: push(purchase: value, currency, transaction_id)
        GTM->>GTM: Dedup Guard (Validates unique transaction_id)
        GTM->>GA4: Record Conversion (Elite dimensions attached)
        Browser->>GA4: gtag(direct backup: en=purchase)
    end
    Note over User,Pocomos: 4. Security Buffer (2s delay)
    Browser->>Browser: Wait 2000ms (Ensures packets reach Google/Inspectlet)
    Browser->>User: Redirect to /thank-you/
3.2 Class diagram — Moziquito Engine v2.2 internals
classDiagram
    class MoziquitoEngine {
        +Config config
        +Object state
        +debug()  // Console tool for Diego
        +init()
    }
    class Config {
        +String gtm_id
        +String ga4_id
        +Number inspectlet_id
        +Object franchises {slug, name, id, state, inspectlet, pocomos}
    }
    class RegionalDetector {
        +String current_path
        +detectMatchedFranchise()
        +pushRegionEvent()
    }
    class InspectletActivator {
        +loadSDK()
        +tagSessionWithMetadata()
    }
    class ElitePurchaseReporter {
        +float transaction_value
        +string transaction_id
        +reportToGA4Direct()
        +reportToDataLayer()
        +enforce2sBuffer()
    }
    MoziquitoEngine *-- Config
    MoziquitoEngine *-- RegionalDetector
    MoziquitoEngine *-- InspectletActivator
    MoziquitoEngine *-- ElitePurchaseReporter
3.3 Data model — entities and relationships
erDiagram
    USER ||--o{ FORM_SESSION : starts
    USER ||--|| FRANCHISE : identified_by_url
    FRANCHISE ||--o{ TRANSACTION : generates
    AGREEMENT ||--o{ TRANSACTION : sold_in
    TRANSACTION ||--|| GA4_REPORT : synced_to
    USER {
        string first_name
        string user_email
        string booking_zip
    }
    FRANCHISE {
        int office_id
        int tax_code_id
        string moshield_region
        bool is_live_tracking
    }
    TRANSACTION {
        string transaction_id
        float final_value
        string offer_name
        bool is_test_environment
        string sale_type
    }
    GA4_REPORT {
        string measurement_id
        string moshield_region
        string franchise_id
        float revenue_value
    }
***4. Access and credentials
> Never share credentials in chat. Use the team password manager. Ask Diego for access.
4.1 GA4 (Google Analytics 4)
Property	ID	Measurement ID	Role	Who has access
Mosquito Shield GA4	`321414391`	`G-91NESGFMXH`	**PRODUCTION / CLIENT-FACING** — Juan reads his reports here	David, Diego, Juan
Moshield.com - GTM Debugging	`485312869`	`G-CP5MFDP8JV`	Legacy — 16K+ historical users. No new data flows since April 12	David, Diego
How to access: analytics.google.com → Mosquito Shield account → property 321414391.
4.2 GTM (Google Tag Manager)
Container	ID	Owning account	Role
Moshield Main	`GTM-KSL4KJ3F`	`peakteamoutcode@gmail.com`	**ACTIVE** — all tracking tags live here
General Tracking	`GTM-5W76HWW`	Other account (no access)	Legacy, pageview only
Active version: v22 (published April 18, 2026 — "Purchase Dedup Guard: ghost events removed, native GA4 deduplication via transaction_id").
How to access: tagmanager.google.com with the peakteamoutcode@gmail.com account.
4.3 WordPress / WP Engine
Environment	URL	WP Admin access	SFTP access	Notes
Staging	`moshieldstg.wpengine.com`	OK (David, Josue, Diego)	OK	Theme Editor available
Production	`moshield.com`	**BLOCKED (403)** for David	Josue only	Theme Editor not reachable from David's IP
Rule: changes are edited in staging first via Theme Editor, then Josue pushes them to production via SFTP. After the deploy, Diego/Mark must run Purge All Caches in WP Engine (otherwise NitroPack serves a stale cached version).
4.4 Inspectlet
Website ID (wid): 708588779 (corrected from anomaly 1610427321 on Apr 17 — see Section 9).
Dashboard: inspectlet.com — requires a verification code from David Blinn (vendor). Pending scheduling through Diego.
Activation: driven by franchise.inspectlet: true/false in Moziquito.Config.franchises (header-ecommerce.php). Currently ON for PA, MI, NJ.
4.5 Pocomos (CRM / billing)
Not our system. Josue owns the franchisee relationship.
Each franchise has its own office_id and tax_code_id, documented in Moziquito.Config.franchises as pocomos: { office_id, tax_code_id }.
Michigan office_id: 1477 — currently has tax_code_id=9680 ("No Tax") — pending franchisee configuration fix by David Blinn (FSF).
PA office_id: to be documented (Josue — fill the null placeholder when available).
Southern NJ office_id: to be documented (Josue).
4.6 ClickUp
Main tickets:
FSF-504 — Revenue fix (COMPLETED Apr 4-8)
FSF-505 — GA4/GTM stabilization (IN PROGRESS)
FSF-473 — GA + Inspectlet Michigan (COMPLETED Apr 14)
Access through Diego.
***5. GA4 — detailed configuration
5.1 Active property: 321414391 (G-91NESGFMXH)
This is where Juan checks his weekly reports. All traffic from April 12 onward lands here.
5.2 Key events
Event	Fires when	Fired by
`region_identified`	On page load on every page	header-ecommerce.php (Plug & Play engine)
`zip_code_entered`	User submits zip on the landing page	Landing page + CJS Remapper
`service_selected`	User picks a plan in the booking flow	Checkout UI + CJS Remapper
`checkout_started`	User reaches the payment step	checkout.js
`form_start`	First focus on any checkout field	checkout.js (`initFormTracking`)
`form_field_complete`	Blur on a field with a non-empty value	checkout.js
`form_abandon`	`beforeunload` without a completed purchase	checkout.js (sendBeacon)
`purchase`	Successful transaction with Pocomos	checkout.js (dataLayer + gtag + 2s buffer + **Dedup Guard v22**)
`form_submit`	Contact forms (non-booking)	Standard GTM trigger
`UA_Click_To_Call_Local` / `UA_Click_To_Call_National`	Phone number click	Standard GTM trigger
5.3 Registered custom dimensions (event-scoped)
> If they are not registered in GA4 Admin, parameters will not show up in reports or in the dimension autocomplete.
Dimension name	Parameter	Scope	Purpose
Moshield Region	`moshield_region`	Event	Display name of the franchise (e.g. "Southwest Michigan")
Franchise ID	`franchise_id`	Event	Stable slug (e.g. "sw-mi")
Franchise State	`franchise_state`	Event	State code (PA, MI, NJ)
Page Type	`page_type`	Event	`lander`, `booking_funnel`, `thank_you`
Funnel Step	`funnel_step`	Event	`zip`, `service`, `checkout`, `purchase`
Offer Name	`offer_name`	Event	Dynamic service name + payment type
Sale Name	`sale_name`	Event	`Promotional` / `One Time` / `Installments`
Booking Zip	`booking_zip`	Event	Zip code at checkout
Booking Service	`booking_service`	Event	Selected service
Booking Plan	`booking_plan`	Event	Payment plan
Is Test	`is_test`	Event	`'true'` on staging or `?test=1`, `'false'` otherwise — filters QA traffic out of Juan's reports
5.4 Ecommerce / Revenue
value is sent with every purchase event along with currency: 'USD'.
items array contains item_id, item_name, price, quantity.
Dual-push (dataLayer + direct gtag) with a 2-second buffer before redirecting to /thank-you/ (prevents the race condition that previously caused $0 revenue).
Dedup Guard (GTM v22, Apr 18 2026): A premature "ghost" purchase event used to fire on button click BEFORE the Pocomos server confirmed the sale, inflating event counts by up to 7×. This has been fixed — the ghost event is removed and a guard in GTM (CJS - Purchase Dedup Guard) plus native GA4 transaction_id deduplication ensure 1 sale = 1 purchase event. Revenue value was always correct; only the event count was inflated.
***6. GTM — container GTM-KSL4KJ3F
6.1 Active tags (state as of April 19, 2026)
Tag	Type	Trigger	Purpose
**TAG - HTML - Regional Dimension Injector**	Custom HTML	Initialization - All Pages	Backup injector (the primary one is inline in header-ecommerce.php). **Scheduled to be paused in v23** — produces a cosmetic double `region_identified` push (see Appendix C).
**CJS Remapper**	Custom JavaScript (variable)	Used in multiple tags	Intercepts dataLayer.push and adds Elite params
**GA4 Config**	Google Tag (G-91NESGFMXH)	All Pages	Base GA4 configuration
**GA4 - Event - zip_code_entered**	GA4 Event	CE - zip_code_entered	Sends event + Elite params
**GA4 - Event - service_selected**	GA4 Event	CE - service_selected	Sends event + Elite params
**GA4 - Event - checkout_started**	GA4 Event	CE - checkout_started	Sends event + Elite params
**CJS - Purchase Dedup Guard**	Custom JS	`purchase`	**Added v22** — prevents double-counting; allows only confirmed Pocomos transactions by transaction_id
**CJS Booking Flow DataLayer Events**	Custom (legacy)	Various funnel	Cleaned up v22 — premature purchase pushes removed
Legacy form tags (form_start, form_field_focus, etc.)	GA4 Event	Custom events	Used for contact forms, NOT for checkout
6.2 Custom variables
Variable	Type	Returns
`CJS Remapper`	Custom JavaScript	Function that enriches events
`DLV - moshield_region`	Data Layer Variable	region from `region_identified`
`DLV - franchise_id`	Data Layer Variable	franchise slug
`DLV - funnel_step`	Data Layer Variable	current funnel step
`DLV - is_test`	Data Layer Variable	'true' on staging/test, 'false' on prod
(other standard DLVs)	Data Layer Variable	zip, service, plan, price, transaction_id, etc.
6.3 Active triggers
Initialization - All Pages — fires the Regional Injector first.
CE - zip_code_entered, CE - service_selected, CE - checkout_started, CE - purchase — funnel custom events.
Form Start, Form Abandon, Form Field Complete, etc. — used for contact forms.
6.4 Critical execution order
1. Initialization     → Regional Injector (inline + GTM backup)  ← MUST FIRE FIRST
2. All Pages          → GA4 Config
3. Custom Events      → GA4 Event tags (use the CJS Remapper variable)
4. Purchase           → Dedup Guard validates transaction_id before firing GA4 event
5. Page Redirect      → checkout.js applies the 2s buffer
If the Regional Injector does not fire first, downstream events arrive with moshield_region: 'General'.
***7. Code map (key files)
These files carry every piece of tracking logic on the site. They live inside the ituza-child WordPress theme and are edited through the WP Theme Editor (staging) or via SFTP (production).
File	Location in WP	Purpose
`header-ecommerce.php`	`wp-content/themes/ituza-child/header-ecommerce.php`	Header used by booking pages. Loads GA4, GTM, the **Plug & Play Moziquito engine** (franchise detection + Inspectlet activation)
`checkout.js`	`wp-content/themes/ituza-child/app/js/checkout.js`	Checkout logic + form tracking + purchase dual-push
`page-ecommerce-v2.php`	`wp-content/themes/ituza-child/page-ecommerce-v2.php`	Booking page template (inherits tracking from the header)
`ecommerce.css`	`wp-content/themes/ituza-child/ecommerce/ecommerce.css`	Ecommerce styling including the `$699` strikethrough pricing rules
The CJS Remapper, Regional Dimension Injector (backup), CJS - Purchase Dedup Guard, and all GA4 Event tags live inside the GTM container (GTM-KSL4KJ3F), not in WordPress. Edit them directly in the GTM UI.
***8. Deploy process (staging → production)
8.1 Standard flow
[5] Verify production
     └─ Load https://moshield.com/<slug>/ → Console → `Moziquito.debug()` — is_test must be 'false'
     └─ GA4 Realtime: confirm events arrive with the correct moshield_region
### 8.3 Visual Promotion Flow
```mermaid
graph TD
    A[Edit code in Staging Header] -->|Validate| B{Moziquito.debug OK?}
    B -- No --> A
    B -- Yes --> C[Josue SFTP to Prod]
    C --> D[Mark: Purge All Caches]
    D --> E[Validate Prod URL]
    E -->|Success| F[Announce in Teams]
    E -->|Error| G[Rollback using .bak file]
8.2 Golden rules
Never edit directly in production — David does not have access (403) and it is unsafe anyway.
The 2-second buffer before redirect is mandatory. If you touch it, validate that purchase still reaches GA4.
Always Purge All Caches after SFTP. NitroPack caches aggressively.
One single property (321414391) for production. Do not mix with the legacy debugging property.
Only publish GTM once the WP code is already in production — if a tag references a new dataLayer key that does not exist on the site yet, events arrive empty.
Always back up the current file before editing. Josue: SFTP down → rename to <file>.bak-YYYYMMDD → edit copy → SFTP up.
***9. Change history (last 50 days)
Date	Change	Owner
Mar 31	Initial diagnosis: Revenue $0.00, GTM pointing to debugging property	David
Apr 4-6	Revenue fix: added dataLayer + gtag + 2s buffer in checkout.js (FSF-504)	David + Josue
Apr 8	GTM v14 published with G-91NESGFMXH config	David
Apr 9-10	Operation Cleanup: 6 custom dimensions + Regional Injector + Elite params	David
Apr 10	OTP double-discount fix (price = full_price)	David + Josue
Apr 10	SW Michigan franchise added to code	David (GTM v16)
Apr 10	Meeting with Juan: OTP fix approved, GA4 confirmed, Juan flagged absence after May 12	All
Apr 10	Inspectlet forced activation fix	David
Apr 12	Form tracking (form_start, form_field_complete, form_abandon) + dynamic offer_name	David
Apr 12	Property unification: staging and production both send to 321414391	David
Apr 13	Initial Internal Handoff Report delivered	David
Apr 14	Josue SFTP to prod: header-ecommerce.php + checkout.js	Josue 12:09 PM
Apr 14	Mark NitroPack Purge All Caches — tracking live in production	Mark (via Diego) ~4:00 PM
Apr 14	Diego created FSF-473 (GA + Inspectlet Michigan)	Diego
Apr 14	Juan requested: use "Test Test" as name for QA transactions	Juan
Apr 15	Juan reports 3 issues: Inspectlet not recording PA, $699 not struck-through PA, MI taxes $0	Juan
Apr 15	Evidence capture (Pocomos JSON for MI office 1477 returns `tax_code=9680 "No Tax"`)	David
Apr 16	Juan updates to 6 issues (adds: payment error Southern NJ, Get Free Quote back on thank-you PA, GA4 vs Caputo discrepancy)	Juan 2:01 PM
Apr 16	David assigns 3-file deploy to Josue: ecommerce.css, header-ecommerce.php, page-ecommerce-v2.php	David 8:52 AM
Apr 16	Josue SFTP complete; Mark NitroPack purge	Josue + Mark
Apr 16	Duplicate Inspectlet snippet removed from page-ecommerce-v2.php (race condition)	David 6:56 PM
Apr 17	Juan sends "GA Discrepancies" email directly to `david.toro@outcodesoftware.com`	Juan 4:29 PM
Apr 17	Root cause found: `/online-booking/` page hardcoded with `wid=1610427321` but dashboard is `wid=708588779`	David 4:11 PM
Apr 17	WID fix deployed to staging via Theme Editor; curl-verified	David 4:03 PM
Apr 17	`/southern-nj/` added to Regional Dimension Injector v1.2 + Inspectlet whitelist	David
Apr 18	GTM v22 published — Purchase Dedup Guard: ghost events removed, 1 sale = 1 event	David
Apr 18	Inspectlet verified on staging: human session recorded, 17-page funnel capture confirmed	David
Apr 18	Closure messages sent to Juan: Inspectlet fix + GTM Dedup Guard	David
Apr 19	**Plug & Play refactor (Engine v2.2):** `Moziquito.Config.franchises` table with 12 pre-declared franchises + `Moziquito.debug()` helper + auto `is_test` detection + conditional Inspectlet activation	David
Apr 19	**This handoff document rewritten** for self-contained delivery to Diego + Josue	David
For per-commit detail, see git log on the developer branch.
***10. Known issues and backlog
10.1 External blockers (waiting on others — NOT our code)
#	Issue	Owner	Status
B2	Michigan `office_id=1477` has `tax_code_id=9680` "No Tax" in Pocomos	**David Blinn (FSF)**	🔴 Evidence captured. Blocks real MI purchases
B3	Inspectlet dashboard access (verification code)	Diego + David Blinn	🟡 Schedule call to get vendor code
B4	PA transactional emails for booking not sending	Josue + David Blinn	🟡 Likely Pocomos config — Josue investigating
B5	Southern NJ payment processing error at checkout	Josue	🟡 Old template — under review
10.2 Internal backlog (prioritized)
#	Issue	Priority	Effort	Notes
N1	`offer_name` still hardcoded in CJS Remapper (GTM) — should read `window.selectedAgreement`	High	Low (15 min)	Affects attribution, not revenue amount. Publish as GTM v19 quick fix
N2	Retire GTM "TAG - HTML - Regional Dimension Injector" backup tag	Medium	Low (10 min)	Double-fires `region_identified` (cosmetic — see Appendix C). Pause it in GTM v23
N3	Looker Studio Dashboard for Juan	High	Medium (4-6 h)	Next visible deliverable for client
N4	Central Data Dictionary document	Medium	Low (2-3 h)	All events, params, dims — centralizes knowledge
N5	GA4 → BigQuery export	Medium	Low (30 min config + 24h wait)	Advanced analysis capability
N6	Consent Mode V2	Medium	Medium (3-4 h)	GDPR/CCPA compliance
N7	Server-side GTM (sGTM) — Phase 3, Q3 2026	Low (future)	High	Stape scan 29/100. Requires Google Cloud + sGTM container + contract renewal
10.3 Resolved (kept here for historical reference)
#	Issue	Resolved
R1	Revenue $0.00	Apr 4-8 (dual-push + 2s buffer)
R2	OTP double-discount	Apr 10
R3	Purchase event triplication (3x per transaction)	Apr 18 (GTM v22 Dedup Guard)
R4	Inspectlet not recording `/online-booking/`	Apr 17-18 (WID swap `1610427321` → `708588779`)
R5	Southern NJ tracking missing	Apr 17 (added to Regional Injector v1.2)
R6	`$699` not struck-through in PA	Apr 16 (CSS redeploy)
R7	"Get a Free Quote" button on PA thank-you page	Apr 17 (CSS + cache purge)
R8	`is_test` URL-based auto-detection	Apr 19 (Plug & Play v2.2 — automatic on `wpengine.com` or `?test=1`)
***11. Troubleshooting Runbook
11.1 First step ALWAYS: Moziquito.debug()
Before anything else, open DevTools Console on the problem page and run:
Moziquito.debug()
This prints the current detection state: URL path, matched franchise, page type, is_test flag, Inspectlet active, Pocomos config, and the last 5 dataLayer events. 80% of tracking issues are visible here.
11.2 "GA4 shows no staging/production data"
Open GA4 Realtime on property 321414391.
Filter by page_location contains staging or production.
If nothing appears: confirm the header has G-91NESGFMXH active via console: typeof gtag === 'function'.
If the script is there but no events fire: NitroPack is likely caching. Request a Purge All Caches.
11.3 "Purchase does not register revenue"
Open DevTools → Network → filter by collect.
Complete a test purchase.
You should see a request to google-analytics.com/g/collect with en=purchase&ep.value=....
If the request is missing: verify the 2s buffer is still in checkout.js. Without it, the redirect kills the request.
11.4 "moshield_region arrives as 'General'"
Run Moziquito.debug() — if Matched franchise: ❌ no match, the URL doesn't match any entry.
Check window.Moziquito.Config.franchises — does a key exist for the current URL path?
Slug must match exactly, including leading and trailing /. Example: key /southwest-michigan/ matches URL /southwest-michigan/online-booking/.
If the key IS there and still no match: NitroPack may be serving a stale header. Purge cache.
11.5 "Duplicate purchase events in GA4"
Should no longer happen post GTM v22. If you see >1 purchase per transaction:
Open GA4 Exploration → filter by unique transaction_id — confirm you only see one.
If you see multiple, check that GTM v22 is the published version (not rolled back).
Check that the Dedup Guard tag CJS - Purchase Dedup Guard is enabled.
11.6 "Inspectlet does not record sessions"
Confirm Moziquito.state.inspectlet_active === true on that page.
If true but not recording: NitroPack changed the script type to nitropack/inlinescript. Purge cache.
If inspectlet_active is false: check Moziquito.Config.franchises['/<slug>/'].inspectlet — should be true for live franchises.
Bot detection: headless browsers (Playwright, Selenium) are blocked by Inspectlet server-side — they return {"noinspectlet":"crawlbot"}. Only real human sessions in a normal browser are recorded. This is expected.
11.7 "Staging works but production is broken"
99% of the time: NitroPack is serving a stale version. Purge All Caches, wait ~1 minute, retest.
11.8 "WP Admin returns 403 in production"
Normal for David (IP blocked by WP Engine). Workarounds:
Ask Josue to edit via SFTP.
Ask Diego to whitelist the IP (not recommended for security).
Use staging and promote via SFTP.
11.9 "dataLayer.filter(e=>e.event==='region_identified').length returns 2"
Expected. The inline injector in header-ecommerce.php AND the GTM backup tag both push. Cosmetic — revenue is unaffected (region_identified has no value). Plan is to pause the GTM backup in v23 (N2 in backlog). See Appendix C.
***12. How to add a new franchise — Plug & Play (THE runbook)
If you only read one section of this document before going live with a new franchise, read THIS ONE.
12.1 The Plug & Play Workflow
flowchart LR
    A[Request office_id from DB] --> B[Add Line to Header-ecommerce.php]
    B --> C[Set inspectlet flag true/false]
    C --> D[Test with debug tool]
    D --> E[Promote to Production]
    E --> F[Purge Cache]
    F --> G[Done: Revenue is now tracked]
12.2 Concept: ONE table, ONE edit
All franchise detection logic lives in a single table at the top of header-ecommerce.php:
window.Moziquito.Config = {
  gtm_id: 'GTM-KSL4KJ3F',
  ga4_id: 'G-91NESGFMXH',
  inspectlet_id: 708588779,
  franchises: {
    '/southeastern-pa/':      { name: 'Southeastern Pennsylvania', id: 'se-pa', state: 'PA', inspectlet: true,  pocomos: { office_id: null, tax_code_id: null } },
    '/southwest-michigan/':   { name: 'Southwest Michigan',        id: 'sw-mi', state: 'MI', inspectlet: true,  pocomos: { office_id: 1477, tax_code_id: 9680 } },
    '/southern-nj/':          { name: 'Southern New Jersey',       id: 'so-nj', state: 'NJ', inspectlet: true,  pocomos: { office_id: null, tax_code_id: null } },
    // ... 9 more pre-declared
  }
};
To add a new franchise → add ONE line. The detection engine + Inspectlet activation + GA4 dimension pushes all adapt automatically. No GTM changes. No GA4 changes. No JavaScript changes elsewhere.
12.2 Prerequisites (before you touch code)
The franchise URL exists in WordPress (e.g., /fort-worth/) — confirm loading https://moshield.com/fort-worth/ returns a valid page.
Josue has the Pocomos office_id from the franchisee. If missing, leave null and keep inspectlet: false until ready.
Agreements/products configured in Pocomos for that office (Josue validates).
Franchisee is actually ready to go live — confirmed by Diego / David Blinn / Juan.
12.3 The one-line edit
Edit header-ecommerce.php in staging first via WP Theme Editor (moshieldstg.wpengine.com/wp-admin → Appearance → Theme File Editor → ituza-child → header-ecommerce.php).
Find the franchises block. Already pre-declared (just need to activate when ready): Central Florida, Central Indiana, Southwest Fort Worth, North Central NJ, Central Tampa, North Salt Lake City, Twin Cities North, North Dallas, Palm Beaches.
If the franchise is pre-declared (most common case): flip inspectlet: false → true and fill pocomos:
'/central-tampa/': { name: 'Central Tampa', id: 'ce-fl', state: 'FL', inspectlet: true, pocomos: { office_id: <from Josue>, tax_code_id: <from Josue> } },
If the franchise is NOT pre-declared (new slug): add a new row. The URL key must start AND end with /:
'/palm-beaches/': {
  name: 'Palm Beaches',        // What shows in Juan's GA4 reports (moshield_region)
  id: 'pb-fl',                  // Short slug (franchise_id) — 5 chars, state suffix
  state: 'FL',                  // 2-letter state code (franchise_state)
  inspectlet: true,             // true to enable session recording
  pocomos: { office_id: <id>, tax_code_id: <id> }  // From Josue
},
12.4 Field reference
Field	Type	Purpose	Example	How to pick
`name`	string	Display name in GA4 reports	'Fort Worth'	Whatever Juan uses to refer to it
`id`	string	Stable slug for GA4 filtering	'ft-worth'	5-char style, state suffix (e.g. 'sw-mi', 'ce-fl')
`state`	string (2)	US state code	'TX'	Standard 2-letter
`inspectlet`	boolean	Enable session recording	`true`	Keep `false` until Pocomos ready
`pocomos.office_id`	number	Pocomos internal ID	`1477`	Josue gets from franchisee
`pocomos.tax_code_id`	number	Tax code from franchisee setup	`9680`	Josue gets from franchisee
12.5 Validate in staging (3 minutes)
Load the franchise URL in staging: https://moshieldstg.wpengine.com/<slug>/
Open DevTools Console (F12 → Console)
Run:
Moziquito.debug()
Expected output:
Moziquito Debug
  URL path          : /<slug>/
  Matched franchise : <Name>  (id=<id>, state=<ST>)
  Page type         : lander
  Test mode (is_test): true          ← staging, so true is correct
  Inspectlet active : <true|false>   ← must match what you set
  Pocomos config    : {office_id: <id>, tax_code_id: <id>}
  All franchises    : [12 slugs listed]
  Last 5 dataLayer  : [...]
If Matched franchise : ❌ no match → your slug key doesn't match the URL. Confirm exact slug with leading/trailing /.
12.6 Deploy to production (Josue)
Josue: back up first — SFTP download current prod header-ecommerce.php → save as header-ecommerce.php.bak-YYYYMMDD.
Josue: SFTP staging header-ecommerce.php → production path wp-content/themes/ituza-child/header-ecommerce.php.
Diego: ping Mark to run Purge All Caches in WP Engine + NitroPack (crítico — sin purge los cambios no se ven).
Validate prod: load https://moshield.com/<slug>/, Console → Moziquito.debug() — should show the franchise matched AND is_test: false.
GA4 Realtime check: filter by moshield_region = <Name> → confirm events arriving.
Inspectlet dashboard (if inspectlet: true): real human browser test → wait 5 min → confirm session appears in dashboard.
12.7 Announce in Teams
In "Five Star Franch. Chat Interno en Espanol":
> "Franquicia {Name} desplegada en producción. Tracking validado:
>
> - Moziquito.debug() → matched ✓
> - GA4 Realtime → eventos llegando con moshield_region=<Name> ✓
> - Inspectlet → {grabando / off}
>
> Visible en reports desde ahora."
12.8 Rollback (if something breaks)
If post-deploy you see broken tracking on OTHER franchises:
Josue: SFTP the backup file back (header-ecommerce.php.bak-YYYYMMDD → header-ecommerce.php).
Diego: Purge All Caches again.
Confirm pre-existing live franchises work: Moziquito.debug() on each.
Investigate in staging before retrying.
12.9 Common errors
Symptom	Probable cause	Fix
`Moziquito is undefined` in console	Header script didn't load (JS error earlier)	Check console for a previous error
`Matched franchise: ❌ no match`	Slug key doesn't match URL	Confirm slug with leading/trailing `/`
Inspectlet not recording	`inspectlet: false` or cache stale	Flip to `true`, purge cache
Production still shows old code	NitroPack cache	Diego → Mark → Purge All Caches
`is_test: true` on production	URL contains `wpengine.com` or `?test=1`	Review URL — shouldn't happen on real prod
`pocomos.office_id: null` showing in debug	Pocomos config not filled	Normal for pre-declared franchises until Josue confirms
12.10 Special case: activating Inspectlet later
If you deploy with inspectlet: false initially and want to enable later:
Staging edit: flip to true.
Deploy: SFTP + purge.
Validate: real human session on the URL → check Inspectlet dashboard for new session.
No other changes needed — the engine activates the Inspectlet SDK automatically.
***13. Operating principles (non-negotiable)
Do not break what works. Juan already has data. Every change ships in staging first, validated, then to production.
Keep the 2-second buffer before redirect. Without it, revenue is lost.
Purge All Caches after every deploy. Always.
One property (321414391) for production. Do not mix with the debugging property.
Regional Injector must fire first. Everything else depends on it.
Namespace everything under window.Moziquito. No globals outside this namespace.
No DOM scraping for pricing/IDs. Pull from Pocomos or selectedAgreement.
Announce in Teams before touching production.
Back up before editing. Josue always downloads current prod file first.
If in doubt, use Moziquito.debug() first.
***14. Onboarding checklist for a new squad member
Day-one actions for anyone joining the squad:
Ask Diego for access: GA4 (321414391), GTM (GTM-KSL4KJ3F), WP Engine (staging at minimum), ClickUp, Teams chat.
Read this document end to end.
Open GA4 property 321414391 → Realtime → watch for 5 minutes to get familiar with the live event stream.
Open GTM container GTM-KSL4KJ3F → review v22 tags.
Load https://moshield.com/southeastern-pa/ → DevTools Console → run Moziquito.debug() → confirm you understand the output.
Load https://moshieldstg.wpengine.com/southeastern-pa/ → Moziquito.debug() → confirm is_test: true on staging.
Run a test purchase in staging with name "Test Test" → confirm purchase event reaches GA4 DebugView with every Elite parameter populated.
Read Appendix A to understand the client relationship and past incidents.
Read Appendix B to know who/where to escalate.
Introduce yourself in Teams "Five Star Franch. Chat Interno en Espanol".
***Appendix A — Client communication history (cronological)
This is a self-contained summary of major interactions with Juan (and Mark Castaldo / David Blinn where relevant). Use it to understand project history, past incidents, and recurring themes.
March 2026 — Project inception
David Toro joined the project on March 31 after Peter Martinez left without handoff and Carlos Lenon went on leave. No documentation existed.
Initial call with Diego + David + Juan (+ Peter on one call): Juan articulated that the Thank You Page must be the primary conversion event, and that Southeastern PA was the initial focus for e-commerce tracking.
April 4-8 — Revenue fix (FSF-504)
Problem: GA4 was showing $0.00 revenue despite confirmed transactions on Pocomos.
Root cause: purchase event was being dispatched via dataLayer.push just before a redirect to /thank-you/ — the redirect killed the request before GA4 could send it.
Fix: dual-push (dataLayer.push + direct gtag('event', 'purchase', ...)) + 2-second buffer before redirect.
Outcome: Juan confirmed $24K+ in revenue visible in GA4 by April 8.
April 9-10 — Operation Cleanup (Elite Schema)
Introduced 6 new custom dimensions (moshield_region, franchise_id, franchise_state, page_type, funnel_step, offer_name).
Added the Regional Dimension Injector (inline in header-ecommerce.php).
Meeting with Juan (Apr 10): approved OTP pricing fix, confirmed GA4 architecture, flagged absence after May 12.
GTM v16 published.
April 12-14 — Michigan rollout + form tracking
Southwest Michigan franchise added to code.
Form tracking rolled out (form_start, form_field_complete, form_abandon).
offer_name made dynamic (reads window.selectedAgreement).
GA4 properties unified: staging + production both send to 321414391.
Apr 14: Josue SFTP deploy to prod + Mark NitroPack purge → tracking confirmed live.
Juan requested: use "Test" in first/last name for QA transactions so they can be filtered.
Diego created FSF-473 (GA + Inspectlet for Michigan).
April 15 — Juan reports 3 issues
Juan reported three concurrent issues:
Inspectlet not recording PA booking funnel (despite fixes the day before).
$699 not struck-through in PA pricing (.price-original CSS rule missing in prod).
MI taxes showing $0.00 (Pocomos office_id=1477 returns tax_code_id=9680 "No Tax").
Root cause investigation began. Playwright agents revealed the initial hypotheses were wrong — browsers are bot-detected by Inspectlet server-side.
April 16 — Juan updates to 6 issues
At 2:01 PM Juan posted a full list. The three previous issues (Apr 15) remained open, plus three new ones:
(Issue 4) Payment processing error in Southern NJ checkout (old template).
(Issue 5) "Get a Free Quote" button reappeared on PA thank-you page (regression).
(Issue 6) GA4 vs Caputo Dashboard discrepancy (Juan sent a separate email about this to Mark Castaldo).
David assigned a 3-file deploy to Josue: ecommerce.css (strikethrough), header-ecommerce.php (Inspectlet tracking), page-ecommerce-v2.php (script inheritance). Also instructed Mark to purge cache and asked David Blinn about MI tax_code_id.
Later that night David removed a duplicate Inspectlet snippet in page-ecommerce-v2.php that was creating a race condition (WID was being injected twice).
April 17 — Root cause of Inspectlet issue found
Juan pushed back on initial "quota exhausted" hypothesis with evidence from the Inspectlet dashboard (plan: 25K sessions/month, used only 2,112 in 3 months).
After a second incorrect hypothesis (heavy-JS blocking the async loader), the real root cause was found:
/online-booking/ page had wid=1610427321 hardcoded.
The dashboard uses wid=708588779 (matches all other PA pages).
So booking recordings were being sent to a different account the whole time.
David swapped the WID on staging via Theme Editor, curl-verified.
Same day, 4:29 PM: Juan sent "GA Discrepancies" email directly to david.toro@outcodesoftware.com with full context about the Caputo Dashboard difference (3 conversions in Caputo vs 440 views + 14 purchases in GA4). This cemented the Apr 22 stakeholder call agenda.
Southern NJ was also added to the Regional Dimension Injector (v1.2).
April 18 — Two major fixes confirmed
Inspectlet fix (PM):
Human session recorded on staging — inspectlet.com/dashboard/watchsession/708588779/1372933713
Real user session on staging with full 17-page funnel captured.
GA4 events (booking_step_view, region_identified) visible as session tags → cross-tool integration confirmed.
Purchase deduplication fix:
Investigation revealed GTM was firing a "ghost" purchase event on button click BEFORE the Pocomos server confirmed the sale — inflating counts by up to 7×.
Revenue value was always correct; only the event count was inflated.
GTM v22 published:
Ghost event removed from "CJS Booking Flow".
CJS - Purchase Dedup Guard tag added.
Native GA4 deduplication via transaction_id confirmed.
Result: 1 sale = 1 purchase event in GA4.
David sent closure messages to Juan for both topics the same evening.
April 19 — Plug & Play refactor + this handoff
header-ecommerce.php refactored to use a single window.Moziquito.Config.franchises table with 12 pre-declared franchises (3 live + 9 pre-declared).
Moziquito.debug() helper added for Diego/Josue self-service validation.
Auto is_test detection for staging + ?test=1 query param.
Conditional Inspectlet activation driven by franchise.inspectlet flag.
Pocomos metadata (office_id, tax_code_id) attached to each franchise entry.
This document rewritten as the single self-contained handoff reference.
Upcoming milestones
Apr 22 (Wed): Stakeholder call with Mark Castaldo (Caputo), Juan, Diego, David, David Blinn — GA Discrepancies alignment.
After May 12: Juan partially unavailable — critical decisions should land before this.
***Appendix B — Escalation guide
When Juan reports an issue, use this table to orient yourself. First column = what Juan says; second = what to check first; third = who to escalate to.
If Juan reports / asks...	Check first	Escalate to
"Revenue looks wrong"	GA4 Realtime → confirm `purchase` events with `value` present. Explorations → dedupe by `transaction_id`	Analytics consultant (David if still reachable)
"Not seeing sessions from franchise X"	`Moziquito.debug()` on that URL + check `franchises[key].inspectlet`	David or Josue
"Inspectlet didn't record session Y"	`Moziquito.state.inspectlet_active === true`? Cache purged? Real human browser (not headless)?	Inspectlet support via David Blinn
"Numbers don't match Caputo Dashboard"	**Known**: Caputo uses GA4 **Landing Pages**, Juan's screenshots are **Pages and Screens** — different definitions. See Appendix A (Apr 16-18).	No escalation — explain the definitional difference
"New franchise not tracking"	`Moziquito.debug()` → matched? Entry in `Config.franchises`? Production cache purged?	Josue (deploy/cache)
"Taxes showing $0 in franchise X"	Pocomos `tax_code_id` for that `office_id`	**David Blinn (FSF)** — not a code bug
"Get a Free Quote button reappeared on thank-you page"	CSS `.thank-you-view` class applied? `/ecommerce.css` has `.free-quote-btn { display: none }` rule? Cache?	Josue (CSS) + Mark (cache)
"Purchase event fires multiple times"	Should NOT happen post GTM v22. Check GTM published version + Dedup Guard enabled	Analytics consultant
"Payment error in checkout"	Payment gateway / Pocomos response — check network tab	Josue (Pocomos)
"Email confirmation not arriving"	Pocomos email configuration for that office	Josue + David Blinn
Golden escalation rule: if you don't know, post in Teams with what you already checked. Never stay silent — Juan monitors the system actively and responds quickly.
***Appendix C — Known gotchas (things that LOOK like bugs but are expected)
Before reporting something as a bug, check if it's one of these:
Observation	Expected?	Why
`dataLayer.filter(e=>e.event==='region_identified').length === 2`	✅ Yes (cosmetic)	The inline injector in `header-ecommerce.php` AND the GTM backup tag (`TAG - HTML - Regional Dimension Injector`) both push. Introduced during Operation Cleanup (Apr 9-10) as redundancy. Doesn't affect revenue. Plan: pause the GTM backup in v23 (backlog N2).
`is_test: 'true'` in staging	✅ Yes	Auto-detection triggers on `wpengine.com` host or `?test=1`. Correct — keeps QA out of Juan's reports.
`pocomos.office_id: null` in debug for pre-declared franchises	✅ Yes	Placeholder. Josue fills when the franchisee is ready.
Purchase event fires 2 network requests (`dataLayer` + `gtag`)	✅ Yes	Intentional dual-push for race-condition protection (Apr 4-8 fix). GA4 natively deduplicates by `transaction_id`. 1 sale = 1 purchase in reports.
`Moziquito.debug()` returns a state object in console	✅ Yes	The function prints the grouped output AND returns `Moziquito.state` for scripts.
Inspectlet blocks Playwright/headless browsers	✅ Yes	Server-side bot detection returns `{"noinspectlet":"crawlbot"}`. Only real human browsers record. Use a real browser for Inspectlet validation.
`is_test` sometimes showing as string `'true'` vs boolean `true`	✅ Yes	The dataLayer push normalizes to string `'true'`/`'false'` for GA4 compatibility. `Moziquito.state.is_test` is boolean.
PA franchise shows `pocomos.office_id: null`	⚠️ Partially	PA is live but Pocomos ID not yet documented in code. Josue to fill. No operational impact — it's metadata, not used at runtime.
`region_identified` event a
