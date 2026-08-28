# vkcloud-dns-client Specification

## Purpose
Defines the contract of the VK Cloud public-dns API v2 client: how it authenticates, how it looks up zones, and how it creates, finds, and deletes TXT records. This is the integration layer between the ACME solver and the VK Cloud DNS service.

## Requirements

### Requirement: Client authenticates via gophercloud OpenStack credentials

The client SHALL authenticate against the VK Cloud identity endpoint using OpenStack-style credentials (identity endpoint URL, username, password, project ID, domain name) via the gophercloud library. The authentication endpoint URL SHALL be taken from the configuration's `osAuthUrlSecretRef` secret.

#### Scenario: Successful authentication

- **WHEN** the client is constructed with valid credentials
- **THEN** a gophercloud `ProviderClient` is established and subsequent API calls are authenticated

#### Scenario: Authentication failure

- **WHEN** the client is constructed with invalid or unreachable credentials
- **THEN** construction returns an error and no API calls can be made

### Requirement: Zone lookup by name

The client SHALL be able to look up a DNS zone by its name. The lookup SHALL query the VK Cloud public-dns API for the list of all zones and return the one whose `zone` field matches the target name. The target name SHALL be compared case-sensitively after stripping any trailing dot.

#### Scenario: Zone exists

- **WHEN** the client looks up a zone by name and that zone exists in the VK Cloud DNS provider
- **THEN** the client returns a `Zone` object containing the zone's UUID and name

#### Scenario: Zone does not exist

- **WHEN** the client looks up a zone by name and no zone with that name exists
- **THEN** the client returns a `ZoneNotFoundError`

### Requirement: TXT record creation

The client SHALL be able to create a TXT record in a given zone. The record SHALL be specified by name (relative to the zone), content (the TXT value), and TTL (in seconds). The API endpoint SHALL be the zone-specific TXT records endpoint.

#### Scenario: Successful record creation

- **WHEN** the client creates a TXT record in an existing zone
- **THEN** the record is created and the API returns success

#### Scenario: Record creation fails

- **WHEN** the client attempts to create a TXT record and the API returns an error
- **THEN** the client returns that error

### Requirement: TXT record lookup by content

The client SHALL be able to find a TXT record in a given zone by its content value. The lookup SHALL query all TXT records in the zone and return the first one whose `content` field matches the target value.

#### Scenario: Record with matching content exists

- **WHEN** the client searches for a TXT record by content and at least one record in the zone has that content
- **THEN** the client returns the matching record (including its UUID)

#### Scenario: No record with matching content

- **WHEN** the client searches for a TXT record by content and no record in the zone has that content
- **THEN** the client returns a `RecordNotFoundErr`

### Requirement: TXT record deletion

The client SHALL be able to delete a TXT record from a given zone by its UUID.

#### Scenario: Successful record deletion

- **WHEN** the client deletes a TXT record by UUID and that record exists
- **THEN** the record is removed and the API returns success

#### Scenario: Record deletion fails

- **WHEN** the client attempts to delete a TXT record and the API returns an error
- **THEN** the client returns that error

### Requirement: API endpoint is fixed to mcs.mail.ru public-dns v2

The client SHALL use the hardcoded base URL `https://mcs.mail.ru/public-dns/v2/dns/` for all API calls. This endpoint is not configurable. The following paths SHALL be used:

- `GET /` — list all zones
- `GET /{zone-uuid}/txt/` — list all TXT records in a zone
- `POST /{zone-uuid}/txt/` — create a TXT record in a zone
- `DELETE /{zone-uuid}/txt/{record-uuid}` — delete a TXT record

#### Scenario: Zone list request

- **WHEN** the client looks up zones
- **THEN** it issues a GET request to `https://mcs.mail.ru/public-dns/v2/dns/`

#### Scenario: TXT record creation request

- **WHEN** the client creates a TXT record in zone with UUID `abc`
- **THEN** it issues a POST request to `https://mcs.mail.ru/public-dns/v2/dns/abc/txt/`

#### Scenario: TXT record deletion request

- **WHEN** the client deletes a TXT record with UUID `xyz` from zone with UUID `abc`
- **THEN** it issues a DELETE request to `https://mcs.mail.ru/public-dns/v2/dns/abc/txt/xyz`

### Requirement: Error types are distinguishable

The client SHALL expose two distinct error types for not-found conditions:

- `ZoneNotFoundError` — returned when a zone lookup fails to find the target zone
- `RecordNotFoundErr` — returned when a record lookup fails to find a matching record

These types SHALL be distinguishable via type assertion so that callers can handle "not found" differently from other errors (e.g., treating a missing record during cleanup as a no-op).

#### Scenario: Zone not found during lookup

- **WHEN** the zone lookup does not find the target zone
- **THEN** the returned error is of type `ZoneNotFoundError`

#### Scenario: Record not found during lookup

- **WHEN** the record lookup does not find a matching record
- **THEN** the returned error is of type `RecordNotFoundErr`
