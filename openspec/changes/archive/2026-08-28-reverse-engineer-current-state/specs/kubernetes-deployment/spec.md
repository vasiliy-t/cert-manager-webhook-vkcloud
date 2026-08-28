## Purpose
Defines how the webhook is deployed to a Kubernetes cluster: the Helm chart, the RBAC, the Service, the APIService registration, and the PKI/certificate setup that lets cert-manager reach the webhook over the Kubernetes API.

## ADDED Requirements

### Requirement: Helm chart packaging

The project SHALL be packaged as a Helm chart under `deploy/cert-manager-webhook-vkcloud/`. The chart SHALL be installable via `helm install` and SHALL render the following resources:

- A `Deployment` running the webhook container
- A `Service` exposing the webhook container port
- A `ServiceAccount` for the webhook
- An `APIService` registering the webhook with the Kubernetes API
- RBAC resources (see the RBAC requirements below)
- PKI resources (self-signed Issuer, CA Certificate, CA Issuer, serving Certificate) for the webhook's serving certificate

#### Scenario: Helm install creates all resources

- **WHEN** a user runs `helm install` with the chart
- **THEN** all of the above resources are created in the target namespace

#### Scenario: Helm template renders valid YAML

- **WHEN** the chart is rendered via `helm template`
- **THEN** the output is valid Kubernetes YAML with no unresolved template variables

### Requirement: Deployment runs the webhook container

The `Deployment` SHALL run the webhook container image. The container SHALL be configured with:

- An image reference (repository and tag) configurable via Helm values; the default is `ghcr.io/tarantool/cert-manager-webhook-vkcloud:latest`
- The `GROUP_NAME` environment variable set to the API group the webhook serves; the default value is `try.tarantool.io` (a legacy of the tarantool origin — a rebranding candidate)
- An `imagePullSecrets` list (optional) for pulling from private registries

#### Scenario: Default deployment

- **WHEN** the chart is installed with default values
- **THEN** the Deployment references `ghcr.io/tarantool/cert-manager-webhook-vkcloud:latest` and sets `GROUP_NAME` to `try.tarantool.io`

#### Scenario: Private registry

- **WHEN** the chart is installed with `imagePullSecrets` specified
- **THEN** the Deployment includes those secrets in its `spec.template.spec.imagePullSecrets`

### Requirement: Webhook serves TLS and exposes a health endpoint

The webhook container SHALL serve HTTPS on container port 443 (port name `https`) using a TLS certificate and key mounted from the serving-certificate Secret and passed via the `--tls-cert-file` and `--tls-private-key-file` arguments. The container SHALL expose a `/healthz` endpoint over HTTPS, used by both the liveness and readiness probes.

#### Scenario: TLS certificate is mounted

- **WHEN** the webhook pod starts
- **THEN** the serving-certificate Secret is mounted read-only at `/tls`
- **AND** the webhook process is started with `--tls-cert-file=/tls/tls.crt` and `--tls-private-key-file=/tls/tls.key`

#### Scenario: Probes target /healthz

- **WHEN** the kubelet probes the webhook pod
- **THEN** both liveness and readiness probes perform an HTTPS GET on `/healthz` at the `https` port

### Requirement: Webhook service exposes the container port

The `Service` SHALL be of type `ClusterIP` (by default) and SHALL expose port 443 (name `https`), targeting the container's `https` port, so that the Kubernetes API server can reach the webhook.

#### Scenario: Service targets the webhook port

- **WHEN** the Service is created with default values
- **THEN** it selects the webhook pods and exposes port 443 targeting the container port named `https`

### Requirement: APIService registers the webhook group

The `APIService` SHALL register the webhook's API group (name `v1alpha1.<groupName>`, version `v1alpha1`) with the Kubernetes API server. The `APIService`'s `spec.service` SHALL point to the webhook's Service. The `caBundle` SHALL NOT be set statically in the chart; instead, the APIService SHALL carry the annotation `cert-manager.io/inject-ca-from` referencing the serving certificate, so that the cert-manager cainjector injects the CA bundle at runtime. This creates a hard dependency on cert-manager (with cainjector) being installed in the cluster.

