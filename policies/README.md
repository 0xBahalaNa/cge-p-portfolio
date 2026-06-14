# Compliance Policy Suite (Lab 3.3)

Rego policies that evaluate `terraform plan -json` output before apply. Each policy maps to a NIST 800-53 control and returns structured deny messages with the resource address and remediation guidance.

## Policies

| Policy | Control | Severity | Enforces | Remediation |
|---|---|---|---|---|
| `sc28_encryption.rego` | SC-28 | high | Every `google_storage_bucket` has a non-empty `encryption { default_kms_key_name }` block (CMEK). | Add an `encryption { default_kms_key_name = ... }` block referencing a `google_kms_crypto_key` you control. |
| `ac3_no_public.rego` | AC-3 | critical | GCS buckets enforce `uniform_bucket_level_access=true` and `public_access_prevention="enforced"`. Firewall rules must not expose management ports (22, 3389) to `0.0.0.0/0` or `*`. | Set `uniform_bucket_level_access = true`, `public_access_prevention = enforced`. For firewalls, narrow `source_ranges` or remove the rule. |
| `cm6_required_tags.rego` | CM-6 | medium | Every taggable resource carries the four required labels: `project`, `environment`, `managed_by`, `compliance_scope`. | Add the four required labels to the resource. |

## Usage

```bash
# Unit tests (inline fixtures in policies/tests/)
opa test -v policies/

# Integration eval against a plan JSON
opa eval -d policies -i plan.json data.compliance.sc28.deny --format=pretty
opa eval -d policies -i plan.json data.compliance.ac3.deny  --format=pretty
opa eval -d policies -i plan.json data.compliance.cm6.deny  --format=pretty
```
