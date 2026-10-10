# PROJECT BRIEF — Embr

> Product name: **Embr**. Use a `NEXT_PUBLIC_APP_NAME` env var and a single `config/brand.ts` (name, tagline, logo paths, domain) so branding is easy to change. Domain: TBD.

## 1. Product summary

Embr is a web app where a user pastes the link of a **public Notion page** and instantly gets an **embed code (iframe)** to place on any website (WordPress, Webflow, custom sites). The embed is served from our own domain, so we can add what Notion's share links lack: **password protection, email restriction, custom styling, analytics, lead capture, custom domains, and an API**.

Core idea: we fetch the public Notion page, render it ourselves, cache it, and serve it at `/e/[id]`. The customer's site loads that URL in an iframe.

A reference product that inspired the idea: embednotionpages.com. We are building our own product with a better offer, not copying their branding or copy.

## 2. Target users

Course creators, freelancers and agencies, startups with docs or help centers, and anyone using Notion as a CMS who wants it on their own site with access control.

## 3. How we are better (differentiators)

- Free plan that is actually usable: 1 embed, **no expiry** (small "Powered by Embr" badge).
- **Analytics** (views, unique visitors, referrers).
- **Lead capture**: "enter your email to unlock this page", with leads exported to CSV.
- **Custom domains** (`docs.theirbrand.com`) on Business.
- **Reliability**: stale-while-revalidate cache with last-good-copy fallback if Notion is slow or down.
- **SEO landing pages** that bring in traffic for searches like "password protect Notion page".
- A product that feels native to Notion users: the UI is Notion-inspired, so it is instantly familiar.

## 4. Plans and limits

| | **Free** | **Pro — $8/mo** | **Business — $20/mo** |
|---|---|---|---|
| Embeds | 1 | 25 (initial value, keep configurable) | Unlimited |
| Auto-sync with Notion | Yes | Yes | Yes |
| Dark mode, sizing, frame styles | Yes | Yes | Yes |
| "Powered by Embr" badge | Shown | Removed | Removed |
| Password protection | No | Yes | Yes |
| Email restriction (one-time code) | No | Yes | Yes |
| Custom CSS | No | Yes | Yes |
| Basic analytics (views) | No | Yes | Yes |
| Advanced analytics (unique visitors, referrers) | No | No | Yes |
| Lead capture + CSV export | No | No | Yes |
| Custom domain | No | No | Yes |
| API access + API keys | No | No | Yes |
| Bulk import (CSV of Notion URLs) | No | No | Yes |
| Priority support | No | No | Yes |

- Billing intervals: monthly and yearly. Yearly discount: **TBD (owner to confirm; suggested ~20%)**.
- Free-plan embed expiry: implement as a config constant, **off by default**.
- All limits live in `lib/plans.ts` → `getPlanLimits(plan)` and nowhere else.

## 5. Tech stack

- **Framework:** Next.js (App Router) + TypeScript + Tailwind CSS
- **Animation:** Motion (Framer Motion) + CSS transitions
- **Auth + DB:** Supabase (email + Google sign-in, Postgres, Row Level Security)
- **Hosting:** Vercel
- **Payments:** Dodo Payments (subscriptions, webhooks, customer portal). Use the official SDK and current docs.
- **Notion fetching/rendering:** `notion-client` + `react-notion-x` (unofficial public-page endpoints, **no Notion API key needed**)
- **Email (one-time codes, transactional):** Resend
- **Rate limiting / short-lived data:** Upstash Redis
- **Validation:** zod
- **Cookies/tokens:** jose (signed, httpOnly cookies)
- **Password hashing:** bcrypt (or argon2)
- **Tests:** Vitest (unit), Playwright (smoke/e2e)
- **Monitoring:** Sentry (optional, DSN from env)

## 6. Architecture

```
Notion public page
      ↓  (server fetch via notion-client, validated host, timeout)
Cache layer (page_cache table, revalidate ~60s, last-good fallback)
      ↓
/e/[id] route: load embed → check plan, expiry, access mode → render with react-notion-x
      ↓
Customer site loads it in an <iframe> (auto-resize via postMessage script)
```

Other pieces: dashboard (create/manage embeds), settings, billing, analytics, leads, API, custom-domain middleware, marketing site + SEO pages.

## 7. Data model (Supabase / Postgres)

