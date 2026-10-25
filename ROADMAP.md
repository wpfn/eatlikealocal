# First Meal Back — Project Roadmap

> Working name (see D8). Core question: **"If you moved away for 10 years and came back for one day, where's the first place you'd eat?"**

Mark items done by changing `[ ]` to `[x]`. Add a date and short note to the Done Log when closing something out.

---

## Product Principles

- One signal only: **a vote.** No comments, no reviews, no star ratings, no questions.
- One vote per user per restaurant.
- Browsing is open to everyone. **Only voting requires an account.**
- Independents are the spirit of the site, but **chains are not blocked.** Natural voting sinks them.
- Keep it simple. Minimal admin burden: this is a solo project.
- Clean, fast, mobile-first.
- All API payloads are JSON.
- **No fake or pre-seeded data in production.** The site launches empty and earns its reputation. Mock data is for development only.
- **No background jobs.** Design so nothing needs a scheduler.
- **All infrastructure is code (Terraform).** No click-ops. Region and provider are variables.
- **SEO is built in, not bolted on.** Analytics and ads load only after consent, and **ads never influence rankings.**

---

## Decisions Log

| # | Decision | Status | Notes |
|---|----------|--------|-------|
| D1 | Stack | **Decided** | **One Next.js app** (App Router, TypeScript strict, Tailwind). Route handlers and server actions are the backend. One repo, one deployment |
| D2 | Alternatives rejected | **Decided** | Blazor (prior bad experience with DOM control and JS interop). ASP.NET Razor + HTMX (less headroom). Split Next.js + .NET API (two deployments, cross-origin auth, type syncing) |
| D3 | Database | **Decided (provisional)** | PostgreSQL + PostGIS + JSONB. Choose a managed host that supports PostGIS. Data layer: Drizzle (leaning) or Prisma. Geo queries may need raw SQL |
| D4 | Place data source | **Decided (approach)** | `PlaceProvider` interface, mock now. Pick the real provider in Phase 2 after testing coverage in 3 real cities. Candidates: OSM, Foursquare. Google lookup-only (Place ID storage only) |
| D5 | Auth | **Decided (approach)** | Google + Apple sign-in primary, email fallback (password or magic link, decide in Phase 1). Library (Auth.js) vs hosted provider (Clerk/Auth0): decide at start of Phase 1. Dev-only stub login until then |
| D6 | Hosting | **Open** | **Container-based** (single Next.js container + managed Postgres with PostGIS), so the app is portable and Terraform controls where it runs. Cloud: Azure (D17). Vercel is no longer the plan |
| D7 | Vote model | **Decided** | One combined ranking by total votes. No Locals/Visitors tabs. No vote cap at launch (backlog). Each vote stores a snapshot of the voter's home city/state |
| D8 | Product name | **Open** | "First Meal Back" is a working title. Needs `.com` and trademark check. "Eat Local" rejected: widely used, including an Ohio restaurant app. Name is not blocking the build |
| D9 | New entries | **Decided** | Go live immediately. No approval queue |
| D10 | Chains | **Decided** | Not blocked, not reported. Nullable `isChain` kept in schema for a future filter |
| D11 | Moderation | **Decided** | Remove entries; ban by account (provider ID + email). **No IP bans.** Banned users' votes are excluded from tallies (not deleted) |
| D12 | Audit log | **Decided** | Records votes, retractions, entry changes, and admin actions with user, target, and timestamp. **IPs are not stored in the database.** IPs exist only in host/server logs, used for investigation if needed |
| D13 | Profiles | **Decided (MVP)** | Votes are private. No public profiles. Display name optional, non-unique, nullable. Public profiles/follow are backlog |
| D14 | Launch data | **Decided** | No pre-seeding. Launch empty with great empty states |
| D15 | Tallies | **Decided** | Counts computed at read time from votes of active accounts, with short caching. No stored counter, no recompute job. Ban/unban takes effect immediately |
| D16 | Background work | **Decided** | None. Stale place data refreshes lazily when viewed, not on a schedule |
| D17 | Cloud provider and region | **Decided (provider); region open** | **Azure** to start: Container Apps, PostgreSQL Flexible Server (with PostGIS), Container Registry, Key Vault, Log Analytics. Terraform modules are provider-specific, so moving clouds later means rewriting `/infra` and migrating data (days, not a rebuild), because the app is a container with plain Postgres and env-var config. Region is a Terraform variable: pick a US region near users, and verify Flexible Server and PostGIS availability and pricing there first |
| D18 | Infrastructure as code | **Decided** | Terraform in `/infra` (OpenTofu-compatible code). Remote state with locking, separate dev/staging/prod environments, no secrets or state in git, plan on every PR. **Apply is run by the owner or an approved pipeline step, never automatically by an AI assistant** |
| D19 | SEO | **Decided** | Server-rendered pages, unique titles/descriptions, canonical URLs, sitemap + robots, schema.org structured data (Restaurant, ItemList), Open Graph cards. **No fake ratings markup** (we have no ratings). Pages below a minimum restaurant count (configurable, start at 3) are `noindex` until they qualify |
| D20 | Analytics | **Decided** | Google Analytics 4 with consent mode, loaded only after consent. Plus Google Search Console. No user IDs, emails, or home city sent to analytics. Not on admin pages |
| D21 | Monetization | **Decided (approach)** | Third-party display ads (network TBD, AdSense is the likely start). Ad slots built into the layout from the start behind a feature flag, **off at launch**. Enable after launch once there is content, traffic, and network approval. Ads never affect ranking. Restaurant-paid placement is **not** planned (backlog, conflicts with the vote-only principle) |
| D22 | Consent | **Decided** | A consent management platform is required for analytics and ads, including EEA/UK handling and US opt-out/Global Privacy Control. Privacy policy covers cookies, analytics, and advertising |

