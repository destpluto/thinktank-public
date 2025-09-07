
# Open Anti-Cheat (OAC) v0.1-DRAFT — Pluggable Specification

## 0) Goals & Non-Goals

**Goals:** Server-auth play; minimal client overhead; modular checks; strong evidence bundles for appeals; opt-in stronger security (attestation) for ranked.  

**Non-Goals:** Kernel drivers, invasive file scans, always-on packet crypto, spyware.

---

## 1) Threat Model (summary)

- **Detected well:** user-mode hooks, DLL injectors, packet tampering, post-TLS mutation, logic aimbots.  
- **Pressured/flagged:** early-load/kernel/hypervisor & DMA/HID spoofers (via attestation + behavior analytics).  
- **Out of scope:** physical collusion (screen peeking), hardware mods that perfectly emulate human motion (statistical pressure still applies).

---

## 2) Profiles (policy presets)

- **CASUAL:** Core integrity + light anomaly checks; no attestation; low challenge rate.  
- **RANKED:** Core + anomaly + transport checks; adaptive challenge rate; optional attestation rewarded.  
- **COMPETITIVE+:** All of the above + REQUIRED platform attestation; tighter thresholds; heavier auditing.

---

## 3) Architecture (pluggable services)

**Client**
- OAC Runtime (thin)  
  - Telemetry Collector (module)  
  - Challenge Executor (module)  
  - Attestation Provider (module, optional)  
- Game Adapter (SDK)

**Server**
- Policy Engine  
- VRF Sampler (module)  
- Challenge Orchestrator (module)  
- Transport Verifier (module)  
- Anomaly Detector (module)  
- Trust Scorer (module)  
- Evidence Store

---

## 4) Module Interfaces (minimum contracts)

### 4.1 Telemetry Collector (client)
- **Input:** Raw HID deltas at poll rate, timestamps; shot events; ADS state.  
- **Output:** Rolling stats (percentiles, MAD), 1–2s ring-buffer of raw deltas for on-demand challenge.  
- **Privacy:** No screenshots, no process lists, no file scans.  
- **Config:** Polling normalization (DPI, Hz).

### 4.2 Challenge Orchestrator (server)
- **Schedule:** Jittered by VRF; mean 60–180s (profile-dependent); per-player adaptive.  
- **Control Message:** challenge_id, type[], nonce, window_ms, params, sig_server.  
- **Signature:** Ed25519 (REQUIRED). Wrapping in OpenPGP is OPTIONAL for ops workflows.

### 4.3 Challenge Executor (client)
Executes types:
- **EXE_MEASURE** → code-section hashes + build id.  
- **NET_WINDOW** → HMAC over outbound payloads for N ms + OS send metadata.  
- **INPUT_SLICE** → last 500–750ms raw deltas around server-selected shots.  

**Response:** Includes challenge_id, nonce, results[], sig_client.

### 4.4 Transport Verifier (server)
- Verifies: HMAC correctness, sequence bounds, OS send-timestamp coherence. Flags post-client mutation.

### 4.5 Attestation Provider (client, optional/required by profile)
- **Windows:** TPM quote + Secure Boot + VBS/HVCI + WDAC policy digest.  
- **macOS:** Code signing + hardened runtime indicators.  
- **Linux:** IMA/EVM measurement list + Secure Boot indicator if present.  

**Output:** Signed attestation blob + verifier report (server).

### 4.6 Anomaly Detector (server)
- **Input:** Rolling stats + occasional INPUT_SLICE windows + server context (enemy enters FOV).  
- **Output:** anomaly_score ∈ [0,1], reasons, counters.  
- **Pluggable:** Start with rules, allow ML later under same interface.

### 4.7 Trust Scorer (server)
- **Inputs:** Attestation (tier), challenge pass-rate, transport verifier status, anomaly score history, prior sanctions.  
- **Output:** trust ∈ [0,100], tier, action hints (increase challenge rate, queue limits, etc).

### 4.8 Evidence Store (server)
- **Bundle:** Signed challenges/responses, attestation verdicts, anomaly timelines, resim comparisons, final action.  
- **Export:** One-click package for appeals (JSON + sigs).

---

## 5) Minimal Wire Messages (normative)

### 5.1 ChallengeControl (server→client)
```json
{
  "v": 1,
  "session_id": "uuid",
  "challenge_id": "uuid",
  "types": ["EXE_MEASURE","NET_WINDOW","INPUT_SLICE"],
  "window_ms": 250,
  "nonce": "base64",
  "params": { "seq_first_hint": 12345 },
  "sig_server_ed25519": "base64"
}
```
### 5.2 ChallengeResponse (client→server)
```json
{
  "v": 1,
  "session_id": "uuid",
  "challenge_id": "uuid",
  "nonce": "base64",
  "exe_measure": { ... },
  "net_window": { ... },
  "input_slice": { ... },
  "sig_client_ed25519": "base64"
}
```

**REQUIRED:** Reject if signature invalid, id/nonce mismatch, or late beyond policy window.

* * *

## 6) Baseline Heuristics (rules-based, tunable)
---------------------------------------------

**Tracked client stats (EWMA on robust percentiles):**

*   P10\_scan\_ms, MAD\_scan\_ms (non-ADS scan micro-adjusts)
*   P10\_prefire\_ms (latency from final snap to shot)
*   snap\_fraction\_50ms (fraction of shots preceded by <50ms snap)
*   headshot\_rate\_anom (headshots within anomalous windows)

**Recommended thresholds (v0.1 defaults):**

*   T\_scan\_low = 200 ms
*   T\_prefire\_low = 80 ms
*   T\_snap = 50 ms

**Anomaly Score:**