All tables have RLS enabled. Users can only access their own rows. Server-side privileged operations use the service-role client.

- **profiles**: `id` (uuid, = auth.users.id), `plan` (`free|pro|business`, default `free`), `dodo_customer_id`, `dodo_subscription_id`, `plan_status`, `current_period_end`, `created_at`. Auto-created by trigger on signup.
- **embeds**: `id` (short unique slug), `user_id`, `notion_url`, `page_id`, `title`, `icon` (emoji, optional), `settings` (jsonb: theme, width, height, frame style, hideTitle, hideCover, customCss), `access_mode` (`public|password|email|lead_capture`), `password_hash`, `allowed_emails` (text[]), `allowed_domains` (text[]), `created_at`, `updated_at`.
- **page_cache**: `page_id` (pk), `record_map` (jsonb), `fetched_at`.
- **embed_views**: `id`, `embed_id`, `viewed_at`, `referrer_host`, `country` (if available), `visitor_hash` (hash of IP + user agent + daily salt; no raw IPs, no cookies).
- **leads**: `id`, `embed_id`, `email`, `created_at`.
- **access_codes** (email one-time codes): `id`, `embed_id`, `email`, `code_hash`, `expires_at`, `attempts`.
- **api_keys**: `id`, `user_id`, `name`, `prefix`, `key_hash`, `last_used_at`, `created_at`. Show the full key once at creation; store only the hash.
- **custom_domains**: `id`, `embed_id`, `domain` (unique), `status`, `verified_at`.
- **webhook_events**: `id` (provider event id, pk), `processed_at`. Used for idempotent webhook handling.

Deliver the schema as SQL migration files in `/supabase/migrations`.

## 8. Routes and pages

**Marketing:** `/`, `/pricing`, `/blog` (+ MDX posts), SEO pages (`/notion-password-protect`, `/notion-course-platform`, `/embed-notion-in-webflow`, `/embed-notion-in-wordpress`), `/terms`, `/privacy`.

**Auth:** `/login`, `/signup`, auth callback.

**App (protected):** `/dashboard`, `/dashboard/embeds/[id]` (settings, analytics, leads tabs), `/dashboard/billing`, `/dashboard/api-keys`.

**Public embed:** `/e/[id]` (+ unlock screens for password, email code, lead capture).

**API:** internal routes for embeds CRUD, refresh, unlock; `/api/v1/*` (Business, API-key auth); `/api/webhooks/dodo`; `/api/health`.

## 9. Feature specifications

### 9.1 Notion rendering
- Input: a Notion public URL. Validate that the host is `notion.site` or `notion.so` (reject everything else). Extract the page ID.
- Fetch the recordMap with `notion-client`, with a timeout. Render with `react-notion-x` (import its CSS).
- Friendly, branded error screens: invalid URL, page not public, Notion unreachable.
- Start by supporting simple pages, then verify images, toggles, callouts, code blocks, databases, and embeds.

### 9.2 Caching and reliability
- Serve from cache and revalidate in the background about every 60 seconds.
- Upsert every successful fetch into `page_cache`.
- If Notion fails or times out, serve the last cached copy. Only show an error if there is no cache.
- Dashboard "Refresh now" button.

### 9.3 Embeds and dashboard
- Create: paste a Notion URL, validate, save, and show the iframe snippet with a copy button and a live preview.
- Edit title and icon, delete, and open settings.
- Optional auto-resize script (embed page sends its height with `postMessage`; a tiny script on the customer site resizes the iframe).
- Plan limit enforced on create (Free: 1 embed), with an upgrade prompt in the dashboard.

### 9.4 Customization
- Dark mode toggle, width/height, frame style (none, border, rounded shadow), hide title and cover.
- Custom CSS (Pro and above): **sanitize** (strip `@import`, `url()`, `expression`, `behavior`, anything that could close the style tag), and scope every rule to the embed container. Unit-test the sanitizer.

### 9.5 Access control
- **Password (Pro+):** bcrypt hash; branded unlock screen; on success set a signed httpOnly cookie scoped to that embed; rate-limit attempts.
- **Email restriction (Pro+):** owner lists allowed emails/domains; visitor enters email, gets a 6-digit code (Resend), enters it, gets a signed cookie. Never reveal whether an email is on the allowed list. Codes expire quickly and have an attempt limit.
- **Lead capture (Business):** visitor enters email before viewing; store in `leads`; owner views leads table and exports CSV.

