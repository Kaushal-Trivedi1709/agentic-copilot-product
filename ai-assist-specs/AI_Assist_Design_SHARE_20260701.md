# AI Assist — Platform Build Strategy

Status: APPROVED
Date: 2026-06-30
Owner: Kaushalendra Trivedi (NiCE Ltd)

---

## Problem Statement

NICE needs to ship AI Assist: an enterprise-grade AI copilot platform for contact-center
agents. The platform must support five use cases:

| UC | What it does | Core value |
|----|--------------|------------|
| UC-1 | Admin self-serves a capability into production (browse template, connect tools, attach knowledge, define rules, set autonomy, publish) | Speed — time to value |
| UC-2 | Fleet-wide governance: autonomy enforcement, guardrail caps, audit trail, RBAC, kill switch | Trust — enterprise readiness |
| UC-3 | PS generalises a bespoke Cognigy build into a reusable template in the library catalogue | Economics — the flywheel |
| UC-4 | One capability set runs across multiple brands/tenants; per-tenant brand tokens at render time; optional external renderer via UI-intent contract | Scale |
| UC-5 | Customer who outgrows templates gets Cognigy license unlock; existing config keeps running; bespoke builds coexist with templates | Retention — the moat |

The platform sits on two layers: CXone (orchestration + config surface) and Cognigy (the
runtime — customers never open it directly). Engineering must build the bridge between them
and all five use cases on top of it.

Reference: AI_Assist_Use_Case_Spec.docx

---

## Terms

- **Human agent**: the contact-center employee interacting with customers.
- **AI agent / capability**: the AI unit (built on Cognigy) that assists the human agent.
- **Copilot surface**: CXone — what the human agent and admin see.
- **Autonomy modes**: See (AI surfaces info, no action); Suggest→approve (AI drafts, human commits); Auto (AI acts, human reviews on exception).
- **UI intent**: what the AI agent emits — a card identifier + data payload, not pixels. The Copilot surface renders this via `render(intent: UIIntent): void`.
- **Guardrail**: a hard limit the AI agent cannot cross regardless of autonomy mode (e.g., spend cap, mandatory human approval threshold, action blocklist entry).
- **Effort labels**: S = 1-2 person-months; M = 3-4 person-months; L = 5-6 person-months; XL = 6+ person-months total engineering effort across the team.

---

## Constraints

- Cognigy is the runtime. This is not up for debate.
- MCP is the tool connectivity standard. As of mid-2026 it is the de facto industry standard.
- The NL → deterministic rules engine is a product to build. It does not exist today.
- The UI-intent contract has no dominant open standard. We build proprietary first, migrate later.
- We must have a live customer on WISMO by Month 6. Microsoft is shipping autonomous agents
  this quarter. A 3-month discovery delay is not acceptable at this competitive moment.

---

## Premises

1. **Cognigy as invisible runtime** — CXone owns config; Cognigy executes. The provisioning
   bridge (CXone → Cognigy API) is the architectural spine. Reliability, idempotency, and
   rollback are load-bearing for every UC.

2. **NL → deterministic compilation = enterprise trust** — Admins author rules in natural
   language at authoring time; the Rules Engine compiles to a deterministic execution graph;
   LLM judgment never runs at runtime. This is the 2026 industry consensus. We build the
   compiler plus a review surface showing the compiled result before publish.

3. **Four-tier model creates the flywheel** — PS generalising into templates requires a
   structural incentive. Working model: each generalised template counts against a PS quota
   target. Must be ratified by PS leadership before Month 1. If not ratified, UC-3 is
   aspirational in Phase 1.

4. **Provisioning idempotency via compensating transactions** — Second publish of the same
   config produces the same Cognigy runtime state. Partial failure recovery: a reconciliation
   job runs every 5 minutes and re-drives any CXone record whose Cognigy state diverges.
   CXone is the source of truth. Validated in Month 1 architecture.

5. **UI-intent contract as thin adapter** — Contract is a versioned JSON schema. Renderer
   implements one interface: `render(intent: UIIntent): void`. Migration to an open standard
   costs one interface swap, not a rewrite.

---

## Approaches Considered

### Approach A: Governance-First (Effort: M+M)
Build UC-2 before UC-1. No customer-visible value until Month 4+. Not chosen: governance is
correct but can be designed in parallel with the first template.

### Approach B: Template Wedge (CHOSEN, Effort: XL)
Build UC-1 + UC-2 end-to-end for one vertical simultaneously. UC-3/UC-4 foundations laid but
not fully shipped in Phase 1 — UC-3 template generalisation is the exception: it is included
at Month 6 to validate the flywheel with a real second customer.

### Approach C: PS Discovery Engine (Effort: S per customer, +3 months delay)
Use bespoke PS builds to answer Section 5 open questions first. Not chosen: competitive timing.

