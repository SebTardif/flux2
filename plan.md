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
  --interval=1m \
  --export
```

And a corresponding golden file verifying the generated `spec.decryption` block. Added
`--interval=1m` explicitly to avoid the `resetCmdArgs()` Cobra flag-state pollution
where a prior test zeroes out the shared `createArgs.interval`.

**Files:**
- `cmd/flux/create_kustomization_test.go`
- `cmd/flux/testdata/create_kustomization/with-sops-decryption.yaml`

### 4. Integration test: `flux build kustomization` with SOPS-encrypted HelmRelease

Added two test cases to `TestBuildLocalKustomization` that run the full `Builder.Build()`
pipeline and verify SOPS metadata is stripped from the output:

- `build helmrelease with sops metadata` — builds a Kustomization directory containing a
  HelmRelease with a `.sops` block; asserts the `.sops` field is absent and the
  `ENC[…]` values are preserved.
- `build configmap with sops metadata` — same for a SOPS-encrypted ConfigMap, closing
  the ConfigMap test-coverage gap identified in the plan.

**Files:**
- `cmd/flux/build_kustomization_test.go`
- `cmd/flux/testdata/build-kustomization/sops-helmrelease/kustomization.yaml`
- `cmd/flux/testdata/build-kustomization/sops-helmrelease/helmrelease.yaml`
- `cmd/flux/testdata/build-kustomization/sops-helmrelease-result.yaml`
- `cmd/flux/testdata/build-kustomization/sops-configmap/kustomization.yaml`
- `cmd/flux/testdata/build-kustomization/sops-configmap/configmap.yaml`
- `cmd/flux/testdata/build-kustomization/sops-configmap-result.yaml`

---

## What Still Needs to Be Done

### Medium priority

- [ ] **Consider masking encrypted field values for non-Secret resources**
  Currently the `ENC[…]` ciphertext values in a HelmRelease are left in the output.
  This is intentional (ciphertext ≠ plaintext), but some teams may prefer all SOPS
  material to be redacted. A future change could replace `ENC[…]` values with
  `**SOPS**` for non-Secret resources as well.

### Low priority

- [ ] **Update `flux build kustomization` command documentation / examples** to mention
  that SOPS-encrypted HelmRelease resources are handled safely.

- [ ] **Integration test** (cloud e2e in `tests/integration/`) that provisions a real
  cluster with a SOPS-encrypted HelmRelease and verifies that `flux build kustomization`
  and `flux diff kustomization` both succeed.