### 9.6 Analytics
- Log each embed view asynchronously (must not slow the render).
- Pro: view counts over time. Business: plus unique visitors, top referrers, and country if available.
- Privacy-friendly: no cookies, no raw IPs.

### 9.7 Billing (Dodo Payments)
- Pricing page with monthly/yearly toggle.
- Checkout route creates a session for the chosen plan, tied to the logged-in user.
- Webhook route: verify signature; handle subscription active, updated, cancelled, and expired; update `profiles.plan` accordingly; idempotent via `webhook_events`.
- Billing page links to the customer portal.
- Downgrade behavior: if a user drops below the embed limit, do not delete embeds. Keep them but disable paid features and block creating new ones. Document the behavior.
- Develop in test mode; list every env var and every dashboard setting I must configure.

### 9.8 Business features
- **API:** hashed API keys, REST endpoints to create/list/update/delete embeds, rate-limited, simple docs page.
- **Bulk import:** CSV or pasted list of Notion URLs creates many embeds, with per-row validation and error reporting.
- **Custom domains:** user enters a domain; use the Vercel Domains API (token from env) to add it; show the exact DNS record to set; verify button and status; middleware maps the incoming hostname to the embed; handle failures and removal.

## 10. Security requirements (non-negotiable)

- SSRF protection: host allowlist on all outbound Notion fetches; timeouts; no following redirects to other hosts.
- RLS on every table; verify with tests that users cannot read or write others' rows.
- Service-role key server-only; never in client bundles.
- Headers: `/e/*` allows framing by anyone (`frame-ancestors *`); all other routes send `X-Frame-Options: DENY` / restrictive `frame-ancestors`. Add sensible CSP, `Referrer-Policy`, `X-Content-Type-Options`.
- zod validation on every API route and form.
- Rate limits on login, signup, unlock, code requests, and API.
- Cookies: signed, `httpOnly`, `secure`, `sameSite` set appropriately; scoped per embed.
- Webhook signature verification and idempotency.
- API keys and passwords stored hashed only.
- No secrets or personal data in logs.
- Sanitize custom CSS as specified.

## 11. Design system — Notion-inspired

**Goal:** Embr should look and feel like Notion: the marketing site, the dashboard, the unlock screens, the graphics, and the animations. A Notion user should feel at home immediately.

**Reference:** open notion.so (marketing site and the app) in the browser and inspect it with dev tools while building. The values below approximate Notion's UI; verify and adjust against the live site.

**IP guardrails (important):** recreate the *style*, not the assets. Do **not** copy Notion's logo, illustrations, icons, proprietary fonts, or source code. Draw original illustrations and use open-source fonts. Add "Embr is not affiliated with Notion Labs, Inc." in the footer. Mention Notion only descriptively (for example "embed Notion pages").

### 11.1 Color tokens

Define as CSS variables, mapped into the Tailwind theme. Support light and dark.

**Light**

| Token | Value |
|---|---|
| `--bg` | `#FFFFFF` |
| `--bg-sidebar` | `#F7F7F5` |
| `--bg-hover` | `rgba(55, 53, 47, 0.08)` |
| `--bg-subtle` | `#F1F1EF` |
| `--text` | `#37352F` |
| `--text-muted` | `rgba(55, 53, 47, 0.65)` |
| `--text-faint` | `rgba(55, 53, 47, 0.45)` |
| `--border` | `rgba(55, 53, 47, 0.16)` |
| `--border-soft` | `rgba(55, 53, 47, 0.09)` |
| `--accent` (Notion blue) | `#2383E2` |
| `--accent-hover` | `#0077D4` |
| `--accent-soft` | `rgba(35, 131, 226, 0.14)` |
| `--danger` | `#EB5757` |

**Dark**

| Token | Value |
|---|---|
| `--bg` | `#191919` |
| `--bg-sidebar` | `#202020` |
| `--bg-hover` | `rgba(255, 255, 255, 0.055)` |
| `--bg-subtle` | `#252525` |
| `--text` | `rgba(255, 255, 255, 0.81)` |
| `--text-muted` | `rgba(255, 255, 255, 0.46)` |
| `--border` | `rgba(255, 255, 255, 0.13)` |
| `--accent` | `#2383E2` |

**Notion block colors** (use for tags, plan badges, callouts, illustration fills, status pills):

