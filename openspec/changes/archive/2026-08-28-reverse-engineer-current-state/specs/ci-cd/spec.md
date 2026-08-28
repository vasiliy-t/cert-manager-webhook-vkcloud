## Purpose

Defines the current CI/CD pipeline: how the Docker image is built, tagged, and published to a container registry, and what triggers the pipeline.

## ADDED Requirements

### Requirement: CI is triggered on push to master

The CI/CD pipeline SHALL run on every push to the `master` branch. Pull requests SHALL NOT trigger the pipeline (as of the current state).

#### Scenario: Push to master triggers build

- **WHEN** a commit is pushed to the `master` branch
- **THEN** the CI pipeline runs

#### Scenario: Push to a non-master branch does not trigger build

- **WHEN** a commit is pushed to a branch other than `master`
- **THEN** the CI pipeline does not run

### Requirement: Go version is pinned in CI

The CI pipeline SHALL use Go 1.17 to build the project. The Go version is declared in the workflow file, not in `go.mod` (which declares 1.13).

#### Scenario: CI uses Go 1.17

- **WHEN** the CI pipeline runs
- **THEN** the Go toolchain used is version 1.17

### Requirement: Docker image is built via Makefile

The CI pipeline SHALL invoke `make build` to build the Docker image. The Makefile's `build` target SHALL run `docker build` with the image name `cert-manager-webhook-vkcloud` and tag `latest`.

#### Scenario: Image is built with default tag

- **WHEN** `make build` is executed
- **THEN** a Docker image named `cert-manager-webhook-vkcloud:latest` is built

### Requirement: Image is published to GitHub Container Registry

The CI pipeline SHALL publish the built image to `ghcr.io/tarantool/cert-manager-webhook-vkcloud` with the tag `latest`. The pipeline SHALL log in to GHCR using the GitHub token provided by the workflow's runner.

#### Scenario: Image is pushed to ghcr.io

- **WHEN** the Docker image is built successfully
- **THEN** it is tagged as `ghcr.io/tarantool/cert-manager-webhook-vkcloud:latest` and pushed to GitHub Container Registry

### Requirement: No versioned tags are published

As of the current state, the pipeline SHALL publish only the `latest` tag. No git-tag-based or semantic-version tags SHALL be published.

#### Scenario: Only latest tag exists

- **WHEN** the pipeline completes successfully
- **THEN** the only tag pushed is `latest`

### Requirement: No linting or unit tests in CI

As of the current state, the CI pipeline SHALL NOT run linters (e.g., golangci-lint), `go vet`, or unit tests. The only quality gate is the Docker build itself.

#### Scenario: CI does not run lint

- **WHEN** the CI pipeline runs
- **THEN** no linter or `go vet` step is executed