#### Scenario: APIService is registered

- **WHEN** the APIService is created
- **THEN** the Kubernetes API server exposes the webhook's group/version under the `apis/` tree

#### Scenario: CA bundle is injected by cainjector

- **WHEN** the APIService is created in a cluster with cert-manager's cainjector running
- **THEN** the cainjector populates `spec.caBundle` with the CA certificate referenced by the `cert-manager.io/inject-ca-from` annotation

#### Scenario: cainjector is absent

- **WHEN** the APIService is created in a cluster without cert-manager's cainjector
- **THEN** `spec.caBundle` remains empty and the Kubernetes API server cannot verify the webhook's serving certificate

### Requirement: RBAC grants the webhook read access to a single named Secret

The chart SHALL create a `Role` (not a ClusterRole) in the cert-manager namespace (`.Values.certManager.namespace`, default `cert-manager`) that grants the webhook's ServiceAccount `get` and `watch` (NOT `list`) permissions on Secret resources, restricted via `resourceNames` to the single Secret named `vkcloud-secret`. A corresponding `RoleBinding` SHALL bind this Role to the webhook's ServiceAccount.

As a consequence, the solver can only read credentials from a Secret named exactly `vkcloud-secret` in the cert-manager namespace, regardless of what the issuer configuration references.

#### Scenario: Webhook reads the named secret

- **WHEN** the webhook's pod runs with its ServiceAccount and the issuer configuration references the Secret `vkcloud-secret` in the cert-manager namespace
- **THEN** the webhook has permission to `get` that Secret

#### Scenario: Webhook cannot read other secrets

- **WHEN** the issuer configuration references a Secret with any name other than `vkcloud-secret`
- **THEN** the Kubernetes API denies the read with a Forbidden error and challenge solving fails

### Requirement: RBAC wires the webhook into API-server auth delegation

The chart SHALL create:

- A `RoleBinding` in the `kube-system` namespace binding the webhook's ServiceAccount to the `extension-apiserver-authentication-reader` Role (so the webhook can read the API server's requestheader CA ConfigMap)
- A `ClusterRoleBinding` binding the webhook's ServiceAccount to the `system:auth-delegator` ClusterRole (so the webhook can delegate authentication decisions to the core API server)

#### Scenario: Webhook authenticates API-server requests

- **WHEN** the Kubernetes API server forwards a challenge request to the webhook
- **THEN** the webhook can validate the request using the delegated authentication configuration

### Requirement: RBAC grants cert-manager access to the solver group

The chart SHALL create a `ClusterRole` granting `create` on all resources in the webhook's API group (`.Values.groupName`) and a `ClusterRoleBinding` binding it to the **cert-manager** ServiceAccount (`.Values.certManager.serviceAccountName` in `.Values.certManager.namespace`, defaults `cert-manager`/`cert-manager`), so that cert-manager may submit ChallengePayload resources to the webhook's API.

#### Scenario: cert-manager creates challenge payloads

- **WHEN** cert-manager solves a DNS01 challenge via this webhook
- **THEN** it is authorized to `create` resources in the webhook's API group

### Requirement: PKI resources for the serving certificate

The chart SHALL provision the webhook's serving certificate through a four-resource cert-manager chain, all in the release namespace:

1. A self-signed `Issuer`
2. A CA `Certificate` (5-year duration, `isCA: true`) issued by the self-signed Issuer
3. A CA `Issuer` backed by the CA Certificate's Secret
4. A serving `Certificate` (1-year duration) issued by the CA Issuer, with `dnsNames` covering `<fullname>`, `<fullname>.<namespace>`, and `<fullname>.<namespace>.svc`

The serving certificate's Secret SHALL be mounted into the webhook pod, and the serving certificate SHALL be referenced by the APIService's `cert-manager.io/inject-ca-from` annotation.

#### Scenario: Certificate chain is provisioned

- **WHEN** the chart is installed in a cluster with cert-manager
- **THEN** the self-signed Issuer, CA Certificate, CA Issuer, and serving Certificate are created
- **AND** the serving certificate's Secret is mounted into the webhook pod at `/tls`
