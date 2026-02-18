# PQ Gateway -- Sovereign AI Governance Layer

* **Specification Version:** 1.0.0
* **Status:** Implementation Ready
* **Date:** 2026
* **Author:** rosiea
* **Contact:** [PQRosie@proton.me](mailto:PQRosie@proton.me)
* **Licence:** Apache License 2.0 — Copyright 2026 rosiea
* **PQ Ecosystem:** PRODUCT — Composes existing PQ specifications into a deployable governance gateway. Introduces no new enforcement primitives.

---

## Summary

PQ Gateway is a sovereignty-preserving AI governance layer that sits between users and any model provider.

Users define their own policy. The gateway enforces it structurally. Model providers do not need to be modified, trusted, or aware they are governed. Enforcement does not depend on model behaviour. It depends on deterministic evaluation of evidence.

Every prompt is session-bound, tick-bounded by Epoch Clock, intent-hashed, and evaluated against the user's policy before it reaches any provider. Every response passes back through the same enforcement boundary. Every operation produces signed, auditable receipts. Missing, stale, or invalid evidence results in refusal. No exceptions. No degraded modes.

Any model. Any provider. Your rules. Enforced.

---

## 0. Authority Boundary and Non-Goals

### 0.1 Authority Boundary (Normative)

PQ Gateway grants no authority.

PQ Gateway components MUST NOT:

1. Evaluate predicates.
2. Produce EnforcementOutcome artefacts.
3. Override or reinterpret PQSEC decisions.
4. Implement degraded allow logic for Authoritative operations.
5. Treat provider-side safety fields as permission.

All enforcement decisions remain exclusively within PQSEC.

### 0.2 Non-Goals (Normative)

This document does not redefine:

* STP (PQSF §27).
* Epoch Clock tick issuance or profile governance.
* PQSEC predicate semantics, enforcement refusal codes, or lockout logic.
* PQAI drift/fingerprint mathematics or tool taxonomy.
* PQHR rendering requirements.

---

## 1. Product Layer Components

PQ Gateway requires six product-layer components. Each composes existing primitives into a deployable surface and produces evidence-only receipts.

| Component | Identifier | Purpose |
|---|---|---|
| Gateway Router | PQGW-ROUTER | Session orchestration and model routing |
| Policy Authoring | PQGW-CONTROL | User-defined governance policy creation |
| Provider Adapters | PQGW-ADAPTER | Inference API normalisation |
| Billing and Metering | PQGW-METER | Governed usage tracking |
| Enrollment and Onboarding | PQGW-ONBOARD | Zero-to-operational user setup |
| Provider Registry | PQGW-REGISTRY | Model discovery and trust catalogue |

---

## What Exists (Specification Layer)

The following capabilities are fully specified and require implementation, not design:

| Capability | Source |
|---|---|
| Enforcement engine | PQSEC (sole enforcement authority) |
| Governed prompt input | PQAI §10 SafePrompt |
| Secure transport | PQSF §27 STP |
| Model identity and drift | PQAI §7, §8, §9 |
| Time authority | Epoch Clock |
| Canonical encoding | PQSF §7 |
| Tool governance | PQAI §27.2 Tool Capability Profiles |
| Credential derivation | PQSF §13 Universal Secret Derivation |
| Platform bridging | PQAA |
| Human-readable rendering | PQHR |
| Persistent state | PQPS |
| Agent enrollment | PQAI Annex AA |
| Gateway adapter pattern | PQSF Annex AI |
| Extension admission | PQSEC Annex AX |
| Delegation and spending bounds | PQHD Annex J |
| Emission discipline | PQAI §20A, PQPS §4 |

---

# P1 — PQGW-ROUTER: Gateway Router and Session Orchestration

## P1.1 Purpose

The Gateway Router is the product front door. It accepts governed requests, establishes STP sessions to provider endpoints, and routes requests in accordance with user policy and registry constraints.

STP defines the protocol mechanics. This section defines the routing and orchestration responsibilities without creating a new authority surface.

## P1.2 Authority Boundary (Normative)

The Gateway Router:

* MUST NOT evaluate predicates.
* MUST NOT approve requests.
* MUST treat PQSEC's EnforcementOutcome as the only authoritative decision input.
* MUST refuse to route Authoritative operations without a valid EnforcementOutcome.

## P1.2A Intent Hash Construction (Normative)

For all PQ Gateway operation attempts, `intent_hash` MUST be computed deterministically as follows:

1. Construct `InferenceRequestCanonical` as defined in P2.4.
2. Compute `canonical_bytes = DetCBOR(InferenceRequestCanonical)` using PQSF deterministic CBOR rules (PQSF §7).
3. Compute `intent_hash = SHAKE256-256(canonical_bytes)` (32-byte output).

Rules:

1. `InferenceRequestCanonical` MUST NOT contain a `signature` field. If a signature exists in a wrapping container, it is not part of the hash input.
2. Implementations MUST reject any non-canonical encoding of `InferenceRequestCanonical` when `intent_hash` is computed or verified.
3. Independent implementations given identical semantic input MUST produce byte-identical `canonical_bytes` and identical `intent_hash`.

**Binding rule:** Any artefact carrying `intent_hash` (including `GovernedRequestEnvelope`, `InferenceReceipt`, and `EnforcementOutcome` binding checks) MUST use the value computed by this procedure.

## P1.3 Inbound Envelope (Normative)

The router MUST accept a single inbound request envelope:

```
GovernedRequestEnvelope = {
  v:                      1,
  session_id:             bstr(16),
  decision_id:            bstr(16),
  intent_hash:            bstr(32),
  operation_class:        "Authoritative" / "NonAuthoritative",
  policy_hash:            bstr(32),
  enforcement_outcome:    bstr / null,
  model_target:           ModelTarget,
  request_body:           bstr,
  issued_tick:            uint,
  expiry_tick:            uint,
  exporter_hash:          bstr(32) / null
}
```

Rules:

1. `issued_tick` and `expiry_tick` MUST be Epoch Clock ticks.
2. `expiry_tick` MUST be strictly greater than `issued_tick`.
3. Authoritative requests MUST include `enforcement_outcome` and `exporter_hash`. Missing `enforcement_outcome` for Authoritative operations MUST result in refusal.
4. `enforcement_outcome` contains canonical PQSEC EnforcementOutcome bytes. The router verifies binding (matching `session_id`, `decision_id`, `intent_hash`) but does not evaluate predicates.
5. `request_body` is a canonical provider-agnostic payload defined by P2.
6. Requests with `current_tick >= expiry_tick` MUST be refused.

## P1.4 Model Target Resolution (Normative)

Routing MUST be based on explicit target constraints, not content inspection.

```
ModelTarget = {
  provider_id:    tstr,
  model_id:       tstr / null,
  model_version:  tstr / null,
  fallback:       "refuse" / "policy_default"
}
```

Resolution rules:

1. If `model_id` is set, resolve only to that model within the Provider Registry Snapshot (P6).
2. If `model_id` is null, use the policy-default model for the operation class.
3. Silent substitution is prohibited. If the resolved model differs from the request target, refuse. Refusal code: `E_MODEL_TARGET_AMBIGUOUS`.
4. If `fallback = "refuse"`, any resolution failure is terminal.
5. If the resolved model is not present in the current registry snapshot, refuse. Refusal code: `E_PROVIDER_NOT_REGISTERED`.

## P1.5 Routing Strategy (Normative)

Among eligible candidates, routing MUST follow a deterministic security-first selection order:

1. **Policy eligibility** — provider and model permitted by active policy.
2. **Evidence posture** — higher-assurance declared evidence preferred (identity pinned > fingerprinted > unverified).
3. **Attestation tier** — when present, per P6 evidence scorecard.
4. **Cost, latency, load** — only among equivalent eligible candidates.

This preference order MUST be policy-configurable. It MUST NOT become an implicit trust oracle. The router MUST NOT infer trust from routing preference.

## P1.6 Session Orchestration (Normative)

The router manages the mapping between user-facing STP sessions and provider-facing adapter connections.

Rules:

1. The router MUST establish a provider-facing STP session per operation attempt (operation-scoped) unless policy explicitly permits session reuse for NonAuthoritative operations.
2. STP handshakes and session lifecycle follow PQSF §27.
3. Provider-facing exporter binding MUST be derived independently from user-facing exporter binding. No key material sharing between the two session contexts.
4. Provider connections MUST be destroyed at the end of the attempt (PQSF §27.5.5 key material lifecycle).
5. Session timeout follows Epoch Clock ticks, not wall clocks.
6. The router MUST maintain session isolation — no state leakage between user sessions.

## P1.7 Response Governance (Normative)

Responses MUST be normalised by P2 and then passed to PQSEC for evaluation before delivery to the user.

Delivery rules:

1. If PQSEC produces ALLOW, the response is delivered within the user's STP session.
2. If PQSEC produces DENY or FAIL_CLOSED_LOCKED, the response is discarded and the user receives the refusal code only.
3. All outcomes produce receipts (P1.8).
4. The router MUST NOT retry on provider failure. Single-attempt semantics apply (ZEB, "Execution Recovery Semantics"). Governed retry requires a fresh PQSEC evaluation cycle.

## P1.8 Inference Receipt (Normative)

Every request MUST produce a ReceiptEnvelope.

**ReceiptEnvelope.type:** `"pqgw.inference_receipt"`

```
InferenceReceiptBody = {
  v:                    1,
  session_id:           bstr(16),
  decision_id:          bstr(16),
  intent_hash:          bstr(32),
  provider_id:          tstr,
  model_id:             tstr,
  model_identity_hash:  bstr(32) / null,
  request_commitment:   bstr(32),
  response_commitment:  bstr(32) / null,
  status:               "DELIVERED" / "REFUSED" / "PROVIDER_FAILED",
  refusal_code:         tstr / null,
  issued_tick:          uint,
  latency_bucket:       "fast" / "normal" / "slow" / "timeout",
  signature:            bstr
}
```

Rules:

1. `model_identity_hash` is SHAKE256-256 of the pinned ModelIdentity (PQAI §7) if the model is identity-pinned. Null if not pinned.
2. `request_commitment` is SHAKE256-256 of the canonical request bytes.
3. `response_commitment` is SHAKE256-256 of the canonical response bytes. Null if refused or provider failed.
4. `refusal_code` is present if and only if `status` is `"REFUSED"`.
5. Precise latency values MUST NOT be recorded in receipts, logs, or telemetry.

**P1.8.1 Constant-Shape Latency Buckets**

| Bucket | Range |
|---|---|
| `"fast"` | < 2 seconds |
| `"normal"` | 2–10 seconds |
| `"slow"` | 10–30 seconds |
| `"timeout"` | > 30 seconds |

---

# P2 — PQGW-ADAPTER: Provider Adapter Layer

## P2.1 Purpose

Provider Adapters translate between proprietary inference APIs and canonical PQ artefacts, enabling any-model-provider interoperability without requiring provider modification.

This is the model-API equivalent of PQAA: a governed normalisation shim that produces canonical receipts but never grants authority.

## P2.2 Authority Boundary (Normative)

Provider Adapters:

* MUST NOT evaluate predicates. Tool call classification (P2.5) is evidence production analogous to PQAI action class classification, not predicate evaluation.
* MUST NOT interpret provider safety fields as permission.
* MUST NOT introduce hidden system prompts or undisclosed transformations.
* MUST NOT retry on failure (single-attempt; governed retry requires fresh PQSEC evaluation).
* MUST NOT cache provider responses beyond the operation scope.
* MUST emit canonical evidence artefacts only.

## P2.3 Governance: Admission and Integrity (Normative)

Each adapter MUST be admitted under PQSEC Annex AX extension/admission discipline and MUST follow PQSF Annex AI gateway adapter pattern.

Adapters MUST be manifest-bound. The adapter manifest extends GatewayManifestBody (PQSF Annex AI.2) with:

```
ProviderAdapterManifest extends GatewayManifestBody {
  provider_class:           "cloud_api" / "local_inference" / "federated",
  supported_model_ids:      [+ tstr],
  max_context_tokens:       uint / null,
  supports_streaming:       bool,
  supports_tool_use:        bool,
  response_format:          "text" / "structured" / "multimodal",
  privacy_classification:   "provider_visible" / "encrypted_inference" / "local_only"
}
```

