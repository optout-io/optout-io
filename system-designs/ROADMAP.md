# OptOut.io Roadmap

Strategic initiatives in rough priority order. Each becomes a design doc or ADR when work begins.

---

## 1. Configuration-driven workflow definitions

**What:** Replace Go-coded Temporal workflow logic with a DSL or config-driven approach so adding a new data broker's opt-out flow doesn't require a Go code change.

**Why:** The webform already uses YAML configs for form steps (`databrokers/*.yaml`). The Temporal workflow layer — which today is a Go file per broker — should be expressible the same way. This decouples broker onboarding from engineering deploys.

**Explore:**
- Temporal's own workflow DSL (experimental — verify current status/naming)
- Rolling our own YAML-driven interpreter that maps to Temporal activities (we already have the webform step model)
- Whether broker-specific child workflows can be replaced by a single generic workflow parameterised by a YAML definition

**Starting point:** `data-erasure-wf/internal/dataerasure/databrokers/` — one Go file per broker today.

---

## 2. Hosting + CI/CD

**What:** Deploy all services to a cloud provider and wire up automated build/test/deploy pipelines.

**Services to host:**
| Service | Notes |
|---|---|
| `data-erasure-wf` (worker + service) | Needs Temporal, NATS, PostgreSQL |
| `lake` | Needs NATS, PostgreSQL |
| `abscond` (adminapp) | Stateless Go binary, easiest to deploy |
| `webform-playwright` | Needs Playwright browsers — heavier runtime |
| Temporal cluster | Managed (Temporal Cloud) or self-hosted |
| NATS | Managed (Synadia Cloud) or self-hosted |

**CI/CD:**
- GitHub Actions already exists for `contracts` and `go-common`
- Add workflows for: lint, test, Docker build, push to registry, deploy on merge to main
- Secrets management (DB credentials, Temporal certs, NATS credentials)

**Candidate platforms:** Fly.io (simple, good Go support), Railway, GCP Cloud Run, or a single VPS for the side-project phase.

---

## 3. Identity, authentication and authorisation

**What:** Secure the system end-to-end — who can call what.

**Two distinct concerns:**

**Admin access (abscond):**
- Currently no auth at all — needs at least HTTP basic auth or OAuth2 (GitHub/Google SSO) before hosting
- Authorisation is simple: admin or not

**End-user identity (future userapp):**
- Users who sign up and receive opt-out services
- Needs full identity: sign up, email verification, password reset, sessions/JWTs
- Could be a managed provider (Auth0, Clerk, Supabase Auth) or self-hosted (Kratos)
- The `UserURN` pattern is already in contracts — foundation is there

**Starting point:** Admin auth first (it's blocking hosting). User identity design doc before userapp work begins.

---

## 4. Webform — automated opt-out delivery (core business)

**What:** The `webform-playwright` service actually submits opt-out request forms to data brokers on behalf of users. This is the core value proposition.

**Current state:**
- Worker is registered and receives Temporal activity tasks
- YAML configs exist for broker form flows
- Playwright is wired up
- The `WebformPlaywright` activity is called from broker child workflows (`acxiom`, `deepsync`)
- **NOT actually executing end-to-end** — local dev doesn't have real broker sites accessible, and configs may be incomplete/stale

**What needs to happen:**
1. Audit existing YAML configs — are the form selectors current?
2. Test each broker's form submission in a real browser (or staging environment)
3. Handle the email confirmation loop properly (signal is already wired, email receiving is not)
4. Error handling + retry strategy for transient failures (CAPTCHA, rate limiting, site changes)
5. Screenshots / evidence collection for audit trail
6. Monitoring: which brokers are failing, success rates per broker

**This is the product** — everything else (workflows, lake, admin) exists to support this.

---

## Notes

- Items 1 and 4 are tightly related — a DSL-driven workflow would make broker onboarding much faster
- Item 2 (hosting) is a prerequisite for showing the product to real users
- Item 3 (auth) must be done before hosting is public-facing
- Recommended order for a demo-ready state: **2 → 3 → 4 → 1**
