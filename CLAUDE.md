# CLAUDE.md

Project: **Embr** — users paste a public Notion page URL and get an iframe embed served from our own domain, with password/email protection, custom styling, analytics, and more.

**Read `PROJECT_BRIEF.md` first. It is the source of truth for scope, plans, data model, security, design, and milestones.**

## Stack
Next.js (App Router, TypeScript, Tailwind) · Supabase (auth + Postgres + RLS) · Vercel · Dodo Payments · notion-client + react-notion-x · Resend (email codes) · Upstash Redis (rate limits) · zod · Motion (Framer Motion) for animation · Vitest + Playwright.

## How to work
1. Work **one milestone at a time**, in the order in `PROJECT_BRIEF.md` section 13.
2. At the start of each milestone: state your plan in a few lines, list files you will create or change, and **wait for my approval** before writing code on the larger milestones (M2, M3, M8, M10, M11).
3. After each milestone: run lint, typecheck, and tests; tell me exactly how to verify it manually; update `PROGRESS.md` (done / next / known issues); make one git commit with a clear message.
4. Do not start the next milestone until I confirm the current one works.
5. If something in the brief is unclear or a decision is missing, ask me. Do not guess on pricing, security, or billing behavior.

## Hard rules
- **Never hardcode secrets.** Read from environment variables only. Keep `.env.example` current. Never print or log secret values. If you need a key, tell me which env var to set and wait.
- Plan limits and feature flags come from **one place**: `lib/plans.ts` → `getPlanLimits(plan)`. Enforce them **server-side** (API routes and embed rendering), never only in the UI.
- Only `/e/*` routes may be framed by other sites. Every other route must block framing.
- Notion URLs must pass the host allowlist (`notion.site`, `notion.so`) before any fetch (SSRF protection).
- The Supabase service-role key is used in server code only and never reaches the browser.
- Validate all input with zod. Rate-limit auth, embed unlock, and API routes.
- For Dodo Payments, read the current official docs and use the official SDK. Do not rely on memory for API shapes.
- Prefer small, readable files and clear names. No unused dependencies. No dead code.

## Design (Notion-inspired)
The whole product should look and feel like Notion: clean white (and dark) surfaces, near-black warm text, thin borders, small radii, Notion's pastel block colors, one blue accent, and quick, subtle animations. Full tokens, typography, components, motion rules, and illustration style are in `PROJECT_BRIEF.md` section 11.

- Use Tailwind theme tokens / CSS variables, never scattered hex values.
- Support light and dark mode from the start.
- Recreate the *style*. **Do not copy Notion's logo, illustrations, icons, proprietary fonts, or code/assets.** Make our own original line-art illustrations and use open-source fonts.
- Never imply we are affiliated with Notion. Keep the "not affiliated with Notion Labs, Inc." line in the footer.
- Respect `prefers-reduced-motion`.

## Commands
Fill in after the scaffold exists (dev, build, lint, typecheck, test, e2e).