Rules:

1. `supported_model_ids` MUST enumerate every model the adapter can route to. Wildcard model identifiers are prohibited.
2. `privacy_classification` is descriptive evidence for policy evaluation. It does not modify enforcement.
3. Adapters with `privacy_classification = "provider_visible"` MUST be declared in the user's policy as acceptable for the relevant operation classes.
4. Any adapter update or scope expansion (new model IDs, new capabilities) requires re-admission via PQSEC Annex AX fail-closed re-admission.
5. `binary_hash` MUST match the installed adapter binary. Mismatch results in refusal. Refusal code: `E_ADAPTER_BINARY_MISMATCH`.

## P2.4 Canonical Request Mapping (Normative)

Adapters MUST translate a provider-agnostic canonical request into provider-specific format:

```
InferenceRequestCanonical = {
  v:                1,
  session_id:       bstr(16),
  decision_id:      bstr(16),
  intent_hash:      bstr(32),
  model_id:         tstr,
  tool_profile_ref: tstr / null,
  content:          bstr,
  parameters:       bstr / null,
  issued_tick:      uint,
  expiry_tick:      uint
}
```

Rules:

1. No content rewriting beyond structural format translation.
2. Parameter mapping MUST be schema-bound and deterministic against the Tool Capability Profile's parameter schema (PQAI §27.2).
3. Tool use mapping MUST be bounded to PQAI tool capability declarations.
4. System prompts not declared in the SafePrompt MUST NOT be injected.
5. Raw prompt content (`InferenceRequestCanonical.content`) MUST NOT be retained beyond the operation boundary. Only hash commitments MAY be recorded.

## P2.5 Canonical Response Mapping (Normative)

Adapters MUST normalise provider responses into:

```
InferenceResponseCanonical = {
  v:                    1,
  provider_id:          tstr,
  model_id:             tstr,
  response_body_hash:   bstr(32),
  tool_calls:           [* ToolCallRecord] / null,
  token_usage:          TokenUsage / null,
  provider_metadata:    bstr(32) / null,
  collected_tick:       uint,
  suite_profile:        tstr,
  signature:            bstr
}

ToolCallRecord = {
  tool_id:              tstr,
  parameters_hash:      bstr(32),
  classification:       tstr
}

TokenUsage = {
  input_tokens:         uint,
  output_tokens:        uint,
  total_tokens:         uint
}
```

Rules:

1. `response_body_hash` is SHAKE256-256 over raw response bytes. Raw response bytes MUST NOT be retained beyond the operation boundary.
2. `tool_calls` are extracted and classified against the active Tool Capability Profile before PQSEC evaluation. `classification` MUST use PQAI §11.1 action class values (`style`, `explain`, `advise`, `decide`, `execute`, `authority`). Unrecognised tool calls MUST be classified as `authority` per PQAI §11 escalation principle.
3. `provider_metadata` is an opaque SHAKE256-256 hash of provider-specific response metadata. Raw metadata MUST NOT be retained.
4. `token_usage` is informative only and MUST NOT be used for enforcement decisions.

## P2.6 Provider Credential Lifecycle (Normative)

Provider API credentials follow PQSF §13 Universal Secret Derivation with `derivation_purpose: "provider_credential"`. This specialises PQSF Annex AI.3 `gateway_credential` for model-provider contexts, using the registered §13.3 context string `"PQSF-Secret:ProviderCredential"`.

1. Credentials are derived per-provider, per-delegation scope.
2. Credentials rotate atomically (PQSF §27.7.4) on a maximum 30-day cycle or on any compromise suspicion.
3. On agent or user deauthorisation, all derived provider credentials are invalidated via delegation scope revocation.
4. Raw API keys provided by the user during onboarding are migrated via `pqsf.credential_migration` receipts and destroyed from gateway memory immediately after derivation context is established.
5. The credential emission prohibition in PQSF Annex AI.3 applies.

---

# P3 — PQGW-CONTROL: Policy Authoring Interface

## P3.1 Purpose

This component realises the user-defined control surface: a structured interface for defining which predicates matter, which tools are permitted, spending limits, drift tolerance, and explicit-approval requirements.

PQHR renders policy for inspection. PQSEC enforces policy. This component creates policy.

## P3.2 Authority Boundary (Normative)

The Policy Authoring Interface:

* MUST NOT evaluate predicates.
* MUST NOT claim a policy is active until accepted by PQSEC policy lifecycle semantics.
* MUST compile policy into canonical artefacts for PQSEC consumption.

## P3.3 Policy Domains (Normative)

The interface exposes policy configuration across the following domains:

| Domain | Controls | Spec Reference |
|---|---|---|
| Model Permissions | Which providers and models are permitted | P6 Registry, PQAI §7 |
| Tool Governance | Which tools the model may use, parameter bounds | PQAI §27.2 Tool Capability Profiles |
| Spending Limits | Transaction caps, per-operation limits, daily/weekly bounds | PQHD Annex J, Annex T, Annex U |
| Drift Tolerance | Thresholds for WARNING and CRITICAL drift states | PQAI §9.2 |
| Time Freshness | Maximum tick staleness before refusal | PQSEC §18 |
| Session Bounds | Maximum session duration, idle timeout | PQSF §27.5 STP |
| Privacy Posture | Which provider privacy classifications are acceptable | P2.3 privacy_classification |
| Supervision Level | Default supervision lattice position per operation class | PQAI §27.5 |
| Lockout Thresholds | Custom K values for lockout escalation | PQSEC §25, Annex AB |
| Notification Preferences | When and how to notify the holder | P5.6 |

## P3.4 Policy Templates (Normative)

The interface MUST provide three starting templates:

**Conservative (default):** All Authoritative operations require HUMAN_APPROVE. No spending authority. No tool use beyond read-only. Drift threshold WARNING at Hamming distance 2. No `provider_visible` providers unless explicitly added.

**Standard:** HUMAN_CONFIRM for Authoritative operations. Spending bounded by explicit DelegationConstraint. Tool use permitted within Tool Capability Profile. Drift threshold WARNING at 3, CRITICAL at 5. Provider selection per model-target policy.

