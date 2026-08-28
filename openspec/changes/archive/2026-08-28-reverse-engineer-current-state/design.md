## Context

This is a reverse-engineering change: the goal is to document the current state of the `vasiliy-t/cert-manager-webhook-vkcloud` codebase as a set of OpenSpec specifications. No code is modified. The design decisions here are therefore about **how to structure the specification**, not about how to implement features.

See proposal.md for the motivation.

The codebase is small (~150 lines of core logic in `vkcloud/client.go` + ~300 lines in `main.go`), a Go binary that implements the cert-manager ACME `Solver` interface, with a Helm chart for deployment and a single GitHub Actions workflow for CI/CD.

## Goals / Non-Goals

**Goals:**

- Capture the observable behavior of the system in a form that future changes can reference.
- Split the specification into four capabilities that map to natural seams in the code: the ACME solver contract, the VK Cloud DNS client, the Kubernetes deployment, and the CI/CD pipeline.
- Use the OpenSpec delta-spec format (`## ADDED Requirements`) so that archiving this change creates the main specs under `openspec/specs/`.
- Keep each requirement testable: every scenario is a potential test case.

**Non-Goals:**

- No code changes. No refactoring. No bug fixes. No new features.
- No changes to the Helm chart, CI workflow, Dockerfile, or Go source.
- No attempt to "improve" the specification beyond what the code actually does. If the code is wrong, the spec documents the wrong behavior; fixing it is a future change.
- No integration with external documentation (README, wiki). The spec is the source of truth for behavior; the README can be aligned later.

## Decisions

### Decision 1: Four capabilities, not one

**Choice:** Split the spec into `acme-solver`, `vkcloud-dns-client`, `kubernetes-deployment`, and `ci-cd`.

**Rationale:** These correspond to the natural seams in the code:
- `main.go` implements the ACME solver contract (Present/CleanUp/Initialize/Name).
- `vkcloud/client.go` implements the VK Cloud DNS API client.
- `deploy/` contains the Helm chart that wires the webhook into a cluster.
- `.github/workflows/main.yml` defines the CI/CD pipeline.

A single monolithic spec would be hard to navigate and would make future deltas noisy. Four capabilities let a future change touch only the relevant spec.

**Alternative considered:** A single `webhook` capability. Rejected because it would conflate the solver logic with the deployment and CI concerns, which change at different rates.

### Decision 2: Specs document behavior, not implementation

**Choice:** Each requirement describes observable behavior (inputs, outputs, error conditions) rather than internal function names or library choices.

**Rationale:** The OpenSpec spec format is a behavior contract. If the implementation can change without changing externally visible behavior, that detail does not belong in the spec. For example, the spec says "the client authenticates via gophercloud OpenStack credentials" (behavior) rather than "the client calls `openstack.AuthenticatedClient`" (implementation).

**Exception:** The `vkcloud-dns-client` spec includes the hardcoded endpoint URL and the API paths, because those are part of the external contract with the VK Cloud service — changing them would change observable behavior (which server is called, with which paths).

### Decision 3: All requirements use `## ADDED Requirements`

**Choice:** Since this is the initial baseline (no existing specs), every requirement is marked as `ADDED`.

**Rationale:** The OpenSpec delta format uses `ADDED`/`MODIFIED`/`REMOVED`/`RENAMED` to express changes relative to the main spec. For a baseline, everything is new, so everything is `ADDED`. When this change is archived, the archive step copies these into `openspec/specs/<capability>/spec.md` as the main spec.

### Decision 4: Scenarios cover the happy path and the main error paths

**Choice:** Each requirement has at least one scenario. Most have two or more, covering the happy path and the most important error conditions.

**Rationale:** Scenarios are the testable units. A requirement without a scenario is unverifiable. The goal is not exhaustive test coverage in the spec (that belongs in the test suite), but enough scenarios that a future change can reason about "does this break any existing scenario?"

## Risks / Trade-offs

### Risk: The spec diverges from actual behavior

**Mitigation:** The spec is derived from a close reading of the current code. Any discrepancy discovered during later changes will be captured as a delta (MODIFIED or REMOVED requirement). The spec is a living document, not a frozen contract.

### Risk: Over-specification

**Mitigation:** The spec intentionally omits implementation details (function names, library choices, internal data structures) except where they are part of the external contract (API endpoints, configuration field names). If a future change wants to add implementation guidance, that belongs in `design.md` of the change, not in the main spec.

### Trade-off: Four capabilities vs. one

The four-capability split adds navigational overhead (four files instead of one) but keeps future deltas focused. For a project of this size, the trade-off favors the split.
