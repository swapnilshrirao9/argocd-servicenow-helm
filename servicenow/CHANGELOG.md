# ServiceNow Informer Helm Chart - CHANGELOG

## Version 2.8.x (from 2.7.1)

---

## Breaking Changes

No breaking changes when upgrading from 2.7.1. However note that when upgrading from 2.5.x to this version using helm you need either to uninstall and install or to run
the following commands before helm upgrade. In those commands replace INSTANCE_NAME by your instance name and
NAMESPACE by the namespace in which the informer is installed. The INSTANCE_NAME is the name of your instance
without the the domain name. For example if your instance is `myinstance.service-now.com` the INSTANCE_NAME is `myinstance`.

```bash
kubectl label configmap k8s-informer-additional-resources-INSTANCE_NAME \
  app.kubernetes.io/managed-by=Helm -n NAMESPACE

kubectl annotate configmap k8s-informer-additional-resources-INSTANCE_NAME \ 
  meta.helm.sh/release-name=k8s-informer -n NAMESPACE

kubectl annotate configmap k8s-informer-additional-resources-INSTANCE_NAME \
 meta.helm.sh/release-namespace=NAMESPACE -n NAMESPACE
```
---

## Security Related Changes

### Hardened DaemonSet container security context when using a Daemonset
**Impact:** Connections-discovery DaemonSet pods now run with stricter Linux security primitives across all configurations.

**Changes:**
- Added `seccompProfile.type: RuntimeDefault` to the DaemonSet container in every branch (packet capture on/off, ServiceNow method, Cilium method).
- Added `allowPrivilegeEscalation: false` to the `packetCaptureEnabled: false` branch (previously only set when packet capture was enabled).
- New Cilium DaemonSet branch ships with the strictest defaults: `readOnlyRootFilesystem: true`, `runAsNonRoot: true`, `runAsUser: 1000`, `allowPrivilegeEscalation: false`, `seccompProfile.type: RuntimeDefault`, all capabilities dropped, and `hostNetwork: false` with `dnsPolicy: ClusterFirst` (no host network namespace sharing).

---

## Functional Changes

### Cilium / Hubble connections discovery (Optional)
**New Feature:** The connections-discovery DaemonSet and Service can now be rendered for a Cilium-based capture path that consumes flows from the Hubble UNIX domain socket on each node, instead of relying on host networking.

**Behavior:**
- DaemonSet and Service templates activate when `connectionsDiscovery.method` is `servicenow` **or** `cilium`.
- In `cilium` mode the pod mounts the Hubble socket from the host (`hostPath` / `type: Socket`), runs without `hostNetwork`, and skips the per-instance `serviceAccountName`.
- New environment variables surfaced to the DaemonSet container in `cilium` mode: `CONNECTIONS_HTTP_SERVER_PORT`, `CONNECTIONS_TLS_MODE`, `CONNECTIONS_CUSTOM_CA`, `HUBBLE_SOCKET_PATH`.
- `NODE_NAME` (from `spec.nodeName`) is now always exposed to the DaemonSet container.

**Configuration:**
- **Variable:** `connectionsDiscovery.cilium.hubbleSocketPath`
- **Default:** `/var/run/cilium/hubble.sock`
- **Use Case:** Path to the Hubble UDS on each node; used both as the `hostPath` source and as `HUBBLE_SOCKET_PATH` in the container.

### IBM Licensing integration (Optional)
**New Feature:** The informer can connect to an in-cluster IBM License Service, read product usage from its API, and forward daily usage reports to the ServiceNow instance.

**What ships:**
- New template `templates/ibm_licensing.yaml` that, when enabled, renders a `ClusterRoleBinding` for the informer ServiceAccount, optionally a `ClusterRole` granting read access to the IBM Licensing non-resource URLs (`/products`, `/snapshot`, `/bundled_products`, `/services`, `/health`, `/status`), and optionally a `Secret` (`ibm-license-ca-cert`) holding the CA used to validate the License Service TLS endpoint.
- When `ibmLicensing.useCaCertificate` is `true`, the deployment mounts `ibm-license-ca-cert` at `/etc/ibm-licensing` (read-only).
- New environment variables surfaced to the informer Deployment: `IBM_LICENSING_ENABLED`, `IBM_LICENSE_SERVICE_URL`, `IBM_LICENSING_PAYLOAD_CHUNK_SIZE`, `IBM_LICENSING_TIME_WINDOW_DAYS`, `IBM_LICENSING_NAMESPACE`.

