# Implementation Tasks — AI Assist Eng Review

Synthesized from /plan-eng-review on 2026-07-01
Design doc: AI_Assist_Design_20260630.md (Status: APPROVED)
Branch: unknown | Repo: ASK AI (NiCE Ltd)

Each task derives from a specific finding in the engineering review.
P1 = blocks Month 6 go-live; P2 = same-phase work; P3 = follow-up.

---

## Tasks

- [ ] **T-E1 (P1, human: ~1h / CC: ~5min)** — Design Doc — Add RBAC enforcement trust assumption
  - Surfaced by: Architecture Review (D2)
  - Action: Add a "RBAC Enforcement" note to the design doc under the RBAC role set (Month 1-2 section). State explicitly: Cognigy API credentials are stored in NICE secrets manager only and never exposed to CXone users; RBAC is enforced at the CXone BFF/gateway layer; this is a convention-based trust assumption in Phase 1. Phase 2 adds API-layer enforcement (Cognigy webhook signature validation).
  - Verify: Design doc contains explicit statement of trust assumption before Month 1 architecture session.

- [ ] **T-E2 (P1, human: ~2h / CC: ~10min)** — Design Doc — Add audit log write failure behavior
  - Surfaced by: Architecture Review (D3)
  - Action: Add to the audit logging section: fail-closed for Auto mode (audit write failure blocks the action; agent sees "action paused" card); fail-open for Suggest→approve mode (audit write queued for async retry; action proceeds). Add both behaviors to the Testing Approach section as required tests.
  - Verify: Design doc specifies fail-closed/fail-open behavior per autonomy mode. Testing Approach includes two audit log failure tests.

- [ ] **T-E3 (P2, human: ~2h / CC: ~10min)** — Design Doc — Extend Open Question 4 (governance enforcement)
  - Surfaced by: Architecture Review (D4, D9)
  - Action: Extend Open Question 4 to explicitly cover: (a) governance state cache location and TTL, (b) hook timeout budget (e.g., 50ms), (c) hook timeout behavior (fail-closed for Auto, fail-open for Suggest), (d) kill switch cache invalidation mechanism and propagation SLA. Month 1 architecture session resolves all four.
  - Verify: Open Question 4 lists all four sub-questions. Architecture session agenda covers them.

- [ ] **T-E4 (P2, human: ~1 sprint / CC: ~30min)** — CI Tooling — Add UI-intent schema compatibility check
  - Surfaced by: Architecture Review (D5)
  - Action: Add a schema compatibility validator to CI: detect field removals, type changes (breaking), reject PRs with breaking changes automatically. Deployed templates pin to their schema version; upgrades are opt-in with diff preview. Document the compatibility rules in the design doc's UI-intent schema section.
  - Verify: CI blocks a PR that removes a required field from the UIIntent schema. Deployed template continues to use pinned schema version after a schema update ships.

- [ ] **T-E5 (P1, human: ~2h / CC: ~10min)** — Design Doc — Add template upgrade drain-window behavior
  - Surfaced by: Code Quality Review (D6)
  - Action: Add to the template packaging / versioning section: upgrades apply on next conversation start (not immediately). Provisioning service marks the new version "pending" and flips to "active" after a drain window (configurable, default 5 minutes). Active conversations complete on the prior pinned version.
  - Verify: Design doc specifies drain-window behavior. Provisioning service spec includes pending→active state transition.

- [ ] **T-E6 (P1, human: ~2h / CC: ~10min)** — Design Doc — Add compile-once / re-review requirement to Rules Engine spec
  - Surfaced by: Code Quality Review (D7)
  - Action: Add to the Rules Engine section: compiled graph is stored as the canonical artifact at publish time. Re-running the NL rule through the compiler produces a new draft; the draft must go through admin review + publish before replacing the deployed graph. Recompilation does not auto-update the deployed graph. Add: "The graph that runs in production is exactly the graph the admin reviewed."
  - Verify: Rules Engine spec states compile-once model. Testing Approach includes re-compile → re-review flow test.

- [ ] **T-E7 (P2, human: ~1h / CC: ~5min)** — Design Doc — Add minimum rule count validation to publish schema
  - Surfaced by: Code Quality Review (note)
  - Action: Add to the template packaging schema spec: minimum 1 rule required before publish is allowed. Validation fires at the wizard's publish step (before provisioning service is called). Error message: "At least one behaviour rule is required before publishing."
  - Verify: Design doc schema spec lists minimum-1-rule constraint. Wizard blocks publish with zero rules.