| Name | Text | Background |
|---|---|---|
| Gray | `#787774` | `#F1F1EF` |
| Brown | `#9F6B53` | `#F4EEEE` |
| Orange | `#D9730D` | `#FBECDD` |
| Yellow | `#CB912F` | `#FBF3DB` |
| Green | `#448361` | `#EDF3EC` |
| Blue | `#337EA9` | `#E7F3F8` |
| Purple | `#9065B0` | `#F6F3F9` |
| Pink | `#C14C8A` | `#F9F3F7` |
| Red | `#D44C47` | `#FDEBEC` |

Provide dark-mode equivalents for the block colors (darker tinted backgrounds, lighter text).

**Usage:** mostly white and warm near-black. One accent (blue) for primary actions, links, focus rings, and selected states. Pastel block colors for tags and highlights. No gradients on UI chrome; keep it flat.

### 11.2 Typography

- **Sans (default):** Inter via `next/font`, falling back to `ui-sans-serif, -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif`.
- **Serif (optional for marketing headings and blog):** an open-source serif such as Lora or Source Serif 4, as a nod to Notion's serif page style.
- **Mono (code, API keys, embed snippets):** an open-source mono such as JetBrains Mono or IBM Plex Mono.
- Body 16px / 1.5 line height; app UI text 14px. Marketing H1 roughly 56 to 72px, bold, tight tracking (about -0.03em), 1.1 line height, centered. H2 32 to 40px. Generous whitespace.
- Emoji are part of the identity: use them as embed icons and in UI where Notion would (Apple/Noto emoji, no custom emoji set).

### 11.3 Shape, spacing, elevation

- Radii: 4px (small elements), 6px (buttons, inputs, menu items), 8 to 12px (cards, modals), pill (tags).
- Borders: 1px, using `--border-soft` or `--border`. Prefer borders over heavy shadows.
- Popover/menu shadow (Notion-like): `rgba(15,15,15,0.05) 0 0 0 1px, rgba(15,15,15,0.1) 0 3px 6px, rgba(15,15,15,0.2) 0 9px 24px`.
- Card hover: subtle background shift and slightly stronger border. No dramatic lifts.
- 4px spacing grid; content max width around 720 to 900px in app pages (page-like reading width), 1100 to 1200px for marketing sections.

### 11.4 Components (build once, reuse everywhere)

Button (primary blue, secondary bordered, ghost, danger), Input/Textarea, Select, Toggle switch, Checkbox, Tag/Pill (block colors), Badge, Card, Tabs, Table (database-like rows with hover), Dropdown/Popover menu, Modal, Tooltip, Toast, Skeleton loader, Empty state, **Command menu** (`Cmd/Ctrl+K`, and a `/`-style menu for quick actions in the dashboard), Sidebar item, Breadcrumb, Callout (emoji + tinted block), Code block with copy button, Property row (label left, value right, like Notion page properties).

### 11.5 App (dashboard) layout

Looks like the Notion app:
- **Left sidebar** (`--bg-sidebar`, collapsible): workspace/account switcher at top, nav items (Embeds, Analytics, Leads, API keys, Billing, Settings) with icons and hover backgrounds, plan badge and "Upgrade" at the bottom.
- **Main area:** page-like layout: large emoji icon, big editable title, and content below.
- **Embeds list** shown as a Notion-style database table: icon + title, status tag, access mode tag, views, last synced, row hover, row actions on hover. Optional gallery (card) view.
- **Embed settings** shown as page properties (label/value rows), with a live preview pane.
- Empty states use a small original line illustration and one clear action.

### 11.6 Marketing site style

- Centered hero with a large bold headline and a short subhead; primary blue button plus a secondary ghost button.
- **Interactive hero demo:** a mock "paste your link" input where a Notion URL is typed, then an embed preview animates in, then the iframe snippet appears. Loops gently, pauses on hover/focus.
- Feature sections as alternating text and product-screenshot blocks, with product UI shown in rounded, bordered frames with soft shadow.
- Logos/social proof strip when available. Use-case cards (courses, docs, help center, client portals) with pastel block-color backgrounds.
- Pricing: three clean cards, thin borders, the recommended plan marked with a blue badge; monthly/yearly toggle.
- FAQ as toggles (Notion-style disclosure triangles). Large, quiet footer with link columns and the non-affiliation line.
- Light and dark mode toggle.

