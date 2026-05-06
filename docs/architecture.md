# AI-Optimized Product Comparison Site Architecture (HE + EN)

## 1) Goals
- Fast public comparison pages with **p95 response time <= 1.2s**.
- Bilingual UX and content: **Hebrew (he) + English (en)**.
- Admin workflow where an operator enters a product category/name and triggers an **AI comparison agent**.
- Agent crawls/searches trusted sources (Reddit, RTINGS, review/comparison sites), selects **Top 3 products**, and recommends a best buy.
- Agent output is validated, normalized, and published into a new SEO page in the site structure.
- Behavioral analytics with **heatmaps** to detect suspicious behavior and evaluate risk around misleading claims.
- Lightweight, scalable, and observable architecture.

## 2) High-Level System

### Frontend (Edge-rendered)
- Framework: Next.js App Router (SSR + ISR).
- Hosting: Vercel / Cloudflare Pages with global CDN.
- Languages:
  - Route-based localization: `/he/...`, `/en/...`.
  - Server-side dictionary loading and per-locale metadata.
- Pages:
  - Welcome page (localized).
  - Comparison pages generated from CMS/DB records.
  - Admin page (authenticated).

### Backend API
- Runtime: Node.js (Fastify) or Python (FastAPI), behind API gateway.
- Responsibilities:
  - Admin auth + RBAC.
  - Agent orchestration endpoints.
  - Content lifecycle (draft -> fact-check -> publish).
  - Caching + invalidation.
  - Audit logs.

### Agent & Data Pipeline
- Queue: Redis + BullMQ (or SQS).
- Worker services:
  1. Query planner (expands product query to candidate search terms).
  2. Source fetchers (Reddit API, RTINGS pages, other trusted sources).
  3. Extractor (structured specs, pros/cons, ratings, citations).
  4. Ranker (Top 3 scoring model).
  5. Recommender (best product + confidence + rationale).
  6. Publisher (writes localized page records + JSON-LD).
- Guardrails:
  - Mandatory citations per claim.
  - Confidence threshold and manual review gate.
  - Hallucination and contradiction checks.

### Data Layer
- Primary DB: PostgreSQL.
- Cache: Redis.
- Search index: Meilisearch/OpenSearch (optional but recommended).
- Object storage: S3-compatible for snapshots/evidence blobs.

### Analytics / Heatmap
- Client event stream via lightweight SDK.
- Tools:
  - Self-hosted PostHog + session replay/heatmap, or Hotjar.
- Risk signals:
  - Rage clicks, dead clicks, excessive compare-page exits.
  - Claim interaction vs bounce correlations.

## 3) Core User Journeys

### A) Public user
1. User lands on localized welcome page.
2. User enters/chooses comparison topic.
3. User opens generated comparison page with Top 3 cards + recommendation.
4. Page includes transparent citations and “last verified at” timestamp.

### B) Admin + Agent
1. Admin opens `/admin` and submits product/category (e.g., “best 55-inch TV”).
2. API creates job in queue (`PENDING`).
3. Workers fetch, extract, rank, and draft content.
4. Fact-check layer validates claims and source links.
5. Admin reviews draft + confidence + risk flags.
6. Publish writes localized pages and revalidates CDN routes.

## 4) Suggested Information Architecture
- `/[locale]/` - welcome page.
- `/[locale]/compare/[slug]` - generated comparison page.
- `/[locale]/compare/[slug]/sources` - transparency page with citations.
- `/admin` - dashboard.
- `/admin/jobs` - job list/status.
- `/admin/jobs/[id]` - agent output review.

## 5) Agent Decision Model (Top 3 + Best Buy)

### Inputs
- Product intent query.
- Source quality priors.
- Freshness window (e.g., last 12 months preferred).

### Scoring (example)
- `total_score = 0.35*expert_score + 0.25*user_sentiment + 0.20*value_score + 0.10*reliability + 0.10*availability`

### Outputs
- Top 3 products with structured fields:
  - Name, price range, key specs, pros, cons, target user, source links.
- Best buy product with explanation.
- Confidence score and flagged uncertainties.

## 6) Content Integrity & Anti-Misinformation
- Every non-trivial claim stores `source_url`, `source_title`, `retrieved_at`.
- Staleness policy: mark draft stale if older than N days.
- Contradiction checker across sources.
- Risk labels:
  - Low / Medium / High claim risk.
- Admin cannot publish if:
  - citation coverage < threshold,
  - confidence < threshold,
  - unresolved contradiction flags.

## 7) Performance Blueprint (<=1.2s p95)
- Static pre-render compare pages + ISR.
- Edge caching headers and CDN route caching.
- API budget:
  - TTFB < 200ms from edge cache.
  - HTML payload < 120KB compressed.
  - JS budget < 170KB for public pages.
- Lazy-load heavy analytics scripts after interaction.
- Database:
  - Read replicas for public queries.
  - Prepared statements + indexed slug/locale.
- Observability:
  - p50/p95 latency dashboards by route.

## 8) Security & Compliance
- Auth: OIDC/SAML for admin.
- RBAC roles: `admin`, `editor`, `reviewer`.
- Rate limits on admin job creation.
- Signed webhooks between services.
- PII minimization and retention policy for session replay.

## 9) Minimal API Contracts
- `POST /api/admin/compare-jobs`
  - body: `{ query: string, localeTargets: ["he","en"] }`
- `GET /api/admin/compare-jobs/:id`
- `POST /api/admin/compare-jobs/:id/publish`
- `GET /api/public/compare/:locale/:slug`
- `GET /api/public/compare/:locale/:slug/sources`

## 10) Suggested Deployment Topology
- `frontend` service at edge/CDN.
- `api` service regional (autoscaled).
- `worker` service regional (queue consumers).
- `postgres`, `redis`, object storage managed services.
- `analytics` isolated project with event forwarding.

## 11) MVP Build Phases
1. **Phase 1 (2-3 weeks)**
   - Welcome page + locale framework.
   - Admin auth + create job.
   - Basic agent with 2-3 trusted sources.
2. **Phase 2 (2 weeks)**
   - Top 3 scoring + publication flow.
   - Comparison page templates + source transparency page.
3. **Phase 3 (1-2 weeks)**
   - Heatmaps + risk dashboard.
   - Performance hardening to p95 <= 1.2s.
4. **Phase 4**
   - A/B testing for copy and recommendation presentation.

## 12) Recommended Tech Stack (Lightweight)
- Frontend: Next.js + TypeScript + Tailwind.
- Backend: Fastify + Zod + Prisma.
- Worker: BullMQ + Redis.
- DB: PostgreSQL.
- Analytics/Heatmap: PostHog.
- Observability: OpenTelemetry + Grafana/Datadog.

