# Plan: SOPS Support for Non-Secret Resources in `flux build/diff kustomization`

## Problem

`flux build kustomization` and `flux diff kustomization` call `maskSopsData` on every
resource in the built output before printing it. Before this work, `maskSopsData` only
handled `Secret` resources — it would replace encrypted values with `**SOPS**` or strip
the `.sops` metadata block.

For non-Secret resources that are encrypted with SOPS (e.g. a `HelmRelease` whose
`spec.values` block is encrypted), the `.sops` top-level metadata block was left in
place. This caused two problems:

1. **Schema violation at apply time:** The `.sops` field is not part of the HelmRelease
   CRD schema. Leaving it in the built output means `flux diff kustomization` (which does
   a server-side apply dry-run) fails with a validation error.
2. **Unnecessary exposure:** SOPS metadata (key fingerprints, recipients, encrypted MAC)
   should not appear in CLI output.

A related gap: there was no unit test and no golden-file test covering the
`--decryption-provider` / `--decryption-secret` flags on `create kustomization`.

---

## What Has Been Done

### 1. Extend `maskSopsData` for non-Secret resources (`internal/build/build.go`)

Added an `else` branch to `maskSopsData` that handles every resource whose `Kind` is
not `Secret`. When a SOPS `.sops` block with an encrypted MAC (`mac: ENC[…]`) is
detected, it is stripped via `yaml.FieldClearer`. The encrypted field values themselves
(e.g. `ENC[AES256_GCM,…]` ciphertext) are intentionally left intact — they are already
opaque ciphertext, not plaintext, so there is nothing to redact.

**File:** `internal/build/build.go` — `maskSopsData` function (lines ~745–758)

### 2. Add `TestMaskSopsDataNonSecret` unit test (`internal/build/build_test.go`)

Added `TestMaskSopsDataNonSecret` with two table-driven cases:
- `HelmRelease with sops metadata` — verifies that the `.sops` block is stripped and
  encrypted values are preserved.
- `HelmRelease without sops metadata` — verifies that a resource without SOPS metadata
  passes through unchanged.

Also fixed a pre-existing broken duplicate test loop that had been left behind by a
prior partial edit.

**File:** `internal/build/build_test.go`

### 3. Golden-file test for `create kustomization` with decryption flags (`cmd/flux/`)

Added a `cmdTestCase` entry to `TestCreateKustomization` that exercises:

```
flux create kustomization mysql \
  --source=GitRepository/apps \
  --path=./apps \
  --decryption-provider=sops \
  --decryption-secret=sops-age \
  --namespace=flux-system \
  --export
```

And a corresponding golden file verifying the generated `spec.decryption` block.

**Files:**
- `cmd/flux/create_kustomization_test.go`
- `cmd/flux/testdata/create_kustomization/with-sops-decryption.yaml`

---

## What Still Needs to Be Done

### High priority

- [ ] **End-to-end / build-kustomization test with a SOPS-encrypted HelmRelease**
  Add a test case in `cmd/flux/build_kustomization_test.go` + testdata fixture (a fake
  Kustomization + HelmRelease with a `.sops` block) that runs through
  `Builder.Build()` and asserts the `.sops` field is absent from the output. This tests
  the full build pipeline rather than just the `maskSopsData` unit in isolation.

- [ ] **Verify `flux diff kustomization` path**
  `diff.go` calls the same `Builder` so benefits from the same fix; however, it uses a
  server-side apply dry-run. A unit or e2e test confirming that a SOPS-encrypted
  HelmRelease no longer triggers a schema error during diff would close this gap.

### Medium priority

- [ ] **Consider masking encrypted field values for non-Secret resources**
  Currently the `ENC[…]` ciphertext values in a HelmRelease are left in the output.
  This is intentional (ciphertext ≠ plaintext), but some teams may prefer all SOPS
  material to be redacted. A future change could replace `ENC[…]` values with
  `**SOPS**` for non-Secret resources as well.

- [ ] **ConfigMap support**
  ConfigMaps can also be SOPS-encrypted via kustomize-controller. The new `else` branch
  already covers them (it applies to all non-Secret kinds), but a dedicated test case
  for ConfigMap would improve confidence.

### Low priority

- [ ] **Update `flux build kustomization` command documentation / examples** to mention
  that SOPS-encrypted HelmRelease resources are handled safely.

- [ ] **Integration test** (cloud e2e in `tests/integration/`) that provisions a real
  cluster with a SOPS-encrypted HelmRelease and verifies that `flux build kustomization`
  and `flux diff kustomization` both succeed.
