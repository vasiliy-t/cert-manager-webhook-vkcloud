# acme-solver Specification

## Purpose
Defines the ACME DNS01 solver behavior of the webhook: how it presents and cleans up ACME challenges for the VK Cloud DNS provider, the solver name it registers under, and the configuration surface exposed to cert-manager.

## Requirements

### Requirement: Solver registration name

The webhook SHALL register itself with cert-manager under the solver name `vkcloud`. This name is used in cert-manager `Issuer` and `ClusterIssuer` resources to select this solver.

#### Scenario: Solver name is unique within the group

- **WHEN** cert-manager loads the webhook's registered solvers
- **THEN** the solver with name `vkcloud` is available for selection in `acme.dns01.providers.webhook` configurations

### Requirement: Present creates a TXT record for the challenge

When cert-manager invokes the `Present` method with a `ChallengeRequest`, the solver SHALL create a TXT record in the target DNS zone containing the ACME challenge key. The record's host name SHALL be derived from the `ResolvedFQDN` and `ResolvedZone` fields of the request.

#### Scenario: Successful challenge presentation

- **WHEN** cert-manager calls `Present` with a valid `ChallengeRequest` whose `ResolvedZone` exists in the VK Cloud DNS provider
- **THEN** the solver creates a TXT record with the challenge key in that zone
- **AND** the record's host name is the FQDN with the zone suffix stripped

#### Scenario: Zone does not exist

- **WHEN** cert-manager calls `Present` with a `ChallengeRequest` whose `ResolvedZone` is not found in the VK Cloud DNS provider
- **THEN** the solver returns an error indicating the zone was not found
- **AND** no DNS record is created

#### Scenario: Record creation fails

- **WHEN** cert-manager calls `Present` and the VK Cloud DNS API returns an error during record creation
- **THEN** the solver returns that error to cert-manager

### Requirement: CleanUp removes the challenge TXT record

When cert-manager invokes the `CleanUp` method with a `ChallengeRequest`, the solver SHALL locate and delete the previously-created TXT record matching the challenge key. If no such record exists, the operation SHALL be a no-op (return success without error).

#### Scenario: Successful record cleanup

- **WHEN** cert-manager calls `CleanUp` with a `ChallengeRequest` whose challenge key matches an existing TXT record in the target zone
- **THEN** the solver deletes that record from the VK Cloud DNS provider

#### Scenario: Record already absent

- **WHEN** cert-manager calls `CleanUp` with a `ChallengeRequest` whose challenge key does not match any existing TXT record in the target zone
- **THEN** the solver returns success without error (no-op)

#### Scenario: Zone does not exist during cleanup

- **WHEN** cert-manager calls `CleanUp` with a `ChallengeRequest` whose `ResolvedZone` is not found
- **THEN** the solver returns an error

### Requirement: Initialize sets up the Kubernetes client

When the webhook starts, the `Initialize` method SHALL be called by the cert-manager webhook framework. The solver SHALL use the provided Kubernetes client configuration to construct a Kubernetes clientset, which is used to read Secret resources containing DNS provider credentials.

#### Scenario: Webhook startup

- **WHEN** the webhook process starts
- **THEN** `Initialize` is called with the in-cluster Kubernetes configuration
- **AND** the solver holds a usable Kubernetes clientset for subsequent `Present`/`CleanUp` calls

### Requirement: Configuration is decoded from cert-manager JSON

The solver SHALL accept configuration as a JSON object in the `ChallengeRequest.Config` field. The configuration SHALL contain five fields, each a Kubernetes `SecretKeySelector`:

- `osAuthUrlSecretRef`
- `osUsernameSecretRef`
- `osPasswordSecretRef`
- `osProjectIDSecretRef`
- `osDomainNameSecretRef`

If the `Config` field is nil, the solver SHALL proceed with an empty configuration (which will cause authentication to fail at call time).

#### Scenario: Configuration with all five secret references

- **WHEN** cert-manager invokes `Present` with a `Config` containing all five `SecretKeySelector` fields
- **THEN** the solver decodes the configuration and uses it to build the authentication options for the VK Cloud DNS client

#### Scenario: Configuration is nil

- **WHEN** cert-manager invokes `Present` with a nil `Config`
- **THEN** the solver proceeds with an empty configuration
- **AND** subsequent authentication against the VK Cloud DNS API will fail

#### Scenario: Malformed configuration JSON

- **WHEN** cert-manager invokes `Present` with a `Config` whose JSON cannot be decoded into the expected structure
- **THEN** the solver returns an error describing the decode failure

### Requirement: Credentials are read from Kubernetes Secrets

The solver SHALL read authentication credentials from Kubernetes `Secret` resources in the namespace specified by the `ChallengeRequest.ResourceNamespace` field. Each of the five configuration fields SHALL reference a distinct key within a Secret.

#### Scenario: All five secrets present

- **WHEN** the solver needs to authenticate and all five referenced Secrets exist in the target namespace with the referenced keys
- **THEN** the solver successfully constructs the authentication options

#### Scenario: One or more secrets missing

- **WHEN** the solver needs to authenticate and one or more referenced Secrets do not exist, or do not contain the referenced key
- **THEN** the solver returns an error identifying the missing secret or key

### Requirement: Record TTL is fixed at 3600 seconds

When creating a TXT record for an ACME challenge, the solver SHALL set the record's TTL to 3600 seconds (1 hour). This value is not configurable.

#### Scenario: Record creation uses fixed TTL

- **WHEN** the solver creates a TXT record during `Present`
- **THEN** the record's TTL is 3600