**Autonomous:** HUMAN_APPROVE for irreversible custody operations only. Broader delegation with explicit spending caps. Full tool profile within declared scope. Strict drift policy. Explicit recovery paths. Suitable for high-trust, bounded agent deployments.

Template names are informative only. The compiled PolicyBundle is the truth.

## P3.5 Policy Compilation (Normative)

Saving a policy MUST compile to:

1. A canonical PolicyObject (PQSF §17.1).
2. A PolicyBundle (PQSF §17.2) binding the PolicyObject with governance metadata and signature.
3. A DelegationConstraint (PQHD Annex J) if spending authority is granted.
4. A Tool Capability Profile (PQAI §27.2) defining permitted tools and parameters.

Compilation rules:

1. All artefacts MUST be canonical CBOR (PQSF §7).
2. All artefacts MUST be signed under the holder's active suite_profile. Any standalone artefact containing `signature: bstr` MUST also contain `suite_profile: tstr` and MUST be verified under that profile.
3. PolicyBundle MUST include `issued_tick` from Epoch Clock.
4. PolicyBundle `governance.version` MUST be monotonically increasing.
5. Policy changes MUST produce a new PolicyBundle version. No in-place mutation.
6. The previous PolicyBundle MUST be retained for audit via PQPS with verifiable deletion semantics.
7. Compilation MUST produce a `pqgw.policy_compiled` receipt.

## P3.6 Policy Rendering Before Activation (Normative)

Every compiled policy MUST be rendered through PQHR before activation:

1. The user MUST see the full policy in PQHR Summary View before signing.
2. The interface MUST NOT activate policy that the user has not reviewed.
3. Policy rendering MUST include the diff from the previous active policy.
4. PQHR prohibited rendering patterns (§4.5–§4.7) apply to the authoring interface.

## P3.7 Deterministic UI-to-Predicate Mapping (Normative)

The interface MUST provide a deterministic mapping between UI controls and predicate requirements.

Rules:

1. Custom rules bind to existing PQSEC predicate names only. The interface MUST NOT invent new predicates.
2. Policy-defined toleration of UNAVAILABLE MUST be limited to NonAuthoritative operations per PQSEC §8A.5.

## P3.8 Custom Policy Rules (Normative)

The holder MAY define custom policy rules beyond the template domains:

```
CustomPolicyRule = {
  v:                1,
  rule_id:          tstr,
  description:      tstr,
  predicate_target: tstr,
  condition:        "require" / "prohibit" / "warn",
  parameters:       { * tstr => tstr / uint / bool },
  issued_tick:      uint,
  suite_profile:    tstr,
  signature:        bstr
}
```

Rules:

1. Custom rules bind to existing PQSEC predicate names only.
2. `condition = "require"` maps to: if the predicate evaluates FALSE or UNAVAILABLE, the operation is refused.
3. `condition = "prohibit"` maps to: if the predicate evaluates TRUE, the operation is refused.
4. `condition = "warn"` maps to: emit a notification (P5.6) but do not refuse.
5. Custom rules are compiled into the PolicyObject and enforced by PQSEC identically to template-derived rules.
6. `rule_id` MUST be unique within the PolicyObject.

---

# P4 — PQGW-METER: Billing and Metering

## P4.1 Purpose

Track governed usage for subscriptions, rate limits, and user-defined caps without turning billing into an enforcement bypass.

PQSF §17B provides privacy-preserving telemetry structure. This component defines product-level metering and billing logic.

## P4.2 Authority Boundary (Normative)

Billing:

* MUST NOT permit anything PQSEC denied.
* MAY refuse for quota reasons, but such refusal MUST be distinguishable from PQSEC refusal codes.
* MUST be constant-shape to avoid timing and usage side channels.
* MUST NOT create side-channel "allow anyway" paths.

Billing refusals are additive only. Billing may refuse an operation that PQSEC has allowed (for quota or rate reasons), but MUST NOT permit an operation that PQSEC has denied. Where product-layer billing refusal is used, it MUST use PQSEC Annex AE.59 product refusal codes (`E_BILLING_QUOTA_EXCEEDED`, `E_BILLING_RATE_LIMITED`).

## P4.3 Usage Record (Normative)

Every inference operation MUST produce a usage record derived from the inference receipt (P1.8):

```
UsageRecord = {
  v:                      1,
  user_binding:           bstr(32),
  operation_class:        tstr,
  provider_id:            tstr,
  model_id:               tstr,
  token_usage:            TokenUsage,
  token_usage_estimated:  bool,
  status:                 "DELIVERED" / "REFUSED" / "PROVIDER_FAILED",
  issued_tick:            uint,
  billing_period_id:      tstr,
  suite_profile:          tstr,
  signature:              bstr
}
```

Rules:

1. `user_binding` is SHAKE256-256 of the user's wallet public key. Not the key itself.
2. If the provider does not report token usage, the field is estimated from request/response byte length with `token_usage_estimated = true`.
3. Usage records MUST NOT include prompt content, response content, or session content.
4. Usage records are stored via PQPS with verifiable deletion semantics.

## P4.4 Billing Tiers (Normative)

| Tier | Description | Policy Constraints |
|---|---|---|
| Free | Rate-limited, single provider, conservative policy template | Rate limit via PQSEC, restricted model set |
| Standard | Multiple providers, full policy authoring, standard rate limits | User-defined within tier bounds |
| Sovereign | Unlimited providers, custom adapters, priority routing, self-hosted option | User-defined, no platform-imposed caps |

Tier enforcement:

1. Tier determines the default policy constraints applied at enrollment.
2. Tier restrictions are expressed as PolicyObject constraints and enforced by PQSEC identically to user-defined policy.
3. Tier upgrades and downgrades are Authoritative operations producing receipts.
4. The gateway MUST NOT enforce tier restrictions outside the PQSEC policy path. No side-channel enforcement.

## P4.5 Payment Integration (Normative)

Payment is accepted via:

1. **Bitcoin (on-chain)** — via PQHD custody, governed by the user's own wallet.
2. **Bitcoin (Lightning)** — via gateway-provided invoice, verified by Epoch Clock tick.
3. **Fiat (card/subscription)** — via external payment processor, bridged through PQSF Annex AI Gateway Adapter Pattern.

Payment receipts:

1. Every payment MUST produce a `pqgw.payment_receipt`.
2. Payment receipts bind to `billing_period_id` and `user_binding`.
3. Payment verification is evidence consumed by policy. Non-payment triggers policy-defined spending limit reduction, not gateway-level enforcement.

## P4.6 Privacy-Preserving Telemetry (Normative)

Aggregate usage telemetry follows PQSF §17B FleetTelemetryEnvelope:

1. No stable device identifiers.
2. Tick-windowed timing (no precise timestamps).
3. Operation-scoped emission.
4. Constant-shape telemetry envelopes.
5. Telemetry is used for capacity planning and billing reconciliation only.

---

# P5 — PQGW-ONBOARD: Enrollment and Onboarding

## P5.1 Purpose

Make the system usable while preserving fail-closed guarantees. Compose existing primitives into a coherent zero-to-operational flow.

PQAI Annex AA defines agent enrollment. This section defines user enrollment and onboarding.

## P5.1A Authority Boundary (Normative)

PQGW-ONBOARD:

* MUST NOT evaluate predicates.
* MUST NOT produce EnforcementOutcome artefacts.
* MUST NOT create approval signals outside PQSEC.

Enrollment gating (steps must complete in order, failure is fail-closed) is a product-layer sequencing requirement, not PQSEC enforcement. Enrollment step completion is operational prerequisite evidence, not authority.

## P5.2 Enrollment Sequence (Normative)

| Step | Action | Spec Primitive | Receipt |
|---|---|---|---|
| 1 | Create PQ wallet | PQHD §10 HD key derivation | `pqgw.wallet_init` |
| 2 | Bootstrap Epoch Clock consumer | Epoch Clock §4 (pinned profile, first verified tick) | `pqgw.epoch_bootstrap` |
| 3 | Select policy template | P3.4 Policy Templates | — |
| 4 | Customise policy | P3.3 Policy Domains | — |
| 5 | Review and sign policy | P3.6 Policy Rendering via PQHR | `pqgw.policy_compiled` |
| 6 | Register model providers | P6.4 Provider Registration | `pqgw.provider_registered` |
| 7 | Provision provider credentials | P2.6 via PQSF §13 | `pqsf.credential_migration` |
| 8 | Configure notifications | P5.6 Notification Policy | — |
| 9 | (Optional) Enroll agent | PQAI Annex AA.1 Agent Enrollment | `pqai.agent_enrollment` |
| 10 | Establish first STP session | PQSF §27.5 STP handshake | — |
| 11 | Send first governed prompt | P1.3 Inbound Envelope | `pqgw.inference_receipt` |
| 12 | Enrollment complete | — | `pqgw.enrollment_complete` |

Rules:

1. Steps 1–5 MUST complete before any model interaction.
2. Steps MUST execute in order. Each step producing a receipt MUST complete receipt generation before the next step begins.
3. Enrollment failure at any step is fail-closed. The user cannot interact with models until all required steps are complete.
4. The enrollment flow MUST be rendered through PQHR at each step so the user understands what they are configuring.

## P5.3 Wallet Bootstrap (Normative)

For users without an existing PQ wallet:

1. The gateway provides a wallet creation flow using PQHD §10 deterministic key derivation.
2. The user receives their root secret via PQHR-rendered display with explicit backup instructions.
3. The root secret is never transmitted, stored by the gateway, or logged.
4. Wallet creation produces a `pqgw.wallet_init` receipt.

For users with an existing PQ wallet:

1. The gateway accepts wallet import via PQSF §27 STP session.
2. Import is an Authoritative operation requiring holder signature verification.

## P5.4 Credential Migration (Normative)

Users with existing provider API keys (OpenAI, Anthropic, etc.):

1. The user provides the raw API key during onboarding via secure input (STP-protected, never logged).
2. The gateway derives a domain-scoped credential via PQSF §13 Universal Secret Derivation.
3. Raw API key lifecycle follows P2.6 (migration receipt, immediate destruction, credential emission prohibition).
4. Subsequent provider access uses only derived credentials.

## P5.5 First NonAuthoritative Request (Normative)

The enrollment flow SHOULD culminate in a first NonAuthoritative governed request (Step 11) to confirm end-to-end operation:

1. This request MUST use the Conservative policy template constraints.
2. This request MUST produce a valid `pqgw.inference_receipt`.
3. If the first request fails, the enrollment is considered incomplete and the failure reason is rendered via PQHR.

## P5.6 Notification Policy (Normative)

The user configures notification preferences during enrollment:

```
NotificationPolicy = {
  v:                      1,
  channels:               [+ NotificationChannel],
  notify_on_refusal:      bool,
  notify_on_drift:        "never" / "warning" / "critical",
  notify_on_lockout:      bool,
  notify_on_spending:     "never" / "per_operation" / "threshold",
  spending_threshold:     uint / null,
  digest_interval_ticks:  uint / null,
  suite_profile:          tstr,
  signature:              bstr
}

NotificationChannel = {
  channel_type:           "in_app" / "email" / "push" / "webhook",
  target:                 tstr,
  priority_floor:         "info" / "warning" / "critical"
}
```

Rules:

1. At least one NotificationChannel MUST be configured.
2. `notify_on_lockout` MUST default to true and MUST NOT be set to false during initial enrollment. The user MAY change it after first successful operation.
3. Notification delivery is best-effort and non-authoritative. Notification failure MUST NOT block enforcement.
4. Notification content MUST NOT include raw prompt content, response content, or credentials. Hash references only.

---

# P6 — PQGW-REGISTRY: Provider Discovery and Trust Catalogue

## P6.1 Purpose

A governed catalogue of available model providers and their capabilities. STP Web Discovery (PQSF §27.8) defines endpoint discovery. This component defines a Provider Registry Snapshot suitable for routing decisions without becoming an authority oracle.

## P6.2 Authority Boundary (Normative)

Registry entries are advisory. No registry entry implies permission. Policy decides. PQSEC enforces.

## P6.3 Provider Registry Snapshot (Normative)

Clients MUST consume provider data via a signed, time-bounded snapshot:

```
ProviderRegistrySnapshot = {
  v:                1,
  snapshot_id:      bstr(16),
  providers:        [+ ProviderEntry],
  issued_tick:      uint,
  expiry_tick:      uint,
  suite_profile:    tstr,
  signature:        bstr
}

ProviderEntry = {
  provider_id:              tstr,
  provider_name:            tstr,
  provider_class:           "cloud_api" / "local_inference" / "federated",
  endpoint_uri:             tstr,
  stp_supported:            bool,
  adapter_manifest_hash:    bstr(32),
  privacy_classification:   "provider_visible" / "encrypted_inference" / "local_only",
  models:                   [+ RegisteredModel],
  status:                   "active" / "suspended" / "revoked"
}

RegisteredModel = {
  model_id:                 tstr,
  model_version:            tstr / null,
  identity_hash:            bstr(32) / null,
  fingerprint_hash:         bstr(32) / null,
  capabilities:             [* tstr],
  max_context_tokens:       uint / null,
  supports_tool_use:        bool,
  declared_evidence:        [* tstr]
}
```

Rules:

1. The snapshot is time-bounded. If the snapshot is unavailable or `current_tick >= expiry_tick`, routing MUST fail closed for Authoritative operations unless policy explicitly permits a restricted fallback for NonAuthoritative operations.
2. `identity_hash` is SHAKE256-256 of a ModelIdentity artefact (PQAI §7) if the model is identity-pinned. Null if not pinned.
3. `fingerprint_hash` is SHAKE256-256 of a baseline BehavioralFingerprint (PQAI §8) if the model has been fingerprinted. Null if not fingerprinted.
4. Models without identity pinning or fingerprinting MAY be registered but MUST be flagged as `unverified` in policy rendering (PQHR).
5. Provider status changes (suspension, revocation) are Authoritative operations requiring holder approval.

## P6.4 Registration Rules (Normative)

1. Provider registration is an Authoritative operation. It MUST require holder approval.
2. `adapter_manifest_hash` MUST reference a valid, admitted Provider Adapter (P2.3).
3. Providers discovered via STP Web Discovery (PQSF §27.8) are presented to the user for approval before registration. Automatic registration without holder approval is prohibited.
4. Registration MUST produce a `pqgw.provider_registered` receipt.

## P6.5 Provider Verification (Normative)

Registered providers are periodically verified:

1. Fingerprint re-measurement against baseline (PQAI §8, §9).
2. Adapter binary hash verification against manifest (PQSF Annex AI.2).
3. STP endpoint reachability (informative only).
4. Verification frequency is policy-defined. Default: every 24 hours by tick window.
5. Verification failure sets `status = "suspended"`. Suspended providers MUST NOT be routed to until re-verified and holder-approved.
6. Verification MUST produce a `pqgw.provider_verification` receipt.

## P6.6 Evidence Scorecard (Informative)

The registry MAY compute and surface an informative scorecard per provider indicating:

* Identity pinned (PQAI ModelIdentity present)
* Fingerprint baseline exists (PQAI BehavioralFingerprint present)
* Drift monitoring cadence
* Adapter admission status
* Privacy classification
* STP support

The scorecard is advisory. Routing MAY prefer higher-posture providers only under policy direction (P1.5). The scorecard MUST NOT be treated as a trust score or authority signal.

---

## 7. Receipt Namespace

Product-layer receipts use the `pqgw.*` namespace:

| Receipt Type | Producer | Purpose |
|---|---|---|
| `pqgw.inference_receipt` | PQGW-ROUTER (P1) | Records every inference operation |
| `pqgw.policy_compiled` | PQGW-CONTROL (P3) | Records policy compilation events |
| `pqgw.usage_receipt` | PQGW-METER (P4) | Records per-operation usage accounting |
| `pqgw.payment_receipt` | PQGW-METER (P4) | Records payment events |
| `pqgw.enrollment_complete` | PQGW-ONBOARD (P5) | Records successful user enrollment |
| `pqgw.wallet_init` | PQGW-ONBOARD (P5) | Records PQ wallet creation during enrollment |
| `pqgw.epoch_bootstrap` | PQGW-ONBOARD (P5) | Records Epoch Clock consumer bootstrap during enrollment |
| `pqgw.provider_registered` | PQGW-REGISTRY (P6) | Records provider registration |
| `pqgw.provider_verification` | PQGW-REGISTRY (P6) | Records provider verification outcomes |

Namespace rules:

1. `pqgw.*` receipts grant no authority. They are evidence only.
2. Producers MUST be admitted via PQSEC Annex AX if operating outside the Holder Execution Boundary.
3. Receipts MUST be deterministic CBOR and signed.

### 7.1 Receipt Body Schemas

`pqgw.inference_receipt` body schema is defined in P1.8 (`InferenceReceiptBody`).

`pqgw.usage_receipt` body is the `UsageRecord` defined in P4.3.

**ReceiptEnvelope.type:** `"pqgw.policy_compiled"`

```
PolicyCompiledBody = {
  policy_bundle_hash:     bstr(32),
  previous_bundle_hash:   bstr(32) / null,
  governance_version:     uint,
  template_id:            tstr / null,
  custom_rule_count:      uint,
  delegation_attached:    bool,
  tool_profile_attached:  bool
}
```

**ReceiptEnvelope.type:** `"pqgw.payment_receipt"`

```
PaymentReceiptBody = {
  user_binding:           bstr(32),
  billing_period_id:      tstr,
  payment_method:         "lightning" / "on_chain" / "fiat",
  amount_sats:            uint / null,
  amount_fiat_cents:      uint / null,
  currency:               tstr / null,
  payment_status:         "confirmed" / "pending" / "failed",
  processor_reference:    bstr(32) / null
}
```

**ReceiptEnvelope.type:** `"pqgw.enrollment_complete"`

```
EnrollmentCompleteBody = {
  user_binding:           bstr(32),
  wallet_init_hash:       bstr(32),
  epoch_bootstrap_hash:   bstr(32),
  policy_bundle_hash:     bstr(32),
  provider_count:         uint,
  enrollment_ticks:       uint,
  first_request_hash:     bstr(32) / null
}
```

**ReceiptEnvelope.type:** `"pqgw.wallet_init"`

```
WalletInitBody = {
  user_binding:           bstr(32),
  wallet_fingerprint:     bstr(32),
  key_class:              tstr,
  derivation_path_hash:   bstr(32)
}
```