---

## Recommended Approach: B — Template Wedge

### Chosen vertical: WISMO ("Where Is My Order")
One MCP connector (order management system), one knowledge attachment (shipping policies),
one deterministic rule set (escalation thresholds). Simplest complete use case — right wedge.

### Latency budget
AI agent action round-trip must complete in under 500ms P99 end-to-end (human agent receives
card in under 500ms from the triggering event). Any enforcement mechanism — including the
Cognigy runtime-level guardrail hook and the provisioning reconciliation job — must not
contribute to this budget. If the sidecar proxy fallback (see Open Question 4) exceeds this
budget in load testing, governance scope must be constrained to eliminate the sidecar.

### Team composition (required — Month 6 is infeasible without this headcount)
- 3–4 backend engineers: provisioning service, Rules Engine, governance enforcement, audit
- 2–3 frontend engineers: CXone admin surface (Month 3–4); experience composer + theming
  (Month 5–6). Note: 1–2 frontend engineers cannot complete three simultaneous frontend
  deliverables. Minimum 2 frontend engineers with a Phase 1 scope decision (see below).
- 1 platform/DevOps engineer: Cognigy environment promotion, CI/CD, compatibility test suite
- 1 product owner embedded from Month 1

**Phase 1 frontend scope decision** (must be made by Month 2, owner: product owner):
Option A — Hardcode the WISMO card set for Phase 1 and defer the experience composer (full
card selection UI) to Phase 2. Customer A goes live with a fixed card set; composer is Phase 2.
Option B — Hire a third frontend engineer and build the full composer in Phase 1.
This decision gates Month 5–6 frontend planning.

### Build sequence (Month 1–6)

**Month 1–2: Platform Spine**

- **Provisioning service** (CXone → Cognigy API bridge):
  - Authentication: OAuth 2.0 client credentials to Cognigy API. Credentials stored in
    NICE secrets manager. Rotation policy: 90-day automatic rotation, alerting on failure.
  - Idempotent publish via compensating transactions (see Premise 4)
  - Rollback trigger: any failed publish leaves Cognigy and CXone in the pre-publish state;
    no partial state is visible to the admin or the agent workspace
  - Cognigy API version-pinning: bridge locks to a validated API version snapshot.
    Compatibility test suite runs nightly; breaks are flagged before they reach production.
    *Contractual requirement*: written commitment from Cognigy on API version support lifetime
    (minimum 12 months from integration start) must be obtained before provisioning service
    ships to production. Owner: [commercial/partnerships lead — to be named by Month 1, Week 2].
  - *Decision gate, Week 3*: Rules Engine compiler approach confirmed

- **MCP connector framework**: connector selection + auth flow for WISMO order system.
  Runtime failure modes for MCP tool calls:
  - Timeout (>5s): surface "Order system unavailable" card to human agent, trigger handoff flag
  - Auth expiry: attempt one re-auth; if fails, surface error card, log event, alert admin
  - Downstream error (5xx): surface degraded-mode card; AI agent falls back to See-only mode
  - These failure modes apply to all MCP connectors; the framework handles them generically.