- [ ] **T-E8 (P1, human: ~1 sprint / CC: ~30min)** — Testing Approach — Add E2E test suite for all 3 autonomy modes
  - Surfaced by: Test Review (D8)
  - Action: Add to Testing Approach section: one E2E test suite per autonomy mode (See-only, Suggest→approve, Auto), using a scripted Cognigy session against the staging tenant. Each suite covers: happy path, MCP failure degradation, double-action edge case (idempotency), agent navigates away mid-action, and audit log verification for the action. Gated: all three suites must pass before Customer A go-live (Month 6 Gate 1 acceptance criterion).
  - Verify: Testing Approach section includes E2E test requirements for all 3 modes. Month 6 Gate 1 acceptance criteria reference the E2E suite pass.

- [ ] **T-E9 (P2, human: ~1h / CC: ~5min)** — Design Doc — Add kill switch propagation to Month 1 agenda
  - Surfaced by: Test Review (D9)
  - Action: Add to Month 1 architecture session agenda (Open Question 4): define kill switch propagation SLA as a customer-facing commitment. The cache invalidation mechanism (push-invalidation vs TTL expiry) determines what SLA is achievable. Month 5-6 staging test: activate kill switch, measure propagation time, gate = must meet stated SLA.
  - Verify: Open Question 4 explicitly includes kill switch SLA. Month 5-6 Testing Approach includes kill switch propagation test.

- [ ] **T-E10 (P2, human: ~2h / CC: ~10min)** — Testing Approach — Add async compilation UX to admin wizard spec
  - Surfaced by: Performance Review (D11)
  - Action: Add to the admin config surface section: NL rule compilation is async. Admin sees "Compiling..." state after submitting NL input. Page polls or uses websocket for completion notification. Compilation timeout: 60 seconds. Timeout error: "Compilation failed — simplify your rule or try again." Add: LLM API unavailability at compile time: same timeout path, same error message.
  - Verify: Admin surface spec includes async compilation UX. Error message specified. Timeout value stated.

- [ ] **T-E11 (P2, human: ~2h / CC: ~10min)** — Design Doc / MCP Spec — Add malformed response handler to MCP connector
  - Surfaced by: TODOS / Failure Modes Registry
  - Action: Add to the MCP connector framework failure modes: malformed tool response (HTTP 200, non-JSON body or schema-invalid JSON) → surface error card to human agent, log the raw response (truncated) for debugging, trigger same fallback as 5xx (See-only mode). Add test: inject malformed response in staging, verify error card appears and event is logged.
  - Verify: MCP failure mode table includes malformed-response row. Test case exists in Testing Approach.

- [ ] **T-E12 (P2, human: ~2h / CC: ~10min)** — Design Doc / UI Intent Spec — Define UI intent rendering fallbacks
  - Surfaced by: TODOS / Failure Modes Registry
  - Action: Add to the UI-intent contract section: (a) brand token missing for tenant → fall back to NICE default theme, log a warning event; (b) schema version mismatch between Cognigy runtime and CXone renderer → renderer ignores unknown fields (additive schema compatibility), logs version mismatch event for ops monitoring. Add tests for both in the Month 5-6 E2E test suite.
  - Verify: UI-intent spec includes fallback behavior for both cases. E2E tests inject both conditions.

- [ ] **T-E13 (P2, human: ~2h / CC: ~10min)** — Design Doc / Provisioning Spec — Add reconciliation "stuck diverged" state
  - Surfaced by: TODOS / Failure Modes Registry
  - Action: Add to the provisioning service / reconciliation job spec: a capability record that has been diverged (CXone ≠ Cognigy state) for >15 minutes (3 consecutive reconciliation cycles) enters "stuck" state. Behavior: fleet view shows a warning badge on the stuck capability; admin alert fires (notification channel TBD in Month 1). Test: inject persistent Cognigy unavailability in staging and verify warning appears within 15 minutes.
  - Verify: Provisioning spec includes "stuck diverged" state definition. Fleet view spec includes the warning badge. Alert mechanism defined in Month 1.

---

## Traceability

| Task | Section | Decision |
|------|---------|----------|
| T-E1 | Architecture | D2 (RBAC trust assumption) |
| T-E2 | Architecture | D3 (Audit log fail-closed/open) |
| T-E3 | Architecture | D4 + D9 (Governance hook + kill switch) |
| T-E4 | Architecture | D5 (Schema compatibility CI) |
| T-E5 | Code Quality | D6 (Template upgrade drain window) |
| T-E6 | Code Quality | D7 (Compile-once model) |
| T-E7 | Code Quality | Section 2 note (empty rule set) |
| T-E8 | Test Review | D8 (Autonomy mode E2E tests) |
| T-E9 | Test Review | D9 (Kill switch propagation) |
| T-E10 | Performance | D11 (Async compilation UX) |
| T-E11 | Failure Modes | TODO 1 (MCP malformed response) |
| T-E12 | Failure Modes | TODO 2 (UI intent rendering fallbacks) |
| T-E13 | Failure Modes | TODO 3 (Reconciliation stuck state) |
