# 57 · Case Study: A Feature Flag and Config Delivery Service

> **Level:** ★★★ advanced
>
> **You will practise:** designing a **feature flag / dynamic configuration service** used by every service in the company: **local evaluation** with SDKs, **streaming updates** to thousands of clients, **targeting rules** and consistent bucketing, **safe rollout of config itself** (validation, staged propagation, rollback), **static stability** (keep working when the service is down), **audit**, **flag lifecycle**, and the failure mode where *config breaks everything at once*.
>
> **You need:** topics 06, 10, 14, 23–24, 33–34, 37, 42.
>
> **Part 9 · Case studies** · Previous: [56 · Job scheduler](56-case-job-scheduler.md) · Next: [58 · Troubleshooting at scale](58-troubleshooting-at-scale.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Design a feature flag and runtime configuration service for 300 services."

| Question | Assumed answer |
|---|---|
| Use cases? | Release toggles (gradual rollout by %), kill switches, experiments (A/B), operational knobs (rate limits, timeouts), per-tenant entitlements |
| Clients? | 300 services × ~20,000 instances total; also web and mobile clients |
| Evaluation volume? | Services evaluate flags on every request: ~2 M evaluations/s total |
| Latency? | Evaluation < 1 µs–1 ms (must not add network calls per request) |
| Propagation? | Changes visible everywhere within 10 s; kill switches within 5 s |
| Availability? | Flags must keep working if the service is down (critical: flags guard production paths) |
| Governance? | Audit log, approvals for prod changes to critical flags, scheduled changes |
| Regions? | 2 regions + mobile clients worldwide |

---

## 2. Estimation

```
Evaluations: 2 M/s → impossible (and unnecessary) as remote calls → SDK evaluates LOCALLY from an in-memory ruleset
Ruleset size: 5,000 flags × ~2 KB rules = 10 MB total; per service only its project's flags (~200 flags ≈ 400 KB)
Distribution: 20,000 instances hold a streaming connection (SSE/gRPC stream) → 20,000 long-lived connections
   → relay fleet of ~10 pods at ~2–5k connections each per region
Change rate: ~500 flag changes/day → tiny write load; fan-out per change: 20,000 notifications → ~400 KB × N? → send DELTAS (~1 KB) → 20 MB per change burst
Mobile/web: millions of clients → evaluate server-side or via CDN-cached per-segment payloads
```
**Conclusion:** this is a **distribution and safety** problem, not a throughput one: evaluate locally, push changes fast, and make the system unable to take everything down.

---

## 3. High-level design

```
Admin UI / API / Git (flags as code for ops knobs) ─▶ Flag service (control plane)
   - validation (schema, types, rule sanity), approvals, scheduled changes, audit log
   - store: Postgres (source of truth, versioned rulesets per project/environment)
   - publish: new ruleset version → event (Kafka) + snapshot to S3/CDN
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
  Relay/stream fleet (per region)   CDN/S3 snapshots (versioned JSON per project+env)
   - holds SSE/gRPC streams           - bootstrap + fallback polling every 30–60 s
          │                             │
          ▼                             ▼
  Service SDK (in every process): in-memory ruleset, LOCAL evaluation (µs),
     on start: load from local disk cache → CDN snapshot → stream for deltas
     on stream loss: keep last good ruleset; poll snapshot
     defaults in code for every flag (if nothing loaded)
     sends evaluation counts (sampled, aggregated) → analytics
Mobile/web: edge evaluation service returns flag values per user (cached, short TTL)
```

**Flow of a change (kill switch "new-checkout" off):**
1. Engineer flips the flag in the UI (prod change on a critical flag → requires second approver or uses an emergency "kill switch" path that's pre-approved).
2. Service validates, writes version 1043, audit logs it, publishes event.
3. Relays push the delta to all connected SDKs (< 2 s), snapshot updated on S3/CDN.
4. SDKs atomically swap rulesets; next evaluation returns `false` → old checkout path.

---

## 4. Deep dives

### 4.1 Local evaluation and bucketing
```
evaluate(flag, context{user_id, tenant, region, app_version}):
   for rule in flag.rules (ordered):
       if rule.matches(context): return rule.variation or rollout(rule.percentages)
   return flag.default
rollout: bucket = hash(flag_key + salt + user_id) % 100_000 → stable per user (consistent bucketing)
   → the same user stays in the same cohort as % grows 1% → 5% → 25% (no flip-flopping)
```
No network calls, no PII sent to the service (evaluation context stays local).

### 4.2 Config is code: safe rollout of flags themselves
The biggest risk: **a bad config change hits every service at once** (a pattern behind many large outages). Controls:
- **Validation:** types (bool/int/string/JSON schema), ranges for numeric knobs (timeout 50–5,000 ms), rule linting.
- **Staged propagation** for operational knobs: apply to canary instances/one region first (relays deliver by ring), observe SLOs, then all. Release toggles change % gradually anyway.
- **Approvals** for prod critical flags; **scheduled changes**; **instant rollback** to previous version (versioned rulesets).
- **Blast radius** limits: changes to flags affecting > X% of traffic in one step need approval.
- **Change events annotated** on dashboards (topic 27) so "what changed?" is answered instantly.

### 4.3 Static stability
| Situation | Behaviour |
|---|---|
| Flag service control plane down | No changes possible; all SDKs keep last ruleset → production unaffected |
| Relays down | SDKs poll CDN snapshots; changes delayed ≤ 60 s |
| CDN + relays down, process restarts | SDK loads ruleset from local disk cache (written on every update) |
| Nothing available, fresh process | **Code defaults** (safe values chosen per flag: usually "off" for new features, "current behaviour" for knobs) |
SDK initialization must **never block startup indefinitely**: bounded wait (e.g. 2 s), then start with cache/defaults.

### 4.4 Flag lifecycle
- Types: release (short-lived), experiment (time-boxed), ops/kill switch (long-lived), permission/entitlement (long-lived).
- Every flag has owner, type, expiry date; stale release flags (100% for 30 days) → automatic cleanup tickets; code references found by scanning repos.
- Limits on flag count per project to curb debt.

### 4.5 Clients outside our fleet
- Mobile/web: don't ship all rules (they leak business logic and are large); call an edge evaluation endpoint with the user context, get only that user's values; cache with short TTL; persist last values on device for offline.

---

## 5. Scaling
- Relays: stateless, scale on connection count; connection max-age with jitter to rebalance after deploys.
- Deltas instead of full rulesets; compression.
- Analytics of evaluations: SDK aggregates counts per flag/variation and flushes every 60 s → small volume.

---

## 6. Reliability and DR
- Control plane: multi-AZ, DB with backups; multi-region not required for serving (static stability), but needed for **making emergency changes** during a region outage → control plane warm standby in the second region with replicated DB (kill switches must work during incidents!).
- Emergency path: a minimal "break-glass" mechanism (e.g. a signed override file in S3 that SDKs also poll) if the control plane is totally down.
- SLOs: propagation p99 < 10 s; control plane availability 99.95%; zero production impact from service outages (measured via game days).

---

## 7. Security
- RBAC per project/environment; approvals for prod critical flags; SSO; audit log immutable.
- SDK keys per environment (server-side keys never shipped in mobile apps).
- Rules may contain tenant IDs or user lists → avoid PII where possible; use segments referencing IDs managed securely.
- Integrity: rulesets signed by the control plane; SDKs verify signatures (protects against tampered snapshots in CDN/S3).

---

## 8. Observability
- Control plane: change rate, validation failures, approval latency.
- Distribution: connected SDK count per region/service, propagation delay (relay timestamps → SDK ack), snapshot freshness.
- SDK: ruleset version per instance (spot stragglers), evaluation errors (type mismatches), fallback-to-default events.
- Business: experiment metrics via analytics pipeline.

---

## 9. Cost and build-vs-buy
- Build: small infra (DB, relays, CDN) but significant engineering for SDKs in every language, UI, governance.
- Buy: LaunchDarkly, Unleash (OSS/self-hosted), Flagsmith, AWS AppConfig (feature flags + config with validators and deployment strategies), OpenFeature as a vendor-neutral SDK API.
- Senior answer: use **OpenFeature** SDK interfaces + a vendor or Unleash/AppConfig unless requirements are unusual; build the governance around it.

---

## 10. Trade-offs

| Decision | Chosen | Gave up |
|---|---|---|
| Local evaluation | µs latency, resilience | Clients must receive rulesets (distribution system) |
| Streaming + snapshot fallback | Fast propagation, robust | Two delivery paths to maintain |
| Staged propagation for knobs | Limits blast radius | Slightly slower global changes |
| Code defaults required | Works with nothing available | Developer discipline (enforced by SDK) |

---

## 11. Follow-up questions

1. **Q: How do you prevent a bad flag change from causing a global outage?**
   A: Validation and typed schemas, approvals for critical flags, staged rollout of the change (canary instances → region → all) with SLO checks, versioned rulesets with one-click rollback, change annotations for fast diagnosis, and limits on how much traffic one change can affect at once.

2. **Q: The flag service is down during an incident and you need to flip a kill switch. What now?**
   A: Use the warm standby control plane in the other region, or the break-glass signed override file that SDKs poll. Design for this in advance; a kill switch you can't flip during an outage is useless.

3. **Q: How do you keep a user in the same experiment group across services?**
   A: Deterministic bucketing: hash(flag key + salt + stable user ID) mod N, evaluated the same way by every SDK; pass the same user context (user ID) through services.

4. **Q: Feature flags vs config in Git (GitOps): which for what?**
   A: Git/GitOps for most configuration: reviewed, versioned, deployed with the app. Runtime flags for things that must change faster than a deploy or per user/tenant: release toggles, kill switches, experiments, emergency knobs. Even runtime changes get audit, validation and staged rollout.