- **Template packaging schema**: parameterised tool configs, knowledge source references,
  rule sets, autonomy defaults, guardrail caps. Versioning: templates carry a schema version;
  deployed instances pin to their configured version; upgrades are opt-in with diff preview;
  rollback to previous pinned version is supported (admin selects version from history; publish
  re-provisions from that version's config).

- **Cognigy environment promotion**: dev → staging → prod pipeline. Owner: [DevOps engineer —
  named by Month 1, Week 1]. Deliverable: promotion pipeline defined and validated by end of
  Month 2.

- **RBAC minimum role set** (to be validated in Month 1 architecture session; owner: product owner):
  - Fleet Admin: read/write all capabilities, all governance settings, all audit logs; can
    issue kill switch
  - Capability Admin: read/write own capabilities, own behaviour rules, own autonomy settings;
    cannot change fleet-wide policy
  - Auditor: read-only access to audit logs and fleet view; no write access
  - Agent (human): read-only access to the agent workspace surface; no admin access

**Month 1–2 (parallel): Governance baseline design**
Governance implementation is Month 5–6. Design begins Month 1 in parallel, so that the
Month 5–6 implementation is against a validated design, not a fresh one.
*Important*: Customer A's Month 6 go-live is gated on governance being live and passing
compliance review. A test-only publish in Month 4 (the Month 4 success criterion) runs
without governance enforcement, but no production traffic reaches Customer A until
governance control plane is live.

**Month 3–4: Rules Engine + Admin Surface**
*(Scope is provisional pending Week 3 compiler decision. Work not dependent on the compiler —
admin surface layout, MCP auth UI, knowledge attachment UI — proceeds independently.)*

- **NL → deterministic compiler**: admin writes a rule in natural language; compiler produces
  a deterministic execution graph; admin reviews the compiled result (diff-style view) before
  publish. Compiled graph is the runtime logic. LLM is used at compile time only.
  *Three compiler options for Week 3 decision*:
  (a) LLM compile step at authoring time → deterministic DSL output. Risk: verifying LLM
      output is provably deterministic is an unsolved production problem as of mid-2026.
  (b) Structured rule builder with NL-assisted authoring (NL → structured form pre-fill).
      Lower risk; less expressive; well-understood implementation path.
  (c) Constraint-satisfaction system with NL description layer.
  *Fallback*: If no option is production-ready by Month 4, Phase 1 ships with Option (b)
  structured rule builder only (no NL compile step). NL compiler becomes Phase 2.
  Customer A can go live with structured rules; the NL layer is an enhancement.

- **Autonomy + guardrail configuration**: per-capability autonomy mode (see Terms) and
  hard guardrail caps. Enforcement is at Cognigy runtime level (see Open Question 4).

- **Admin config surface in CXone**: multi-step wizard — template browse → MCP connector
  auth → knowledge attachment → behaviour rules (NL or structured input + compiled review)
  → autonomy + guardrails → experience card selection (or hardcoded Phase 1 set) → publish
  → live status.

- **UI-intent schema re-validation checkpoint** (Month 4): schema owner reviews whether
  the Month 1 schema definition holds against the experience composer requirements.
  Any schema changes before Month 5 are backwards-compatible additions. Breaking changes
  require sign-off from product owner.
  *UI-intent schema owner*: [name to be assigned by Month 1, Week 2].

**Month 5–6: Experience + Governance**

- **Experience composer** (or hardcoded WISMO card set — see Phase 1 frontend scope decision
  above): approved card library; AI agent emits UI intent (card identifier + data payload);
  Copilot surface renders via `render(intent): void`.

- **Brand-token / theming system**: per-tenant brand tokens applied at render time.
  This is UC-4 infrastructure only. Full UC-4 (external renderer SDK) is Phase 2.

- **Governance control plane (UC-2)**:
  - Fleet view: every capability with current autonomy mode, guardrail caps, behaviour
  - Autonomy enforcement at Cognigy runtime level. Concrete mechanism: pre-action hook in
    Cognigy's flow engine that checks the capability's current autonomy mode and guardrail
    caps before executing any tool call or external action. If Cognigy does not support
    pre-action hooks at the required granularity, fallback is a sidecar proxy between CXone
    and Cognigy — subject to the 500ms latency budget constraint above.
  - Guardrail enforcement: hard caps checked in the pre-action hook before every agent action
  - Audit logging: every agent action logged with inputs, outputs, decision path, autonomy mode
    at time of action. Tamper-evidence: append-only log with SHA-256 hash chain (each entry
    includes the hash of the previous entry). Log storage: write-once store (WORM or equivalent).
  - RBAC (see Month 1 role set above) + pause/kill switch
  - *Compliance review*: agreed criteria and reviewer availability confirmed by Month 4.
    Owner: [compliance team lead — named by Month 1, Week 2].

**Month 6: Go-live**

Month 6 gates, priority-ordered:
1. **Non-negotiable minimum**: Customer A live on WISMO template with governance enforcement
   active. This is the external commitment and the competitive rationale.
2. **High priority**: Governance control plane passes internal compliance review.
3. **High priority**: WISMO template generalised (UC-3) and published to catalogue;
   Customer B adopts via UC-1 self-serve in under 5 business days.
4. **Can slip to Phase 2 without ceding competitive position**: Full experience composer
   (if hardcoded card set was chosen in Phase 1 frontend scope decision); full UC-4
   external renderer SDK; UC-5 Cognigy license entitlement unlock.

If any gate 2–3 is not ready at Month 6, Customer A go-live proceeds with governance in
place but the compliance review and second customer adoption slip to Month 7.
Gate 1 does not slip.

---

## Open Questions (to be resolved in Month 1 architecture session, Weeks 1–4)

1. **Provisioning partial failure** (Week 1): Validate compensating-transaction rollback
   against Cognigy API. Confirm reconciliation job error surface. Owner: [lead backend engineer].

2. **Rules Engine compiler approach** (Week 3 decision gate): Choose from three options above.
   Fallback position defined (structured rule builder). Owner: [name to be assigned].

3. **Template packaging schema versioning** (Week 2): Ratify opt-in upgrade model with diff
   preview and rollback to pinned version. Owner: [lead backend engineer].

4. **Governance enforcement location** (Week 2): Validate Cognigy pre-action hook support.
   If unsupported, design sidecar proxy and validate against 500ms latency budget.
   Owner: [lead backend engineer].

5. **UI-intent schema initial version** (Week 4): Define JSON schema for WISMO card set.
   Owner: [UI-intent schema owner — named by Week 2]. Re-validation checkpoint: Month 4.

---

## Testing Approach

- **Provisioning service**: 100 idempotent publish tests against Cognigy staging. Failure
  injection at each step; rollback verified by querying both CXone and Cognigy state. Run
  in CI on every merge.
- **Rules Engine**: compiled graph verified against known-good expected outputs for a
  library of representative NL rules. Determinism verified by running the same input 100
  times and comparing outputs.
- **MCP connector failure modes**: simulated timeout, auth expiry, and 5xx responses;
  correct fallback cards and handoff flags verified for each.
- **Governance/audit**: tamper-evidence verified by out-of-order write attempts and
  hash-chain validation. Audit completeness verified against a scripted agent session.
- **Governance enforcement**: attempt to bypass autonomy enforcement at surface level;
  verify runtime still blocks. Attempt to bypass guardrail cap; verify runtime blocks.
- **Autonomy mode E2E**: one end-to-end test suite per autonomy mode (See-only,
  Suggest→approve, Auto) covering happy path, MCP failure degradation, double-action
  idempotency, agent navigates away mid-action, and audit log verification. All three
  suites must pass as a Month 6 Gate 1 acceptance criterion.
- **Latency**: P99 end-to-end round-trip measured in staging under load before any
  enforcement mechanism goes to production. Gate: must be under 500ms.
- **Test environment**: dedicated Cognigy staging tenant provisioned Month 1, Week 1.

---

## Success Criteria

- **Month 2**: Provisioning service publishes to Cognigy staging via CXone. Idempotency
  verified (100 repeated publishes). Partial failure → clean rollback confirmed.
- **Month 4**: Admin completes WISMO capability end-to-end — template selection to live
  status — in under 30 minutes from first login. (Test publish only; governance not yet live.)
- **Month 6, Gate 1**: Customer A live on WISMO with governance enforcement active.
- **Month 6, Gate 2**: Governance control plane passes internal compliance review. Audit
  trail is tamper-evident (hash chain verified). Autonomy enforcement is unbypassable
  (surface-level bypass attempt fails).
- **Month 6, Gate 3**: Customer B adopts WISMO template via UC-1 self-serve in under 5
  business days (template selection to capability live in human agent workspace).
  PS time-to-deploy for WISMO under 1 week (customer contract signature to capability live).
  Current PS baseline for comparable bespoke builds: [to be filled by PS leadership before
  Month 1 — needed to validate whether "under 1 week" is ambitious or trivial].

---

## Distribution Plan

Internal platform — distribution via CXone tenant provisioning. No external package manager.
CI/CD: NICE internal pipeline. Cognigy environment promotion (dev → staging → prod):
Month 1 deliverable, owned by [DevOps engineer].
Phase 2 (UC-4 external renderer SDK) will require an SDK distribution channel — deferred.

---

## Dependencies

- **Cognigy API access + written stability commitment** (12-month support): required before
  provisioning service ships to production. Owner: [commercial/partnerships lead]. If
  commitment cannot be obtained, abstraction layer required (see Cognigy API risk above).
- **Cognigy staging tenant**: provisioned by Month 1, Week 1.
- **Pilot Customer A (WISMO)**: named account + contact + verbal deployment commitment.
  Required by end of Week 2, Month 1. Without this, Month 6 Gate 1 has no forcing function.
- **Pilot Customer B (WISMO)**: identified by Month 4 for self-serve adoption validation.
- **PS incentive model**: ratified by PS leadership before Month 1. If not ratified, UC-3
  is aspirational in Phase 1.
- **Compliance review criteria + reviewer availability**: confirmed by Month 4.
  Owner: [compliance team lead].
- **CXone design system**: component library available for experience composer. Confirmed
  or fallback chosen by Month 2.
- **Team headcount**: 3–4 backend, 2–3 frontend (with Phase 1 scope decision made by Month
  2), 1 DevOps, 1 product owner — confirmed and onboarded by Month 1, Week 1.
- **PS time-to-deploy baseline**: current average for a comparable bespoke build.
  Required before Month 1 to validate the Month 6 Gate 3 success criterion.

**Blocked fields** — this document is not approved for Month 1 execution until the
following are filled in:
- [ ] Commercial/partnerships lead (Cognigy API commitment)
- [ ] DevOps engineer named (Cognigy environment promotion)
- [ ] UI-intent schema owner named
- [ ] Rules Engine compiler owner named
- [ ] Compliance team lead named
- [ ] PS time-to-deploy baseline filled
- [ ] Frontend scope decision (hardcoded card set vs full composer in Phase 1)