---

## Core Rules (summary)

1. Browsing: open. Voting and adding a place: account required.
2. Adding a place = add it and cast your first vote.
3. **Duplicate rule:** if the address (including suite/stall number) matches an existing entry, creation is blocked and the user is sent to vote on the existing one. Admin can override (e.g. food halls).
4. Search order when adding: our database first, then the external provider, then manual entry (name + full address, geocoded).
5. Votes are never hard-deleted by moderation. A banned account's votes are excluded from tallies; unbanning restores them.
6. Changing home city/state affects only future votes. Past votes keep the snapshot they were cast with.
7. Business rules live in plain, tested modules (`/lib`), not inside route handlers.

---

## Phase 0 — Foundations

- [x] Choose stack (D1): single Next.js app (2026-10-24)
- [ ] Create GitHub repo, branching strategy, README
- [ ] Commit `ROADMAP.md` and `CLAUDE.md` to repo root
- [ ] Install Claude Code extension in VS Code and open the repo
- [ ] Scaffold Next.js (TypeScript strict, Tailwind, App Router)
- [ ] Folder structure: `/app`, `/lib` (domain, data, providers), `/db` (schema, migrations), `/infra` (Terraform), `/docs`
- [ ] Dockerfile for the app (Next.js standalone output, small image, non-root user)
- [ ] Health-check endpoint (`/healthz`) that also verifies database connectivity
- [ ] All configuration through environment variables (no environment-specific code)
- [ ] Docker Compose dev environment (app + Postgres/PostGIS)
- [ ] Choose data layer (Drizzle or Prisma) and set up migrations
- [ ] Test runner (Vitest suggested) and a single `npm test` command
- [ ] CI pipeline (lint, type-check, test on PR)
- [ ] Design tokens: colors, type scale, spacing (minimal and clean)
- [ ] Input validation approach (Zod suggested) shared by handlers and forms

## Phase 1 — MVP (mocked data)

