# CLAUDE.md — First Meal Back

Working name (the product name is still undecided). Read this file and `ROADMAP.md` before starting any task. `ROADMAP.md` holds the decisions log, phases, and backlog.

## What this is

A site for finding local, independent-spirited restaurants that people love, built around one question:

> "If you moved away for 10 years and came back for one day, where's the first place you'd eat?"

The only user signal is a **vote**. There are no reviews, comments, star ratings, or questions. Ever.

## Product rules (do not violate)

1. **Browsing is open to everyone.** Only voting (and adding a place, which includes a first vote) requires an account.
2. **One vote per user per restaurant.** Enforce with a database unique constraint on `(userId, restaurantId)`, not just application logic. Vote endpoints are idempotent.
3. **No comments, reviews, star ratings, or free-text user content about restaurants.** If a feature seems to need one, stop and ask.
4. **No vote cap at launch.** A cap is in the backlog. Do not build one.
5. **One combined ranking** by total votes. There are no Locals/Visitors tabs. City pages have **Top Voted** and **Newest**.
6. **Chains are not blocked or reported.** Keep a nullable `isChain` field in the schema but do not use it yet.
7. **Every vote stores a snapshot of the voter's home city and region** at the time of the vote. It is not shown in the UI. Changing home city later never rewrites old votes.
8. **Duplicate rule:** if a new entry's address (including suite/stall number) matches an existing restaurant, block creation and send the user to vote on the existing one. Admin can override.
9. **Adding a place** searches our database first, then the external place provider, and only then falls back to manual entry (name + full address, geocoded). New entries go live immediately. There is no approval queue.
10. **Moderation is minimal:** admins can remove an entry and ban an account (by provider account ID and email). **There are no IP bans.**
11. **Banned users' votes are excluded from all tallies but never deleted.** Vote counts are **computed at read time** from votes of active accounts (with short caching). Do not store a vote counter and do not build a recompute job. Unbanning restores the votes automatically.
12. **Votes are private.** No public profiles in the MVP. Display name is optional, nullable, and non-unique.
13. **Location is always user-initiated and opt-in.** Coordinates are used once and never stored. No IP-based location guessing.
14. **No fake or pre-seeded data in production.** Mock data is for development only and must never ship or be seeded to a production database.
15. **No background jobs or schedulers.** If a feature seems to need one, find a design that doesn't (compute on read, refresh lazily on view) or ask.
16. **Ads and analytics never influence rankings or ordering.** No restaurant can pay for placement.
17. **Never send user identifiers** (user ID, email, display name, home city) **to analytics or ad networks.**

## Audit log and privacy

- Log vote casts and retractions, restaurant creation and removal, and all admin actions, with timestamp, acting user, and target.
- **Do not store IP addresses in the database.** IPs exist only in host/server logs.
- Anything that stores personal data (email, home city/state, third-party sign-in) must be reflected in the privacy policy. The policy must also disclose that server logs contain IP addresses. Flag it if you add something new.

## SEO, analytics, and ads

### SEO (build it in, do not bolt it on)
- Every public page has a unique title and meta description, a canonical URL, and Open Graph/Twitter metadata.
- Generate `sitemap.xml` (use a sitemap index as it grows) and `robots.txt`.
- Add JSON-LD structured data: schema.org `Restaurant` on restaurant pages and `ItemList` on city pages. **Never add `AggregateRating` or review markup.** There are no ratings.
- City pages with fewer restaurants than the minimum (a config constant, start at 3) must be `noindex` and are indexed automatically once they qualify. The site launches empty, so thin pages must not get indexed.
- Sort and tab variants (e.g. `?sort=newest`) canonicalize to the base page.
- When a restaurant is removed, redirect to its city page or return 410. Do not leave broken indexed URLs.
- Protect Core Web Vitals: fast server-rendered HTML, reserved space for images and ads, no layout shift.

### Analytics and consent
- Google Analytics 4 with consent mode. **Load it only after consent**, using a lazy script strategy that does not block rendering.
- Never load analytics on admin pages. Never send user identifiers (rule 17). Anonymous aggregate events (e.g. `vote_cast` without a user ID) are fine.
- A consent management platform is required (EEA/UK consent, US opt-out, Global Privacy Control). Anything that adds cookies or third-party scripts must be reflected in the privacy policy.

### Ads (built in but off at launch)
- Render ads only through a single `AdSlot` component with fixed reserved dimensions, controlled by a feature flag that is **off by default**.
- Ads load only after consent and never block rendering.
- Never place an ad next to the vote button or other tap targets. Never show ads on sign-in, add-a-place, profile, or admin pages.
- Serve `ads.txt` from the site root when ads are enabled.
- If an ad placement hurts Core Web Vitals, remove it.

## Stack

| Layer | Choice |
|-------|--------|
| App | **Single Next.js app** (App Router), TypeScript strict, Tailwind CSS. One repo, one deployment |
| Backend | Next.js route handlers (public JSON endpoints) and server actions (form mutations) |
| Database | PostgreSQL with PostGIS + JSONB. Data layer: Drizzle (leaning) or Prisma. Use raw SQL for geo queries where needed |
| Validation | Zod (or equivalent), shared by handlers and forms |
| Auth | Google and Apple sign-in primary, email fallback with verification. Library vs hosted provider is still to be decided. Dev-only stub login until then |
| Local dev | Docker Compose (app + Postgres/PostGIS) |
| Hosting | **Container-based**: the app ships as one Docker image (Next.js standalone output) plus managed Postgres with PostGIS. Cloud: **Azure** (Container Apps, PostgreSQL Flexible Server, Container Registry, Key Vault, Log Analytics). Region is a Terraform variable and not yet chosen |
| Infrastructure | **Terraform** in `/infra` (keep code OpenTofu-compatible). See the Infrastructure section |
| Rejected | Blazor, ASP.NET Razor + HTMX, and a split Next.js + .NET API. Do not reintroduce them |

## Repo layout

```
/app      Next.js routes, pages, route handlers
/lib      domain logic, data access, place providers
/db       schema and migrations
/infra    Terraform (modules, environments)
/docs     notes, decisions, design tokens
ROADMAP.md
CLAUDE.md
```

## Infrastructure

- All infrastructure is defined in Terraform under `/infra`. **No manual console changes.**
- Region and environment values must be variables, never hard-coded. Provider-specific resources stay **inside their modules**, and application code never calls cloud SDKs directly except behind an interface in `/lib`, so a move to another cloud stays a matter of rewriting `/infra`.
- Do not build a multi-cloud abstraction or write modules for other clouds unless asked.
- Organize as reusable modules (network, database, registry, app hosting, secrets, logging) with separate `dev`, `staging`, and `prod` environments.
- Use remote state with locking. **Never commit `.tfstate`, `.tfvars` containing secrets, or credentials.** Mark secret variables and outputs as sensitive.
- The app must be container-friendly: Next.js standalone output, a non-root Dockerfile, configuration only through environment variables, and a `/healthz` endpoint that checks the database.
- Production database has deletion protection and tested backups. Log retention is set explicitly in Terraform and must match the privacy policy.
- **You may run `terraform fmt`, `validate`, and `plan`. Never run `terraform apply` or `destroy`, and never change prod state, unless the owner explicitly asks in that moment.** Show the plan and wait.
- Pin provider and Terraform versions. Prefer small, readable modules over clever abstractions.

## Conventions

### Backend
- **Business rules live in plain, testable modules under `/lib`**, not inside route handlers or server actions. Handlers validate input, call a `/lib` function, and return the result.
- JSON only. camelCase property names. Dates are ISO 8601 in UTC.
- IDs are prefixed ULID strings (`rst_`, `usr_`, `vte_`, `aud_`, `cty_`).
- Place data goes behind a `PlaceProvider` interface. Phase 1 uses a mock implementation. Never call a real provider directly from handlers.
- Persist provider data only in line with that provider's terms. Google Places data may only be stored as a Place ID.
- Validate all input at the edge. Return RFC 7807 problem-details style errors from JSON endpoints.
- Rate limit vote and add-restaurant endpoints using a shared store. In-memory limits break as soon as there is more than one container instance.
- Use repository interfaces so mock data can be swapped for the real database through configuration.
- Keep database connections pool-friendly, and size the pool for a small number of container instances.
- Design JSON endpoints as a real API from the start, so a future native app can reuse them.

### Front end
- Mobile-first. Design for ~375px width, then scale up.
- Clean and minimal: lots of whitespace, few colors, one typeface, spacing from the design tokens in `/docs`.
- Server-render city and restaurant pages. SEO matters. Use server components by default and add client components only for real interactivity.
- Clean shareable URLs (e.g. `/c/warren-oh`, `/r/marias-kitchen-warren-oh`) with proper metadata.
- Accessibility is required: labels, focus states, contrast, keyboard navigation.
- Build loading skeletons and empty states. **Empty states are important** because the site launches with no data. They should invite the first contribution (e.g. "Be the first to add a place in Warren").
- Keep it fast: minimal client JavaScript, optimized images, no unnecessary dependencies.

### Mock data (development only)
- Lives in JSON files under `/lib/data/mock`, loaded through the same repository interfaces as the real database.
- About 5 cities, ~15 restaurants each, varied vote counts, some entries missing hours or phone numbers.

## Testing
- Unit test every vote rule: uniqueness, idempotency, retraction, and banned-vote exclusion.
- Test address-match duplicate blocking, including suite/stall variations.
- Integration-test the unique vote constraint against a real Postgres from Docker Compose.
- Keep tests fast and runnable with a single `npm test`.

## Working agreement

- **Keep `ROADMAP.md` current.** When a task is done, tick its checkbox and add a Done Log line with the date.
- Work in small, reviewable commits with clear messages.
- Do not build backlog items. Ask before adding a major dependency, changing a decision in the decisions log, or building anything outside the roadmap.
- Ask before changing anything in `/infra` that would alter running production resources.
- Prefer simple solutions. This is solo-maintained, so avoid clever architecture one person can't comfortably own.
- The owner will not review most code line by line, so lean on tests, database constraints, and clear naming to keep behavior verifiable.
- If a request conflicts with the product rules above, say so rather than quietly working around it.
