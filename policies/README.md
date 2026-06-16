# Compliance Policy Suite (Lab 3.3 + Lab 3.4)

Rego policies that evaluate `terraform plan -json` output before apply. Each policy maps to a NIST 800-53 control and returns structured deny messages with the resource address and remediation guidance.

## Policies

| Policy | Cloud | Control | Severity | Enforces | Remediation |
|---|---|---|---|---|---|
| `sc28_encryption.rego` | GCP | SC-28 | high | Every `google_storage_bucket` has a non-empty `encryption { default_kms_key_name }` block (CMEK). | Add an `encryption { default_kms_key_name = ... }` block referencing a `google_kms_crypto_key` you control. |
| `sc28_encryption_aws.rego` | AWS | SC-28 | high | Every `aws_s3_bucket` has an `aws_s3_bucket_server_side_encryption_configuration` referencing it. | Add `aws_s3_bucket_server_side_encryption_configuration { bucket = aws_s3_bucket.<name>.id ... }` for the bucket. |
| `ac3_no_public.rego` | GCP | AC-3 | critical | GCS buckets enforce `uniform_bucket_level_access=true` and `public_access_prevention="enforced"`. Firewall rules must not expose management ports (22, 3389) to `0.0.0.0/0` or `*`. | Set `uniform_bucket_level_access = true`, `public_access_prevention = enforced`. For firewalls, narrow `source_ranges` or remove the rule. |
| `ac3_no_public_aws.rego` | AWS | AC-3 | critical | Every `aws_s3_bucket` has an `aws_s3_bucket_public_access_block` referencing it, with all four flags true. | Add a public access block with `block_public_acls`, `block_public_policy`, `ignore_public_acls`, and `restrict_public_buckets` all set to `true`. |
| `cm6_required_tags.rego` | GCP | CM-6 | medium | Every taggable resource carries the four required labels: `project`, `environment`, `managed_by`, `compliance_scope`. | Add the four required labels to the resource. |
| `cm6_required_tags_aws.rego` | AWS | CM-6 | medium | Every taggable AWS resource carries the four required tags: `Project`, `Environment`, `ManagedBy`, `ComplianceScope`. | Add the missing tags or use provider `default_tags`. |

## Usage

```bash
# Unit tests (inline fixtures in policies/tests/)
opa test -v policies/

# Integration eval against a plan JSON
opa eval -d policies -i plan.json data.compliance.sc28.deny --format=pretty
opa eval -d policies -i plan.json data.compliance.ac3.deny  --format=pretty
opa eval -d policies -i plan.json data.compliance.cm6.deny  --format=pretty

# Conftest gate (Lab 3.4)
./scripts/policy-gate.sh --workspace terraform/primitives/compliant-s3
```