**ReceiptEnvelope.type:** `"pqgw.epoch_bootstrap"`

```
EpochBootstrapBody = {
  user_binding:           bstr(32),
  pinned_profile_hash:    bstr(32),
  first_verified_tick:    uint,
  clock_source:           tstr
}
```

**ReceiptEnvelope.type:** `"pqgw.provider_registered"`

```
ProviderRegisteredBody = {
  provider_id:            tstr,
  provider_name:          tstr,
  provider_class:         "cloud_api" / "local_inference" / "federated",
  model_count:            uint,
  privacy_classification: "provider_visible" / "encrypted_inference" / "local_only",
  adapter_manifest_hash:  bstr(32),
  registration_status:    "active" / "pending_verification"
}
```

**ReceiptEnvelope.type:** `"pqgw.provider_verification"`

```
ProviderVerificationBody = {
  provider_id:            tstr,
  verification_type:      "scheduled" / "holder_initiated" / "anomaly_triggered",
  identity_verified:      bool,
  fingerprint_verified:   bool,
  adapter_hash_verified:  bool,
  previous_status:        tstr,
  resulting_status:       "active" / "suspended"
}
```

---

## 8. Product Refusal Codes

Product-layer refusal codes, registered via PQSEC Annex AE patterns. These are product refusals, not PQSEC predicate failures. They MUST NOT be confused with PQSEC refusal codes in rendering.

| Code | Component | Meaning |
|---|---|---|
| `E_PROVIDER_NOT_REGISTERED` | P1 / P6 | Requested provider or model not in registry snapshot |
| `E_PROVIDER_SUSPENDED` | P1 / P6 | Provider currently suspended pending verification |
| `E_PROVIDER_DISCOVERY_UNAVAILABLE` | P6 | Registry snapshot unavailable or expired |
| `E_ADAPTER_NOT_ADMITTED` | P2 | Provider adapter has not been admitted via Annex AX |
| `E_ADAPTER_BINARY_MISMATCH` | P2 | Adapter binary hash does not match manifest |
| `E_POLICY_NOT_COMPILED` | P3 | User has not completed policy compilation |
| `E_POLICY_NOT_REVIEWED` | P3 | Policy has not been rendered and reviewed via PQHR |
| `E_ENROLLMENT_INCOMPLETE` | P5 | User enrollment sequence not complete |
| `E_MODEL_TARGET_AMBIGUOUS` | P1 | Model target resolution failed |
| `E_PROVIDER_PRIVACY_MISMATCH` | P1 / P3 | Provider privacy classification not permitted by policy |
| `E_BILLING_QUOTA_EXCEEDED` | P4 | Usage exceeds billing period allocation |
| `E_BILLING_RATE_LIMITED` | P4 | Request rate exceeds tier or policy limit |

---

## 8A. Threat Model

PQ Gateway is the most externally-exposed component in the PQ ecosystem. It sits between holders and model providers, handles credentials, processes prompts, manages billing, and orchestrates enrollment. Adversaries may target it from both sides.

PQ Gateway assumes adversaries may:

* return malicious, fabricated, or manipulated responses from provider APIs
* compromise or substitute provider adapter binaries after admission
* attempt billing bypass via quota manipulation, tier spoofing, or rate limit circumvention
* attempt policy downgrade by manipulating policy authoring inputs or exploiting template defaults
* poison the Provider Registry Snapshot with fabricated provider entries or inflated evidence scores
* exploit the enrollment flow to phish credentials, intercept root secrets, or bypass onboarding steps
* exfiltrate derived credential material, raw API keys, or prompt content from gateway memory
* observe timing variations in routing, billing, or refusal paths to infer enforcement decisions or usage patterns
* replay stale GovernedRequestEnvelopes, InferenceReceipts, or ProviderRegistrySnapshots
* target the policy rendering surface to manipulate holder understanding of active policy (addressed by PQHR Annex A)

PQ Gateway does not assume trusted model providers, trusted cloud infrastructure, honest provider responses, or benign enrollment inputs.

### Trust Assumptions

PQ Gateway operates under the following trust assumptions:

* PQSEC is correctly implemented and is the sole enforcement authority
* Epoch Clock provides accurate, verifiable time
* PQSF deterministic CBOR encoding is implemented correctly
* The holder's root secret is generated and stored outside the gateway's persistent memory
* STP session keys are established before any sensitive data transit
* Provider adapter binaries are verified against manifest hashes at admission and on every load

---

## 8B. Security Considerations

### Threats Addressed

PQ Gateway addresses the following threats within its defined scope:

* **Malicious provider responses:** Canonical response mapping (P2.5) hashes raw response bytes and destroys originals beyond the operation boundary. Provider metadata is opaque-hashed. Tool calls are classified conservatively with unknown calls escalated to `authority` (PQAI §11).

* **Adapter supply chain compromise:** Provider adapters are manifest-bound and admitted via PQSEC Annex AX. Binary hash verification occurs at admission and on every load. Hash mismatch produces `E_ADAPTER_BINARY_MISMATCH` (Lockout Contributing). Scope expansion requires re-admission.

* **Billing bypass and quota manipulation:** Billing refusals are additive only (P4.2). Tier restrictions are expressed as PolicyObject constraints enforced by PQSEC where possible. Usage records are constant-shape to prevent timing and usage side channels. No side-channel "allow anyway" paths exist.

* **Policy downgrade attacks:** Policy compilation (P3.5) produces monotonically-versioned PolicyBundles. Previous versions are retained via PQPS with verifiable deletion semantics. Policy changes require PQHR rendering and holder review before activation (P3.6). No in-place mutation.

* **Registry poisoning:** The Provider Registry Snapshot (P6.3) is signed, time-bounded, and verified. Verification failure sets provider status to suspended. The Evidence Scorecard (P6.6) is advisory and explicitly prohibited from becoming a trust score or authority signal.

* **Credential exfiltration:** Raw API keys are destroyed from gateway memory immediately after derivation context is established (P2.6). Derived credentials follow PQSF Annex AI.3 emission prohibition. Only hash commitments are recorded. Provider connections are destroyed at the end of each attempt.

