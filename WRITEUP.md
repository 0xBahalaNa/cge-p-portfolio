# Lab 4.4 — Evidence Chain of Custody

Every pull request against `main` now leaves a **signed, timestamped, immutable**
record of what the `grc-gate` pipeline tested and what happened. An auditor
reconstructs and verifies that record with a single command —
`scripts/verify-evidence.sh <run_id>` — and needs zero trust in the operator.

## Architecture

```
PR → grc-gate run → bundle evidence/ → cosign sign-blob (keyless, GitHub OIDC)
   → Fulcio issues a short-lived cert · Rekor logs the signature + timestamp
   → aws s3 cp bundle + .sha256 + .sig.bundle + receipt.json → s3://<vault>/runs/<run_id>/
   → Object Lock applies retention
Auditor → verify-evidence.sh <run_id>
   → integrity (SHA-256) + authenticity/timestamp (cosign) + preservation (retention)
   → CHAIN INTACT
```

## Reference run

- **Run ID:** `29622061513` (PR #3 — `grc-gate` green, including *Install Cosign* and *Bundle + sign + upload*).
- **`verify-evidence.sh 29622061513`** → `CHAIN INTACT` (integrity matched, `cosign verify-blob` → `Verified OK`, Object Lock retention in force).
- **Tamper test:** appending one byte flipped the integrity check to `FAIL: SHA mismatch` (exit 1). The Object-Lock vault copy stayed pristine, so a re-verify still returns `CHAIN INTACT` — the tampered artifact only ever lives on the laptop.

## Chain-of-custody property → proving artifact

| Property | Proving artifact | Verified by |
|--------------|------------------|-------------|
| **Authenticity** | `*.sig.bundle` — the Cosign signature + Fulcio short-lived cert carrying the GitHub OIDC subject (`repo:0xBahalaNa/cge-p-portfolio:…`) | `cosign verify-blob` (check 2): the cert chains to Fulcio and the OIDC issuer matches → proves *who* produced it |
| **Integrity** | `*.tar.gz.sha256` — the SHA-256 digest captured at sign time | check 1: recompute the bundle's SHA-256 and compare to the sidecar (`FAIL: SHA mismatch` on any change) → proves the bytes are *unchanged* |
| **Timeliness** | the Rekor transparency-log entry packed inside `*.sig.bundle` — a public, append-only timestamp of the signing event | `cosign verify-blob` (check 2): confirms the Rekor entry exists and matches → proves *when* it was made |
| **Preservation** | the S3 Object Lock retention on the vaulted object under `runs/<run_id>/` | check 3: `get-object-retention` confirms `RetainUntilDate` is still in the future → proves it *still exists, intact and undeletable* |

> The `.sig.bundle` legitimately proves **two** properties — **authenticity** (the Fulcio cert) and **timeliness** (the Rekor entry) — both validated in the single `cosign verify-blob` call. The `.sha256` sidecar and the Object Lock retention each map to exactly one property. The `.tar.gz` bundle itself is the *subject* being proven, not a proof.

## Controls (derived — the lab checklist names none)

- **AU-10 (non-repudiation)** — the keyless Cosign signature binds the bundle to the GitHub OIDC identity (`repo:0xBahalaNa/cge-p-portfolio:...`); no operator can forge it, because the signing identity is minted by Sigstore, not held in the AWS account.
- **AU-9 (protection of audit information)** — S3 Object Lock makes the vaulted bundle immutable.
- **AU-11 (audit record retention)** — Object Lock default retention keeps the evidence for its retention window.
- **SI-7 (software/information integrity)** — the SHA-256 sidecar plus verify-time recompute detects any modification.

**FedRAMP High / CJIS delta:** same pattern; CJIS v6.0 evidence for CJI would use `COMPLIANCE`-mode Object Lock (not `GOVERNANCE`) and agency-managed keys, versus this lab's `GOVERNANCE`/AES256 baseline.
