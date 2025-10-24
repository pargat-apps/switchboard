# Switchboard — Privacy-First Plug-and-Play Features for Any Website

**TL;DR:** Switchboard lets website owners **flip on useful features in <15 minutes** (via a script tag or light API) **without storing end-user PII** on our platform.  
Start with three modules: **Lead Capture Autopilot**, **Announcement/Changelog Bar**, **Zero-ETL Events & Funnels**.

---

## Why this exists

Shipping common website features usually takes **weeks** of engineering, introduces **privacy/compliance risk**, and forces teams to glue together **heavy tools**. Small teams and creators rarely have growth engineers to do this reliably.

**Switchboard** removes that friction:

- **Fast**: copy/paste a snippet → enable a feature → see value quickly  
- **Private**: identities remain with the customer’s IdP; **we never store end-user PII**  
- **Lightweight**: rules, targeting, and analytics without a CDP or cookie banner mess  
- **Affordable**: low cloud cost by design (aggregates, stateless paths)

---

## Who it’s for

- **SMBs, creators, e-commerce, indie SaaS** who need growth features quickly  
- **Agencies** that integrate features repeatedly across client sites  
- **Privacy-sensitive teams** that won’t centralize user data in third-party tools

---

## Problems we solve

- Lost leads due to broken form-to-CRM wiring  
- Users not seeing product updates → poor feature adoption  
- Heavy analytics stacks for simple funnels and UTM tracking  
- Long integration cycles, vendor sprawl, higher costs  
- Privacy & compliance risk from storing end-user data elsewhere

---

## What you get (MVP modules)

1) **Lead Capture Autopilot (Form Mapper)**  
   Auto-detects forms, maps fields to your CRM/webhook, adds spam guard, retries, and delivery logs.  
   _Data:_ pass-through only; we keep **mappings + aggregates** (no PII at rest).

2) **Announcement/Changelog Bar**  
   Target by path/UTM/geo, schedule messages, A/B test variants, measure CTR.  
   _Data:_ messages + rules + **aggregated impression/click counts**.

3) **Zero-ETL Events & Funnels (privacy-safe)**  
   Cookieless page/CTA events; simple funnels; UTM normalization; CSV/webhooks export.  
   _Data:_ **aggregates only** (no unique user identifiers).

---

## How it works (high level)

**Integration:**  
1) Owner signs up on Switchboard → registers domain  
2) Installs **snippet** (or Edge worker/Server middleware) and verifies domain  
3) Picks where widgets render via **Selector Picker** (DOM click-to-select)  
4) Enables modules and sets **Policies** (path/geo/A-B/rate)  
5) Traffic flows; SDK fetches **render config** → mounts widgets; aggregates get recorded

**Trust boundary:** end-user authentication stays with the **customer’s IdP** (Auth0/Keycloak/Cognito/etc.).  
**Optional**: a **stateless Auth Relay** validates tokens (verifies JWKS → drops claims).  
**We never store end-user PII**.

---

## Architecture (clean, multi-tenant)

**Client lane:** Website/App • Customer IdP • Customer API/Data  
**Platform lane:** SDK (script tag) / Edge Injector • Control Plane API • Policy Engine • (optional) Auth Relay  
**Stores (owner-only):** Config, Policies, **Secrets Vault (IdP metadata only)**, Webhooks, Usage Aggregates, Audit Logs  
**Observability:** Sentry errors, OTel metrics/logs (no PII), dashboards

Targets: p95 render-config <120ms • p95 token-validation (relay) <200ms • 99.9% availability

---

## Privacy by design

- **No end-user PII stored** on Switchboard  
- Identities stay with the customer’s IdP; optional stateless token validation  
- Logs scrubbed; signed webhooks; secrets encrypted; strict CORS/domain verification  
- Aggregation-only analytics; GPC/CMP aware; cookieless by default

---

## Repository layout (monorepo)

```
switchboard/
├─ 02-product/                # Charter, Scope/SRS, WBS, Gantt, Risks, etc.
├─ 03-design/                 # Wireframes, Figma, tokens, design system
├─ 04-engineering/
│  ├─ frontend/              # Next.js (marketing + admin)
│  ├─ backend/               # Node/Nest or Express+TS (Control Plane)
│  ├─ sdk/                   # Browser SDK (script tag)
│  ├─ infra/                 # IaC, Docker, env templates
│  ├─ ci-cd/                 # GitHub Actions, pipelines
│  └─ docs/                  # OpenAPI specs, ADRs, READMEs
├─ 05-qa/                    # Test plans/cases/reports, Playwright, k6
├─ 06-ops/                   # Runbooks, monitoring, security, deployment
└─ 07-execution/             # Sprints, burndowns, retros
```