```
A1 = clamp((T_scan_low   - P10_scan_ms)   / T_scan_low, 0, 1)
A2 = clamp((T_prefire_low- P10_prefire_ms)/ T_prefire_low, 0, 1)
A3 = snap_fraction_50ms
A4 = headshot_rate_anom

Score = 0.2*A1 + 0.4*A2 + 0.3*A3 + 0.1*A4
```

**Escalate only if** Score > 0.8 for ≥3 consecutive minutes (server time).  
**Context guards:** exclude ADS tracking segments; normalize for DPI/polling; align with “enemy enters FOV” server tags.

* * *

### 6.1 AI-Assisted Anomaly Analysis (extension, optional module)
-------------------------------------------------------------

The baseline rules provide fast, explainable heuristics. AI is layered on **post-match** to reduce false negatives and detect cheats tuned to sit just inside human thresholds.

### Data for AI (anonymized)

*   Per-poll HID deltas (resampled & normalized)
*   Server context (target entry events, velocities)
*   Derived stats (latency distributions, curvature, jitter entropy)
*   No raw screen/process/file capture

### Feature families

*   **Micro (per-shot windows):**
    *   Pre-fire latency distribution (esp. <50ms tail)
    *   Path curvature & jerk before shots
    *   Entropy of micro-jitter vs. precision at click
    *   Speed–accuracy slope (Fitts-like law)
*   **Macro (session-level):**
    *   Distribution tails (P05/P10 of pre-fire)
    *   Consistency metrics across match
    *   RNG sanity (interval variance, periodicity tests)

### Model approach

*   Offline training with known-honest and known-cheat corpora.
*   Gradient boosting (XGBoost/LightGBM) or conformal prediction for low false positives.
*   Personalized baselines: Gaussian/quantile models per player.
*   Active learning from appeals & reports.

### Integration

*   Exposes same interface as rules-based anomaly module (`score, reasons, counters`).
*   Used for **post-match escalation** and evidence bundling, not live input gating.
*   Explanation cards: “42% of shots had <50ms snaps (baseline 4%)” etc.

* * *

## 7) Policy & Enforcement (graduated, evidence-first)
---------------------------------------------------

*   Adaptive challenges (↑ frequency) on suspicion.
*   Probational ranked queue (reduced MMR impact) while collecting more signal.
*   Temporary competitive lockouts with evidence bundle available in client UI.
*   Bans only on persistent, multi-signal proof across sessions (challenge fails AND anomaly persistence AND transport anomalies). Always export evidence.

* * *

## 8) Privacy & Security Posture
-----------------------------

*   Client MUST NOT enumerate processes, scan disks, capture screens, or read personal files.
*   All challenge contents and responses are transparent (documented JSON); signatures are verifiable.
*   Telemetry kept minimal and ephemeral; only evidence bundles persist server-side per policy.
*   Attestation blobs contain only platform measurements/signatures—no PII.

* * *

## 9) Performance Budget (targets)
-------------------------------

*   CPU: telemetry <0.2% avg; challenge bursts <2% for <300ms.
*   Bandwidth: <4 KB per challenge average.
*   Latency: zero input throttling; no per-packet crypto.
*   Hitching: challenges executed off main thread; drop/defers allowed during frame spikes (with server notice).

* * *

## 10) Extensibility (versioning & registry)
-----------------------------------------

*   **Module IDs:** oac.anomaly.rules.v1, oac.anomaly.ml.v1, oac.attest.win.v1, etc.
*   **Semantic versions:** MAJOR changes require negotiation; MINOR are backward-compatible fields.
*   **Public registry:** JSON manifests for modules (inputs, outputs, config knobs, license).

* * *

## 11) Developer Integration (SDK)
-------------------------------

*   **Client SDK:** C/C++ API (Rust FFI optional) with:
    *   oac\_init(cfg), oac\_tick(inputs), oac\_on\_challenge(msg) → response
    *   Ring-buffer helper for INPUT\_SLICE
*   **Server SDK:** gRPC/HTTP handlers:
    *   SubmitResponse, GetTrust, ExportEvidence
    *   Plug-points for custom anomaly/attestation backends.

* * *

## 12) Test & Evaluation
---------------------

*   False-positive target: < 1e-5 per competitive hour.
*   Cheat corpora: replay suites (DLL hooks, packet mutators, logic aimbots, DMA/HID).
*   A/B in live pilots: measure ban appeal overturn rates (<2% target) and time-to-detection.
*   Red-team bounty: publish challenge schemas; reward bypass reports.

* * *

## 13) Roadmap (suggested)
-----------------------

*   **v0.1 (this):** rules-based anomaly + integrity + transport checks.
*   **v0.2:** optional TPM/IMA attestation provider; trust tiers.
*   **v0.3:** ML anomaly module spec (same interface).
*   **v1.0:** reference implementations + compliance tests; indie-friendly PoC.

* * *

### Appendix A — Reference Defaults (tunable)

*   Challenge cadence: mean 120s (Casual), 90s (Ranked), 60s (Competitive+). Poisson-jittered via VRF.
*   INPUT\_SLICE window: 500–750 ms; sample random ~5% of shots (Ranked+).
*   NET\_WINDOW length: 200–300 ms; HMAC key = HKDF(exporter\_secret, nonce); AEAD optional.
*   Hashes: sha256 for code sections; Ed25519 for signatures.
*   Evidence retention: 30 days (Casual), 90 days (Ranked), 180 days (Competitive+).

* * *

### Appendix B — Evidence Bundle (export)

```
bundle/
  meta.json
  challenges.ndjson
  responses.ndjson
  anomaly_timeline.json
  resim/          # divergence proofs
  attest/         # quotes + verifier report
  signatures/     # detached sigs
```

* * *
