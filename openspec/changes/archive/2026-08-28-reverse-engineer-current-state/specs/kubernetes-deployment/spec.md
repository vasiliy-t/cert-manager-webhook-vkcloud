## Purpose

Defines how the webhook is deployed to a Kubernetes cluster: the Helm chart, the RBAC, the Service, the APIService registration, and the PKI/certificate setup that lets cert-manager reach the webhook over the Kubernetes API.

## ADDED Requirements

### Requirement: Helm chart packaging

The project SHALL be packaged as a Helm chart under `deploy/cert-manager-webhook-vkcloud/`. The chart SHALL be installable via `helm install` and SHALL render the following resources:

- A `Deployment` running the webhook container
- A `Service` exposing the webhook container port
- A `ServiceAccount` for the webhook
- An `APIService` registering the webhook with the Kubernetes API
- RBAC resources (`ClusterRole`, `ClusterRoleBinding`) granting the webhook permission to read Secrets
- PKI resources (Certificate, CertificateRequest) for the webhook's serving certificate

#### Scenario: Helm install creates all resources

- **WHEN** a user runs `helm install` with the chart
- **THEN** all of the above resources are created in the target namespace

#### Scenario: Helm template renders valid YAML

- **WHEN** the chart is rendered via `helm template`
- **THEN** the output is valid Kubernetes YAML with no unresolved template variables

### Requirement: Deployment runs the webhook container

The `Deployment` SHALL run the webhook container image. The container SHALL be configured with:

- An image reference (repository and tag) configurable via Helm values
- The `GROUP_NAME` environment variable set to the API group the webhook serves
- A `imagePullSecrets` list (optional) for pulling from private registries

#### Scenario: Default deployment

- **WHEN** the chart is installed with default values
- **THEN** the Deployment references the default image and sets `GROUP_NAME` appropriately

#### Scenario: Private registry

- **WHEN** the chart is installed with `imagePullSecrets` specified
- **THEN** the Deployment includes those secrets in its `spec.template.spec.imagePullSecrets`

### Requirement: Webhook service exposes the container port

The `Service` SHALL expose the webhook's container port (default 9443) so that the Kubernetes API server can reach the webhook.

#### Scenario: Service targets the webhook port

- **WHEN** the Service is created
- **THEN** it selects the webhook pods and exposes port 9443

### Requirement: APIService registers the webhook group

The `APIService` SHALL register the webhook's API group with the Kubernetes API server. The `APIService`'s `spec.service` SHALL point to the webhook's Service. The `CABundle` SHALL be populated with the webhook's serving certificate.

#### Scenario: APIService is registered

- **WHEN** the APIService is created
- **THEN** the Kubernetes API server exposes the webhook's group/version under the `apis/` tree

#### Scenario: CA bundle is set

- **WHEN** the APIService is created
- **THEN** its `spec.cabundle` contains the base64-encoded serving certificate

### Requirement: RBAC permits reading Secrets

The webhook's `ServiceAccount` SHALL be bound to a `ClusterRole` (or `Role`) that grants `get` and `list` permissions on `Secret` resources. This is required for the solver to read the DNS provider credentials from Secrets.

#### Scenario: Webhook can read Secrets

- **WHEN** the webhook's pod runs with its ServiceAccount
- **THEN** it has permission to `get` and `list` Secret resources in the namespaces where it is invoked

### Requirement: PKI resources for the serving certificate

The chart SHALL include resources (typically a cert-manager `Certificate` and `CertificateRequest`, or equivalent) that provision a serving certificate for the webhook. The certificate SHALL be mounted into the webhook pod and referenced by the APIService's `CABundle`.

#### Scenario: Certificate is provisioned

- **WHEN** the chart is installed in a cluster with cert-manager
- **THEN** a serving certificate is issued and mounted into the webhook pod