### 11.7 Graphics and illustration

- Original **hand-drawn-feeling line illustrations**: black or near-black outlines, flat pastel fills from the block-color palette, simple characters/objects (pages, blocks, browsers, links, locks, charts). Consistent stroke width across the set.
- Deliver as inline SVG so they can be themed and animated (dark mode: lighter outlines).
- Use illustrations for hero accents, empty states, onboarding, and error pages (404, "page not public", "Notion unreachable").
- Product visuals use Notion-like UI mockups built in HTML/CSS, not stock images.

### 11.8 Motion

Notion's motion is quick, quiet, and purposeful. Rules:
- Durations: 120ms (hover, press), 200ms (menus, toggles), 300 to 400ms (page and section entrances). Never over 600ms except the hero demo.
- Easing: ease-out for entrances (for example `cubic-bezier(0.16, 1, 0.3, 1)`), ease-in-out for state changes.
- Patterns: hover background fades; menus/popovers fade + 4px slide + slight scale; modals fade in with a soft backdrop; toggles and accordions animate height smoothly; **scroll-reveal** (fade + 12px rise, staggered) for marketing sections; skeleton shimmer while loading; copy button swaps to a check mark; list items animate in/out on add/delete; page transitions are a short fade.
- Subtle micro-interactions: cursor-like highlight on the hero demo, a gentle "drag handle" appearing on row hover, a smooth progress bar when syncing.
- Honor `prefers-reduced-motion` by disabling transforms and keeping only opacity changes.
- Keep animations GPU-friendly (transform/opacity) and never block interaction.

### 11.9 The embed itself

