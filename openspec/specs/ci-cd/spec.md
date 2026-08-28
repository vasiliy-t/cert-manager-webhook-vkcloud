# ci-cd Specification

## Purpose
Defines the current CI/CD pipeline: how the Docker image is built, tagged, and published to a container registry, and what triggers the pipeline.

## Requirements

### Requirement: CI is triggered on push to master

The CI/CD pipeline SHALL run on every push to the `master` branch. Pull requests SHALL NOT trigger the pipeline (as of the current state).

#### Scenario: Push to master triggers build

- **WHEN** a commit is pushed to the `master` branch
- **THEN** the CI pipeline runs

#### Scenario: Push to a non-master branch does not trigger build

- **WHEN** a commit is pushed to a branch other than `master`
- **THEN** the CI pipeline does not run

### Requirement: Go toolchain versions are inconsistent across the project

The project SHALL exhibit the following (inconsistent) Go version declarations, documented here as the current state:

- The CI workflow installs Go 1.17 via `actions/setup-go`, but this toolchain is NOT used to build the shipped artifact — the build happens inside `docker build`
- The Dockerfile builds the binary with the `golang:1.16-alpine` image, so the shipped binary is compiled with Go 1.16
- `go.mod` declares `go 1.13`

Unifying these versions is a known improvement candidate, not part of the current state.

#### Scenario: Shipped binary is built with Go 1.16

- **WHEN** the CI pipeline builds the Docker image
- **THEN** the binary inside the image is compiled by the `golang:1.16-alpine` toolchain, regardless of the Go version installed in the CI runner

#### Scenario: CI-installed Go toolchain is unused

- **WHEN** the CI pipeline runs
- **THEN** the Go 1.17 toolchain installed by `actions/setup-go` is not invoked by any subsequent step (the only build step is `make build`, which delegates to `docker build`)

### Requirement: Docker image is built via Makefile

The CI pipeline SHALL invoke `make build` to build the Docker image. The Makefile's `build` target SHALL run `docker build` with the image name `cert-manager-webhook-vkcloud` and tag `latest`.

#### Scenario: Image is built with default tag

- **WHEN** `make build` is executed
- **THEN** a Docker image named `cert-manager-webhook-vkcloud:latest` is built

### Requirement: Image is published to GitHub Container Registry

The CI pipeline SHALL publish the built image to `ghcr.io/tarantool/cert-manager-webhook-vkcloud` with the tag `latest`. The pipeline SHALL log in to GHCR using the GitHub token provided by the workflow's runner.

The target registry namespace (`ghcr.io/tarantool/`) is hardcoded in the workflow. In a fork (such as `vasiliy-t/cert-manager-webhook-vkcloud`), the workflow's `GITHUB_TOKEN` has no permission to push to the `tarantool` namespace.

#### Scenario: Image is pushed to ghcr.io

- **WHEN** the Docker image is built successfully in the upstream `tarantool` repository
- **THEN** it is tagged as `ghcr.io/tarantool/cert-manager-webhook-vkcloud:latest` and pushed to GitHub Container Registry

#### Scenario: Push fails in a fork

- **WHEN** the CI pipeline runs on a push to `master` in a fork outside the `tarantool` organization
- **THEN** the publish step fails with a permission error, because the fork's `GITHUB_TOKEN` cannot push to `ghcr.io/tarantool/`

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

### Requirement: Conformance test suite exists but is not run in CI

The project SHALL contain a cert-manager DNS01 conformance test (`main_test.go`) runnable via `make test`. The test:

- Downloads kubebuilder 2.3.2 binaries into `_test/kubebuilder/` on first run
- Uses the manifest fixture at `testdata/vkcloud-solver/` (config snippet plus secret manifest)
- Is hardcoded to the resolved zone `example.com.` and the DNS server `ns2.mcs.mail.ru:53` (the `TEST_ZONE_NAME` environment variable is read but not applied to the fixture)
- Requires live VK Cloud credentials to pass

The CI pipeline SHALL NOT run this test (as of the current state).

#### Scenario: Conformance test runs locally

- **WHEN** a developer runs `make test` with valid VK Cloud credentials in the fixture
- **THEN** the cert-manager DNS01 conformance suite runs against the live VK Cloud DNS API

#### Scenario: CI does not run the conformance test

- **WHEN** the CI pipeline runs
- **THEN** no test step is executed
