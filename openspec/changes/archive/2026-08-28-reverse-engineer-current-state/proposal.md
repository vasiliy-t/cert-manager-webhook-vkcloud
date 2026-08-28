## Why

The `vasiliy-t/cert-manager-webhook-vkcloud` repository is currently a working codebase with no formal specification. It is a fork of `tarantool/cert-manager-webhook-vkcloud` that has diverged slightly. As the project owner taking over maintenance, the first step is to establish a baseline: capture the **current state** of the system as a set of OpenSpec specifications. This gives us:

- A reference point for any future change (feature, fix, refactor)
- A shared language for discussing requirements
- A foundation for the OpenSpec workflow (propose → apply → verify → archive)

Without this baseline, every future change is made against an implicit, undocumented understanding. With it, we can reason about what changes, what stays the same, and what is broken.

## What Changes

This is a **documentation/specification change**, not a code change. No code, configuration, or CI will be modified by this change itself. The deliverable is a set of OpenSpec specification files that capture:

- The system's purpose and scope
- The ACME DNS01 solver behavior (Present, CleanUp, Initialize)
- The VK Cloud DNS API integration contract
- The Kubernetes deployment model
- The current CI/CD pipeline
- The Helm chart packaging

After this change is applied, the `openspec/specs/` directory will contain the baseline specification, and subsequent changes (quality improvements, feature additions, bug fixes) will be made as deltas against it.

## Capabilities

### New Capabilities

- `acme-solver`: The ACME DNS01 solver behavior — how the webhook presents and cleans up ACME challenges, including the `Present`, `CleanUp`, and `Initialize` contracts, the `vkcloud` solver name, and the configuration surface (the five `SecretKeySelector` fields).

- `vkcloud-dns-client`: The VK Cloud public-dns API v2 integration — the `Client` struct, `GetZone`, `CreateRecord`, `FindRecordByContent`, `DeleteRecord`, the hardcoded endpoint, the gophercloud-based authentication, and the error types (`ZoneNotFoundError`, `RecordNotFoundErr`).

- `kubernetes-deployment`: The Kubernetes deployment model — the Helm chart, the RBAC, the webhook service, the APIService registration, the PKI/cert setup, and how the webhook is invoked by cert-manager.

- `ci-cd`: The current CI/CD pipeline — the GitHub Actions workflow, the Docker image build, the image registry (ghcr.io), the tagging strategy, and the trigger model.

### Modified Capabilities

(None — this is the initial baseline; no existing specs are being modified.)

## Impact

- **Affected code:** None. This change adds specification files only.
- **Affected APIs:** None. The specification documents existing behavior; it does not change it.
- **Dependencies:** None. No new Go modules, no new tools.
- **Systems:** The `openspec/specs/` directory of this workspace. Future changes will reference these specs as their baseline.
- **Risk:** Low. The main risk is that the reverse-engineered spec diverges from actual behavior. Mitigation: the spec is derived from a close reading of the code, and any discrepancy discovered during later changes will be captured as a delta.