The embedded Notion page stays **neutral and native-looking** (Notion's own rendering, light or dark) so it blends into customer sites. Only the free-plan badge, unlock screens, and error states carry Embr styling, and they follow the Notion-like system above. The "Powered by Embr" badge is small, quiet, and bordered.

### 11.10 Logo

Simple and professional; leaning toward a box-style mark, no mascot, no text-only logo. The owner will supply logo files (separate, transparent images). Until then use a neutral placeholder in `/public/brand/` referenced via `config/brand.ts`. Provide light and dark variants.

### 11.11 Accessibility and performance

Meet WCAG AA contrast (check text on pastel backgrounds and the blue button), visible focus rings (accent blue), keyboard navigation for menus and the command palette, semantic HTML, and fast loads (optimize fonts and images, lazy-load below-the-fold animation).

## 12. Environment variables

Keep `.env.example` complete. Never commit real values.

```
NEXT_PUBLIC_APP_URL=
NEXT_PUBLIC_APP_NAME=Embr
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
EMBED_COOKIE_SECRET=
DODO_PAYMENTS_API_KEY=
DODO_PAYMENTS_WEBHOOK_SECRET=
DODO_PRODUCT_PRO_MONTHLY=
DODO_PRODUCT_PRO_YEARLY=
DODO_PRODUCT_BUSINESS_MONTHLY=
DODO_PRODUCT_BUSINESS_YEARLY=
RESEND_API_KEY=
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=
VERCEL_API_TOKEN=
VERCEL_PROJECT_ID=
SENTRY_DSN=
```

Use the exact variable names Dodo's current SDK/docs expect where they differ.

## 13. Build milestones

Work in this order. Each milestone ends with: lint + typecheck + tests pass, manual verification steps given to me, `PROGRESS.md` updated, one git commit.

**M0 — Scaffold and design system.** Next.js + TS + Tailwind, folder structure, `config/brand.ts`, `.env.example`, `PROGRESS.md`, lint/format/test setup. Implement the design tokens (light and dark), fonts, base components from 11.4, and the motion primitives (fade, slide, scroll-reveal, with reduced-motion support). Add a `/design` page (dev only) showing every component. *Done when:* the app runs, light/dark toggle works, and the component gallery looks Notion-like.

**M1 — Notion renderer.** `lib/notion.ts` (validation, fetch, errors) + temporary `/preview?url=` page. *Done when:* simple and complex public pages render correctly; invalid/private/unreachable cases show friendly errors; URL validation is unit-tested.

**M2 — Database and auth (plan first).** Migrations, RLS, signup trigger, Supabase clients, email + Google auth, protected `/dashboard`. *Done when:* two test users cannot see each other's data.

**M3 — Embeds and dashboard (plan first).** App shell (Notion-style sidebar), embeds database-table view, create/edit/delete, snippet + copy, live preview, `/e/[id]`, auto-resize script, framing headers. *Done when:* the snippet works in a plain HTML page and resizes.

**M4 — Caching and reliability.** 60s revalidate, `page_cache` fallback, refresh button. *Done when:* Notion edits show within about a minute, and a broken fetch still serves the cached copy.

**M5 — Plans and limits.** `lib/plans.ts`, server-side enforcement, upgrade prompts, free badge. *Done when:* a free user cannot create a second embed, including by calling the API directly.

**M6 — Customization.** Theme, size, frame, hide title/cover, sanitized custom CSS, properties-style settings page with live preview. *Done when:* malicious CSS test strings are neutralized (tests included).

**M7 — Landing and pricing pages.** Hero with the interactive demo, features, use cases, pricing with monthly/yearly toggle, FAQ toggles, footer, original illustrations, scroll animations, light/dark. *Done when:* responsive, matches section 11, and passes Lighthouse performance and accessibility checks reasonably (target 90+).

**M8 — Billing (plan first).** Dodo checkout, webhook, customer portal, plan sync. *Done when:* a test purchase upgrades the plan, cancellation downgrades it, and replayed webhooks do nothing twice.

**M9 — Access control.** Password, email codes, rate limits, Notion-styled unlock screens. *Done when:* wrong/right password and allowed/disallowed emails all behave correctly, with no email enumeration.

**M10 — Analytics (plan first).** Async logging and dashboard charts (Notion-style, flat and minimal) with Pro/Business gating.

**M11 — Business features (plan first).** Lead capture + CSV, API keys + endpoints + docs, bulk import. *(Custom domains: build last; may ship after launch.)*

**M12 — SEO, content, legal.** SEO pages, sitemap, robots, OG image, MDX blog, `/terms`, `/privacy`, 404 and error pages with illustrations.

**M13 — Hardening.** Full security audit against section 10, tests (RLS, plans, URL validation, CSS sanitizer, webhook), Playwright smoke test (sign up → create embed → view embed), Sentry. Fix everything found.

**M14 — Production deploy.** Checklist in section 14, executed step by step with me.

**M15 — Launch kit.** Product Hunt copy, X thread, community post drafts, welcome email sequence, affiliate outline.

## 14. Production readiness checklist

- [ ] Production Supabase project with migrations applied; auth redirect URLs and Google provider configured
- [ ] All env vars set in Vercel (no secrets in the repo)
- [ ] Dodo live-mode products created; webhook URL set; live purchase tested and refunded
- [ ] Resend domain verified (SPF/DKIM); codes deliver to inbox
- [ ] Upstash configured; rate limits verified
- [ ] Custom domain DNS and SSL on the main app
- [ ] Security headers verified; `/e/*` frameable, everything else not
- [ ] RLS tests passing; service-role key not present in client bundles
- [ ] Embed tested inside Webflow, WordPress, and plain HTML; mobile and incognito checked
- [ ] Light and dark mode checked on every page; reduced-motion checked
- [ ] No copied Notion assets (logo, illustrations, icons, fonts); non-affiliation line in footer
- [ ] Error monitoring and uptime monitoring active; `/api/health` works
- [ ] Terms, privacy, and support contact in the footer
- [ ] No test data or debug logs left

## 15. Known risks

- **Unofficial Notion endpoints can change or rate-limit.** Mitigate with caching, the stale fallback, timeouts, and clear error handling. Keep Notion-specific code isolated in `lib/notion.ts` so it is easy to patch.
- **Notion may ship competing features.** Our edge is analytics, lead capture, reliability, and niche positioning.
- **Brand/IP risk from the Notion-inspired look.** Keep the style similar but all assets original, avoid any suggestion of affiliation, and use "Notion" only descriptively.
- **Crowded market.** Focus the marketing on specific use cases (courses, docs, client portals).

## 16. Open decisions (ask me, don't assume)

- Domain name
- Yearly discount percentage
- Whether the Pro embed cap stays at 25
- Whether custom domains ship at launch or right after
- Logo files (to be supplied)

## 17. First instruction to Claude Code

Read `CLAUDE.md` and this file. Summarize your understanding in a few lines, list any questions, then propose the plan for **M0 and M1** and wait for my approval before writing code.