Each folder contains its own `README.md` with purpose and links.

---

## Quick start (local dev)

> Prereqs: Node 18+, pnpm or npm, Docker (optional), a Postgres URL (Neon or local)

```bash
# Clone
git clone https://github.com/<your-org>/switchboard.git
cd switchboard

# 1) Backend (Control Plane API)
cd 04-engineering/backend
cp .env.example .env    # set DATABASE_URL, REDIS_URL, JWT, etc.
pnpm install            # or npm install
pnpm prisma migrate dev # create schema
pnpm dev                # http://localhost:4000

# 2) Frontend (Admin + Marketing)
cd ../frontend
cp .env.example .env    # set NEXT_PUBLIC_API_URL, etc.
pnpm install
pnpm dev                # http://localhost:3000

# 3) SDK (script)
cd ../sdk
pnpm install
pnpm build              # outputs dist/ps-sdk.js
```

---

## Minimal SDK usage (script tag)

```html
<script>
!function(w,d,s){var f=d.createElement('script');f.src=s;f.defer=1;d.head.appendChild(f);
w.ps = w.ps || {};
w.ps.init = { tenant:"t_123", site:"s_abc" };
w.addEventListener('DOMContentLoaded', async () => {
  const cfg = await fetch(`https://api.example.com/render-config?site=s_abc&path=${{}}encodeURIComponent(location.pathname)`).then(r=>r.json());
  // Mount the module (e.g., changelog bar) at the configured selector
  if (cfg.widgets) { /* ps.mount(cfg.widgets[0]) */ }
});
}(window,document,"https://cdn.example.com/ps-sdk.v1.js");
</script>
```

---

## REST API (glimpse)

- `POST /tenants` — create tenant  
- `POST /sites` — register site + domain verification  
- `GET /render-config?siteId&path&region` — returns widget instructions  
- `POST /policies` — path/geo/A-B/rate rules  
- `POST /webhooks/test` — test signed delivery  
- `POST /events/track` — aggregate event counts (page/cta)  
- `POST /auth/validate` (optional relay) — stateless JWT verification

Full OpenAPI spec lives in `04-engineering/docs/`.

---

## Roadmap (MVP → next)

- ✅ MVP: Lead Autopilot • Changelog Bar • Zero-ETL Funnels • Admin • SDK • Webhooks  
- Next: Edge injector fallback, more CRM/ESP integrations, dashboards & alerts, templates gallery

---

## Security & compliance

- RLS per tenant, PBKDF2/Argon2 for owner auth, rotated keys  
- HMAC-signed webhooks, rate limits, IP allowlists (optional)  
- Privacy posture: no end-user PII at rest; data-processing addendum available for pilots

---

## Contributing

We use a **Hybrid Waterfall + Agile** approach:
- Stable artifacts (Charter, SRS, Architecture) live under `02-product/` and ADRs in `00-admin/decisions/`.  
- Iteration happens in weekly **sprints** under `07-execution/sprints/`.  
- PRs require: tests, docs update, and telemetry hooks where relevant.

Run `pnpm lint && pnpm test` before opening a PR.

---

## Maintainers

- **Pargat (Founder / PM / Architect)** – product, architecture, approvals  
- **Harveen Kaur (Frontend & UX)** – Admin UI, SDK, accessibility/SEO  
- **Harmandeep Kaur (Backend & DevOps)** – Control Plane API, Policy Engine, CI/CD, observability

Contact: `hello@switchboard.dev`

---

## License

Choose a license for your goals (MIT/Apache-2.0/BSL). Put it in `/LICENSE`.

---

## FAQ

**Q: Do you store my users’ personal data?**  
A: **No.** End-user identities remain with your IdP. We only keep configuration, policies, signed webhook activity, usage **aggregates**, and audit logs.

**Q: Can I export my data?**  
A: Yes—config and aggregate analytics export as JSON/CSV. No lock-in.

**Q: How long to integrate?**  
A: <15 minutes for the first module via snippet; Edge/middleware optional.

---

> If you’re evaluating Switchboard, start at **`03-design/wireframes`** for the UI walkthrough, then open **`04-engineering/frontend`** and **`backend`** READMEs for local dev. For the “why,” read our **Project Charter** in `02-product/charter-and-vision/`.