* **Prompt and response content leakage:** Raw prompt content must not be retained beyond the operation boundary (P2.4 rule 5). Raw response bytes must not be retained beyond the operation boundary (P2.5 rule 1). Usage records, notifications, and telemetry must not include prompt or response content.

* **Timing side channels:** Latency is recorded only as constant-shape buckets (P1.8.1), not precise values. Billing and refusal surfaces are constant-shape (P4.2, §10 rule 8). Telemetry envelopes are constant-shape (P4.6).

* **Replay attacks:** All artefacts are tick-bounded (`issued_tick`, `expiry_tick`). Requests with `current_tick >= expiry_tick` are refused. Sessions are STP-bound with key material destroyed at session end.

### Threats NOT Addressed (Out of Scope)

PQ Gateway does NOT protect against:

* **Model correctness or truthfulness:** PQ Gateway does not guarantee factual accuracy of provider responses. PQAI drift detection identifies behavioural change, not correctness.

* **Compromised PQSEC implementation:** If the enforcement engine is compromised, all gateway guarantees fail. PQSEC integrity is a foundational trust assumption.

* **Hardware-level attacks on gateway runtime:** Side-channel attacks against the gateway process itself (memory inspection, CPU cache timing) are out of scope. PQEA and hardware attestation (PQAA) provide complementary coverage where deployed.

* **Compromised Epoch Clock source:** If the Bitcoin-anchored time source is compromised, all time-bounded artefacts are unreliable. Epoch Clock integrity is a foundational trust assumption.

* **Social engineering of holders during enrollment:** PQ Gateway enforces structural sequencing during enrollment but cannot prevent holders from being socially engineered into providing credentials to adversaries outside the enrollment flow.

---

## 8C. Deployment Models and Holder Execution Boundary (Normative)

PQ Gateway can operate in different deployment configurations. The placement of gateway components relative to the Holder Execution Boundary (HEB) has security implications that implementers MUST understand.

### Model 1: Sovereign (Self-Hosted)

All gateway components (P1–P6) run inside the Holder Execution Boundary. The holder controls the execution environment, key material, and network connections.

* PQSEC runs inside HEB.
* Provider connections originate from inside HEB.
* Root secret never leaves HEB.
* Derived credentials never leave HEB.
* Prompt content never leaves HEB except to the selected provider over STP.

This is the strongest deployment model. All PQ Gateway guarantees apply without qualification.

### Model 2: Cloud-Hosted (Managed)

Gateway components run outside the Holder Execution Boundary on infrastructure operated by a gateway provider. The holder trusts the gateway operator with:

* Prompt content in transit through the gateway
* Derived credential material during operation lifetime
* Policy compilation and enforcement orchestration

In this model:

* PQSEC MUST still run as the sole enforcement authority. The gateway operator MUST NOT substitute or bypass PQSEC.
* Derived credentials MUST be scoped to the gateway operator's infrastructure and MUST NOT be extractable by the operator. Hardware-backed key storage (where available) SHOULD be used.
* The holder's root secret MUST NOT be transmitted to or stored by the gateway operator. Credential derivation MUST use the registered PQSF §13 context strings, ensuring the operator holds derived material only.
* Prompt content destruction rules (P2.4 rule 5, P2.5 rule 1) apply to the operator's infrastructure. The operator MUST NOT retain prompt or response content beyond the operation boundary.
* The holder SHOULD verify gateway operator attestation via PQAA (if available) before trusting the deployment.

### Model 3: Split (Router Outside, Enforcement Inside)

A hybrid model where the gateway router (P1) and adapter layer (P2) run outside the HEB, but PQSEC and policy storage run inside the HEB. EnforcementOutcome artefacts are produced inside the HEB and consumed by the external router.

* PQSEC runs inside HEB.
* Policy authoring and compilation run inside HEB.
* Router and adapters run outside HEB, consuming signed EnforcementOutcome artefacts.
* The router cannot forge or modify EnforcementOutcome artefacts (signature-bound).
* Prompt content passes through the external router — the same trust implications as Model 2 apply to prompt handling.

### Deployment Model Declaration

Implementations MUST declare which deployment model they implement. Holders MUST be informed of the deployment model via PQHR rendering before enrollment. The deployment model MUST NOT change without holder notification and re-consent.

---

## 9. Dependencies

| Specification | Minimum Version | Purpose |
|---|---|---|
| PQSF | ≥ 2.0.3 | Canonical encoding, STP, credential derivation, gateway adapter pattern, telemetry envelope |
| PQSEC | ≥ 2.0.3 | Sole enforcement authority |
| PQAI | ≥ 1.2.0 | SafePrompt, model identity, drift, tool governance, agent enrollment (Annex AA) |
| Epoch Clock | ≥ 2.1.0 | Time authority |
| PQHD | ≥ 1.2.0 | Wallet, custody, delegation constraints |
| PQHR | ≥ 1.0.0 | Policy rendering and holder inspection |
| PQPS | ≥ 1.0.0 | Persistent state and verifiable deletion |
| PQAA | ≥ 1.0.0 | Platform attestation bridging (optional) |

---

## 10. Conformance

PQ Gateway is conformant if:

1. PQSEC is the sole enforcement authority.
2. All time references are Epoch Clock ticks. No fallback clocks.
3. All artefacts are canonical CBOR (Epoch Clock artefacts remain JCS JSON per PQSF exception).
4. All sessions use STP. Exporter binding is enforced where required.
5. Provider adapters are manifest-bound, admitted, and fail-closed on update.
6. Policy is compiled to canonical PolicyBundle and rendered via PQHR before activation.
7. Every operation produces a signed receipt.
8. Refusal surfaces are constant-shape.
9. No product-layer component evaluates predicates or produces EnforcementOutcome artefacts.
10. No product-layer component creates an approval signal outside PQSEC.

---

## Migration Notes (Non-Normative)

**`sid` → `session_id` rename (v1.0.0):** Prior drafts of PQ Gateway used `sid` as the session identifier field. All artefacts now use `session_id: bstr(16)` for consistency with PQSF, PQSEC, and PQAI. Implementations MAY accept `sid` as an alias of `session_id` for a transitional period, but MUST emit `session_id` in all new artefacts.

---

If you find this work useful and wish to support continued development, donations are welcome:

**Bitcoin:**
bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw