# OpenSpec — baseline specification

This directory holds the **baseline specification** for
`vasiliy-t/cert-manager-webhook-vkcloud`, reverse-engineered from the
current code on 2026-08-28.

It is the reference point for every future change (feature, fix,
refactor). Changes are proposed as **deltas** against these specs and
landed via the OpenSpec workflow:

```
openspec-propose "<change>"   -> creates openspec/changes/<change>/
openspec-apply-change         -> implements the tasks
openspec-archive-change       -> merges deltas into openspec/specs/
```

## Capabilities

| Capability              | What it covers                                             | Requirements |
|-------------------------|------------------------------------------------------------|--------------|
| `acme-solver`           | ACME DNS01 solver behavior (Present/CleanUp/Initialize)    | 7            |
| `vkcloud-dns-client`    | VK Cloud public-dns API v2 integration                     | 7            |
| `kubernetes-deployment` | Helm chart, RBAC, Service, APIService, PKI                 | 9            |
| `ci-cd`                 | GitHub Actions pipeline, Docker build, ghcr.io publishing  | 7            |

**Total: 30 requirements across 4 capabilities.**

## How to use

- **Read a capability:** `openspec/specs/<capability>/spec.md`
- **List all specs:** `openspec list --specs`
- **Validate a spec:** `openspec validate <capability>`
- **Propose a change:** use the `openspec-propose` skill (or `openspec new change "<name>"`)
- **View archived changes:** `openspec/changes/archive/`

## Provenance

The baseline was created by the change
`2026-08-28-reverse-engineer-current-state` (see
`openspec/changes/archive/2026-08-28-reverse-engineer-current-state/`).
That change's `design.md` documents the decisions made while
reverse-engineering the code into this spec.

## Ground rules for future changes

1. **Every change is a delta.** Use `## ADDED Requirements`,
   `## MODIFIED Requirements`, `## REMOVED Requirements`, or
   `## RENAMED Requirements` in the delta spec.
2. **No silent behavior changes.** If a change alters observable
   behavior, it MUST update the relevant spec.
3. **Implementation details stay out of specs.** Specs describe
   *what* the system does, not *how*. Function names, library
   choices, and internal data structures belong in `design.md` of
   the change, not in the main spec.
4. **Scenarios are testable.** Each scenario is a potential test
   case. If you can't test it, it probably doesn't belong in the spec.