### Data and API
- [ ] Mock dataset (dev only): ~5 cities, ~15 restaurants each, varied vote counts, some missing hours/phone
- [ ] Repository interfaces so mock data can be swapped for the real database via configuration
- [ ] City search endpoint
- [ ] City restaurant list endpoint (`sort=top|newest`)
- [ ] Restaurant detail endpoint
- [ ] Vote (idempotent) and retract-vote
- [ ] Place search (our database first, then external provider)
- [ ] Create restaurant (from external place or manual; includes address-match duplicate block)
- [ ] Tallies computed at read time from active accounts' votes, with short caching (D15)
- [ ] Rate limiting on vote and add endpoints (use a shared store, since in-memory limits don't work across serverless instances)

### Accounts
- [ ] Decide auth library vs hosted provider (D5)
- [ ] Sign in with Google
- [ ] Sign in with Apple
- [ ] Email fallback with verification before first vote
- [ ] Sign out
- [ ] Preferences (JSON): home city and state, editable any time
- [ ] Optional display name (non-unique, nullable, not public)
- [ ] Short auto-generated user ID

### Pages
- [ ] Landing page: hero, city search box, featured cities
- [ ] City page: **Top Voted** and **Newest** tabs
- [ ] Restaurant card/detail: name, address, phone, hours, website, map link, cuisine, vote button and count
- [ ] Add-a-restaurant flow
- [ ] Profile/settings: home city and state, my votes (private)
- [ ] Clean shareable URLs and page metadata (e.g. `/c/warren-oh`, `/r/marias-kitchen-warren-oh`)
- [ ] **Empty states that invite the first contribution** ("Be the first to add a place in Warren")
- [ ] Loading skeletons
- [ ] 404 and error pages

### SEO foundations (built in from the start)
- [ ] Unique `<title>` and meta description for every public page (landing, city, restaurant)
- [ ] Canonical URLs; sort/tab variants (`?sort=newest`) canonicalize to the base page
- [ ] `sitemap.xml` (sitemap index split by state/city as it grows) and `robots.txt`
- [ ] Structured data (JSON-LD): schema.org `Restaurant` on restaurant pages (name, address, phone, hours, geo) and `ItemList` on city pages. No `AggregateRating`
- [ ] Open Graph and Twitter card metadata
- [ ] `noindex` for city pages below the minimum restaurant count (config constant), indexed automatically once they qualify
- [ ] Internal linking: landing, state, city, restaurant, and nearby cities
- [ ] Sensible handling when a restaurant is removed (redirect to its city page or return 410)
- [ ] Core Web Vitals budget (fast load, no layout shift) checked on mobile
- [ ] State pages (e.g. `/s/ohio`) listing cities that qualify for indexing

### Monetization-ready layout
- [ ] `AdSlot` component with reserved dimensions (no layout shift), **disabled by a feature flag**
- [ ] Defined placements (e.g. between list items on city pages, bottom of restaurant page). Never adjacent to the vote button or other tap targets, and never on sign-in, add-a-place, profile, or admin pages
- [ ] Ad and analytics scripts load only after consent and never block rendering

### Quality
- [ ] Mobile-first layout verified on real devices
- [ ] Accessibility pass (contrast, focus, labels, keyboard)
- [ ] Unit tests for vote rules (uniqueness, idempotency, retraction, banned-vote exclusion)
- [ ] Tests for address-match duplicate blocking
- [ ] Database-level unique constraint on `(userId, restaurantId)` verified by test

## Phase 1.5 — Infrastructure (Terraform)

Needed before real sign-in testing (Google and Apple require real redirect URLs) and before Phase 2.

- [x] Choose cloud provider: Azure (D17, 2026-10-24)
- [ ] Choose Azure region (variable; verify PostgreSQL Flexible Server + PostGIS availability and pricing there)
- [ ] Keep provider-specific code inside modules and document each module's inputs/outputs, so an equivalent for another cloud could be written later. Do not build a multi-cloud abstraction
- [ ] Terraform remote state backend with locking
- [ ] Environments: `dev`, `staging`, `prod` (separate state and variables; region and provider-specific values in `tfvars`)
- [ ] Module: network (Azure virtual network; private access between app and database)
- [ ] Module: Azure Database for PostgreSQL Flexible Server with PostGIS enabled; automated backups and a tested restore
- [ ] Module: Azure Container Registry
- [ ] Module: Azure Container Apps (HTTPS, autoscaling limits, health check)
- [ ] Module: Key Vault for auth keys, Apple sign-in key, database credentials (no secrets in git or tfstate output)
- [ ] Module: Log Analytics workspace with explicit retention (this decides how long request IPs exist; match the privacy policy)
- [ ] Azure budget alerts and spend caps
- [ ] Deletion protection on the production database
- [ ] CI/CD (GitHub Actions): build image, push, deploy; Azure auth through workload identity federation (no long-lived keys)
- [ ] `terraform plan` output posted on every PR; apply restricted to a protected branch and manual approval
- [ ] DNS and TLS once the name and domain are settled (D8)
- [ ] Runbook in `/docs`: how to deploy, roll back, restore the database, and rotate secrets

## Phase 2 — Real Data

- [ ] Evaluate place providers in 3 real cities (hours, phone, coverage) and choose (D4)
- [ ] Implement real `PlaceProvider`
- [ ] Normalize provider data to our JSON shape
- [ ] Confirm provider storage terms before persisting anything
- [ ] Move from mock repositories to PostgreSQL + PostGIS
- [ ] Lazy refresh of stale hours when a restaurant is viewed (no scheduler)
- [ ] "Permanently closed" report (a flag, not a review)
- [ ] Finalize auth provider and replace dev stub login

## Phase 3 — Admin, Audit, Legal (required before public launch)

- [ ] Audit log: votes, retractions, entry creation/removal, admin actions; with timestamp, user, target (no IP)
- [ ] Admin: remove an invalid entry
- [ ] Admin: ban an account (provider ID + email); unban
- [ ] Admin access protected (separate role, strong auth)
- [ ] Confirm the log retention set in Terraform matches what the privacy policy says
- [ ] **Privacy policy** (must disclose: server logs include IP addresses and are kept per the host's retention; home city/state storage; third-party sign-in; email; cookies and analytics; advertising and ad partners; how to opt out)
- [ ] Terms of service
- [ ] Consent management platform (analytics and ads): EEA/UK consent, US opt-out and Global Privacy Control signal
- [ ] Google Analytics 4 with consent mode; verify no user identifiers are sent
- [ ] Google Search Console verified; sitemap submitted
- [ ] Serve `ads.txt` from the site root (when ads are enabled)
- [ ] Contact/abuse-report email

## Phase 4 — Location (post-MVP)

All location features are **user-initiated and opt-in**.

- [ ] "Use my location" button on landing page (never prompt on load)
- [ ] Reverse geocode to nearest city and suggest it
- [ ] Pre-fill home city/state at sign-up with a confirm step
- [ ] "Near me" sort on city pages
- [ ] Coordinates used once and **never stored**
- [ ] Graceful behavior when permission is denied
- [ ] No IP-based location guessing

## Phase 5 — Monetization (post-launch)

- [ ] Wait for real content and traffic; confirm enough indexed pages to pass ad network review
- [ ] Apply to an ad network (AdSense likely first); keep the site clean and policy-compliant while waiting
- [ ] Add ad network scripts through the consent flow and publish `ads.txt`
- [ ] Turn on `AdSlot` placements via the feature flag, one placement at a time
- [ ] Measure Core Web Vitals before and after; remove any placement that hurts them
- [ ] Review ad categories and quality controls
- [ ] Track revenue vs. cost (hosting, database, ad-related performance impact)

## Launch Checklist

- [ ] Production contains **no mock data**
- [ ] Privacy policy and terms published
- [ ] Audit log and admin tools working
- [ ] Search Console verified, sitemap submitted, thin city pages confirmed `noindex`
- [ ] Consent banner working; no analytics or ad scripts load before consent
- [ ] Ads feature flag is **off**
- [ ] Domain and name settled (D8)
- [ ] Production infrastructure applied from Terraform only; backups restore-tested; budget alerts on
- [ ] Empty states reviewed on mobile
- [ ] Decide which cities to promote first (your home turf first, real contributors only)

---

## Backlog (undecided, unscheduled)

### Vote model
- [ ] **Per-city vote cap** (e.g. 5). Undecided. Concern: friction when a great new place opens
- [ ] **One-tap "swap out which vote?"** prompt at the cap (pairs with the vote cap)
- [ ] **Hometown field:** a voter marks a hometown that counts as local for that place
- [ ] **Locals vs Visitors view** (data is already captured via the home city/state snapshot)
- [ ] **Locals' favorite badge** (automatic, based on self-declared data; no verification)

### People
- [ ] **Public profiles** (opt-in), following, and seeing others' votes
- [ ] **Taste matching:** "people who like what you like also like..."
- [ ] **Personal top 10 list** (a separate curated, ranked list, not derived from votes)

### Discovery
- [ ] Hide-chains filter (uses nullable `isChain`)
- [ ] Cuisine and price-tier filters (tags, not ratings)
- [ ] "Open now" filter
- [ ] Map view toggle
- [ ] Hidden-gem score
- [ ] City leaderboard by contributors
- [ ] Travel mode: follow a city and get alerts for new picks (would need scheduled work)
- [ ] Weekly digest email (would need scheduled work)

### Restaurant data
- [ ] Optional one-dish "what to order" field (decide if it conflicts with "no comments")
- [ ] Claim this restaurant (owner corrects hours/phone, cannot vote or affect rank)
- [ ] Admin tool to merge duplicates
- [ ] Restaurant-sponsored listings (conflicts with the vote-only principle and ranking integrity; decide carefully, if ever)

### Personal tools
- [ ] Save / want-to-try list (private, separate from votes)
- [ ] Import from Google Takeout / saved places

### Platform
- [ ] Social share preview images for city pages
- [ ] PWA install
- [ ] Native app (would call the same API routes)
- [ ] Multi-language support

---

## Data Model (draft, JSON-first)

```jsonc
// Restaurant
{
  "id": "rst_01HXYZ",
  "name": "Maria's Kitchen",
  "address": {
    "line1": "123 Main St",
    "line2": null,                 // suite/stall; part of duplicate matching
    "city": "Warren",
    "region": "OH",
    "postalCode": "44481",
    "country": "US"
  },
  "location": { "lat": 41.2373, "lng": -80.8184 },
  "phone": "+13305550123",
  "website": "https://example.com",
  "cuisine": ["italian", "family-style"],
  "hours": { "mon": [], "tue": [{ "open": "11:00", "close": "21:00" }] },
  "externalRefs": [{ "provider": "osm", "id": "node/123456" }],
  "status": "open",
  "isChain": null,                 // unused for now; reserved for a future filter
  "createdBy": "usr_01ABC",
  "createdAt": "2026-10-24T12:00:00Z",
  "voteCount": 142                 // computed at read time; excludes banned accounts' votes
}

// Vote (unique on userId + restaurantId; never hard-deleted by moderation)
{
  "id": "vte_01DEF",
  "userId": "usr_01ABC",
  "restaurantId": "rst_01HXYZ",
  "voterHomeCity": "Warren",       // snapshot at time of vote
  "voterHomeRegion": "OH",
  "createdAt": "2026-10-24T12:05:00Z",
  "retractedAt": null
}

// User
{
  "id": "usr_01ABC",
  "shortId": "k7x2f",
  "displayName": null,             // optional, non-unique, not public
  "email": "user@example.com",
  "authProvider": "google",
  "authProviderId": "1234567890",
  "status": "active",              // active | banned
  "preferences": {
    "homeCity": "Warren",
    "homeRegion": "OH"
  },
  "createdAt": "2026-10-24T12:00:00Z"
}

// AuditEvent (no IP address stored)
{
  "id": "aud_01GHI",
  "type": "vote.cast",             // vote.cast | vote.retracted | restaurant.created | restaurant.removed | user.banned | user.unbanned ...
  "actorUserId": "usr_01ABC",
  "targetId": "rst_01HXYZ",
  "createdAt": "2026-10-24T12:05:00Z"
}
```

---

## Data Sources (research needed)

> **Warning:** Google Places API terms restrict scraping and long-term storage of place data (generally only the Place ID may be stored indefinitely). Scraping Google outside the API violates their terms. Confirm current terms before committing.

| Source | Pros | Cons |
|--------|------|------|
| OpenStreetMap (Nominatim / Overpass) | Free, open license, storable | Hours/phone coverage is patchy |
| Foursquare Places API | Rich POI data, free tier | Check storage terms and limits |
| Geoapify / Photon / Mapbox | Good autocomplete | Pricing and storage terms vary |
| Google Places API | Best coverage and hours | Cost, caching restrictions, attribution requirements |
| Manual entry | Always works | Needs address-match duplicate check |

---

## Naming Notes (D8)

- Need a short, catchy, distinctive name with an available `.com` and no trademark conflict. Check in that order, and buy a domain only after both clear.
- "Eat Local" rejected: widely used by small regional products (including an Ohio restaurant app) and too generic to protect. It can still work as a tagline.
- `.us` domains require a US nexus and do not allow WHOIS privacy.
- `ipostyouread.com` is unused and was ruled out as a home for the product.
- Use a neutral codename (`firstmeal`) for the repo until the name is chosen.

---

## Done Log

| Date | Item | Notes |
|------|------|-------|
| 2026-10-24 | Initial roadmap drafted | |
| 2026-10-24 | Planning pass complete: D3-D14 resolved | Vote model, moderation, audit, profiles, chains, launch data |
| 2026-10-24 | Stack simplified to a single Next.js app (D1, D2) | Blazor, HTMX, and split .NET API rejected |
| 2026-10-24 | IPs no longer stored; tallies computed at read time (D12, D15, D16) | No background jobs needed |
| 2026-10-24 | `CLAUDE.md` rewritten for the new stack | |
| 2026-10-24 | Terraform added (D18); hosting moved to containers (D6); Phase 1.5 Infrastructure added | |
| 2026-10-24 | D17 decided: Azure | Region still open |
| 2026-10-24 | SEO, analytics, ads, and consent added (D19-D22); Phase 5 Monetization added | Ads off at launch |