**Configuration block** (`ibmLicensing.*`):
- `enabled` (default `false`) — master switch for the integration.
- `createClusterRole` (default `false`) — set to `true` if the cluster does not already have an `ibm-licensing-default-reader` ClusterRole.
- `clusterRole` (default `ibm-licensing-servicenow-reader`) — name prefix used when this chart creates the ClusterRole.
- `namespace` (default `ibm-licensing`) — namespace of the IBM License Service.
- `ibmLicenseServiceUrl` (default `""`) — override the in-cluster License Service URL.
- `payloadChunkSize` (default `500`) — records per upload chunk to the ServiceNow instance.
- `timeWindowDays` (default `90`) — max number of days included in a daily report.
- `useCaCertificate` (default `false`) — when `true`, validates the License Service TLS using `caCertificate`.
- `caCertificate` (default `""`) — PEM CA cert bundled into the rendered `ibm-license-ca-cert` Secret.

### Memory pressure monitoring (Optional)
**New Feature:** When a memory limit is configured, the informer now monitors its own memory usage and sends an alert to the ServiceNow instance when usage crosses a threshold.

**Behavior:** When `memoryLimit` is set, the deployment exposes two additional env vars to the informer container — `MEMORY_LIMIT` (sourced via `resourceFieldRef` from `limits.memory` of the `k8sinformer` container) and `MEMORY_PRESSURE_THRESHOLD_PCT`.

**Configuration:**
- **Variable:** `memoryPressureThresholdPct`
- **Default:** `80`
- **Use Case:** Percentage of `memoryLimit` at which a memory-pressure alert is emitted. Accepted range 1–100. Only active when `memoryLimit` is set.

---

## Configuration Summary

### New Configuration Variables

| Variable | Default | Optional | Description |
|----------|---------|----------|-------------|
| `memoryPressureThresholdPct` | `80` | Yes | Percent of `memoryLimit` at which a memory-pressure alert is emitted |
| `connectionsDiscovery.cilium.hubbleSocketPath` | `/var/run/cilium/hubble.sock` | Yes | Hubble UDS path on host nodes (used when `connectionsDiscovery.method` is `cilium`) |
| `ibmLicensing.enabled` | `false` | Yes | Master switch for the IBM Licensing integration |
| `ibmLicensing.createClusterRole` | `false` | Yes | Create an `ibm-licensing-*-reader` ClusterRole if one isn't already present |
| `ibmLicensing.clusterRole` | `ibm-licensing-servicenow-reader` | Yes | Name prefix for the ClusterRole this chart creates |
| `ibmLicensing.namespace` | `ibm-licensing` | Yes | Namespace where the IBM License Service is installed |
| `ibmLicensing.ibmLicenseServiceUrl` | `""` | Yes | Override the in-cluster License Service URL |
| `ibmLicensing.payloadChunkSize` | `500` | Yes | Records per upload chunk to the ServiceNow instance |
| `ibmLicensing.timeWindowDays` | `90` | Yes | Max days included in a daily usage report |
| `ibmLicensing.useCaCertificate` | `false` | Yes | Validate License Service TLS using `caCertificate` |
| `ibmLicensing.caCertificate` | `""` | Yes | PEM CA certificate bundled into the rendered `ibm-license-ca-cert` Secret |

### Environment Variables Added
- `NODE_NAME`
- `CONNECTIONS_HTTP_SERVER_PORT`
- `CONNECTIONS_TLS_MODE`
- `CONNECTIONS_CUSTOM_CA`
- `HUBBLE_SOCKET_PATH`
- `MEMORY_LIMIT`
- `MEMORY_PRESSURE_THRESHOLD_PCT`
- `IBM_LICENSING_ENABLED`
- `IBM_LICENSE_SERVICE_URL`
- `IBM_LICENSING_PAYLOAD_CHUNK_SIZE`
- `IBM_LICENSING_TIME_WINDOW_DAYS`
- `IBM_LICENSING_NAMESPACE`

---

### Backward Compatibility
Most new features (Cilium connections discovery, IBM Licensing, memory-pressure monitoring) are gated behind opt-in values and default to disabled, so an existing deployment that does not override the new keys will keep its current behavior.
