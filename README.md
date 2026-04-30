# fwportal Helm Chart

Deploys `eliassengroup/fwportal` as a Kubernetes StatefulSet.

---

## Prerequisites

- Kubernetes 1.21+
- Helm 3.x
- Required Secrets and ConfigMaps created in the target namespace (see below)

---

## Required ConfigMap

One ConfigMap must exist in the target namespace before installing the chart.

### application-config

Mounted at `/app/config/application.properties`.

```bash
kubectl create configmap application-config \
  --from-file=application.properties=/path/to/application.properties \
  --namespace <your-namespace>
```

> **Note:** The key name used in `--from-file` must be exactly `application.properties` as the chart mounts it using `subPath`.

### Overriding ConfigMap Names

```yaml
applicationConfig:
  configMapName: my-application-config
```

---

## Required Secrets

Three secrets are required before installing the chart. Additional optional secrets can be provided for TLS and database certificates.

### Generating the key files

Use the `LwEncryption.jar` tool (obtained separately) to generate each key file:

```bash
java -jar LwEncryption.jar -k portal.key
java -jar LwEncryption.jar -k framework.key
```

This produces `portal.key` and `framework.key` in your current directory.

### portal-key

Mounted at `/app/config/portal.key`.

```bash
kubectl create secret generic portal-key \
  --from-file=portal.key=./portal.key \
  --namespace <your-namespace>
```

### framework-key

Mounted at `/app/config/framework.key`.

```bash
kubectl create secret generic framework-key \
  --from-file=framework.key=./framework.key \
  --namespace <your-namespace>
```

> **Note:** The key filename used in `--from-file` must match exactly (`portal.key` and `framework.key`). The chart mounts each secret using `subPath`, so the filename inside the container is derived from the secret key name.

### lw-license

Mounted at `/app/config/customer_LW_license.properties`.

```bash
kubectl create secret generic lw-license \
  --from-file=customer_LW_license.properties=/path/to/customer_LW_license.properties \
  --namespace <your-namespace>
```

> **Note:** The key name used in `--from-file` must be exactly `customer_LW_license.properties` as the chart mounts it using `subPath`.

### TLS / Certificate Secrets (optional)

JKS/certificate Secrets can be provided. When any are set, their files are **projected together into `/app/config/tls/`** — each file appears at `/app/config/tls/<key-name>` based on the key name used in `--from-file`.

| Value key | Description |
|---|---|
| `tlsCert.secretName` | TLS keystore |
| `dbCert.secretName` | Database certificate |
| `truststore.secretName` | Truststore |

Example — create each secret:

```bash
kubectl create secret generic tls-cert \
  --from-file=tls.jks=./tls.jks \
  --namespace <your-namespace>

kubectl create secret generic db-cert \
  --from-file=db.jks=./db.jks \
  --namespace <your-namespace>

kubectl create secret generic truststore \
  --from-file=truststore.jks=./truststore.jks \
  --namespace <your-namespace>

```

Then set whichever apply in your `overrides.yaml`:

```yaml
tlsCert:
  secretName: tls-cert

dbCert:
  secretName: db-cert

truststore:
  secretName: truststore

```

If all are empty (the default), the projected volume is omitted entirely.

### Overriding Secret Names

```yaml
portalKey:
  secretName: my-portal-key

frameworkKey:
  secretName: my-framework-key

lwLicense:
  secretName: my-lw-license

tlsCert:
  secretName: my-tls-cert      # optional

dbCert:
  secretName: my-db-cert       # optional

truststore:
  secretName: my-truststore    # optional

```

---

## Configuration

The following values can be overridden in `overrides.yaml` or via `--set`.

| Key | Default | Description |
|---|---|---|
| **Workload** | | |
| `replicaCount` | `1` | Number of StatefulSet replicas |
| `statefulsetAnnotations` | `{}` | Annotations added to the StatefulSet resource itself |
| `env` | `[]` | Environment variables injected into the container (standard `name`/`value` list) |
| **Image** | | |
| `image.registry` | `docker.io` | Container registry |
| `image.repository` | `eliassengroup/fwportal` | Image repository |
| `image.tag` | `""` | Image tag — defaults to `Chart.AppVersion` when empty |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `imagePullSecrets` | `[]` | List of registry pull secret names |
| **Service Account** | | |
| `serviceAccount.create` | `false` | Create a ServiceAccount for the pods |
| `serviceAccount.name` | `""` | SA name to use or create — falls back to the release fullname (create=true) or `default` (create=false) |
| `serviceAccount.annotations` | `{}` | Annotations added to the created ServiceAccount |
| **Service** | | |
| `service.type` | `ClusterIP` | Kubernetes service type |
| `service.port` | `10680` | Service port |
| `service.targetPort` | `10680` | Container port — also used as the readiness probe port |
| `service.annotations` | `{}` | Annotations applied to the Service resource |
| **Readiness Probe** | | |
| `readinessProbe.enabled` | `true` | Set to `false` to disable the readiness probe entirely |
| `readinessProbe.httpGet.scheme` | `HTTP` | Set to `HTTPS` if the container serves TLS directly |
| **Resources** | | |
| `resources.requests.cpu` | `250m` | CPU request |
| `resources.requests.memory` | `256Mi` | Memory request |
| `resources.limits.cpu` | `500m` | CPU limit |
| `resources.limits.memory` | `512Mi` | Memory limit |
| **Config Mounts** | | |
| `applicationConfig.configMapName` | `application-config` | ConfigMap name for `application.properties` |
| `portalKey.secretName` | `portal-key` | Secret name for `portal.key` |
| `frameworkKey.secretName` | `framework-key` | Secret name for `framework.key` |
| `lwLicense.secretName` | `lw-license` | Secret name for `customer_LW_license.properties` |
| `tlsCert.secretName` | `""` | **Optional.** TLS keystore secret — files projected into `/app/config/tls/` |
| `dbCert.secretName` | `""` | **Optional.** DB certificate secret — files projected into `/app/config/tls/` |
| `truststore.secretName` | `""` | **Optional.** Truststore secret — files projected into `/app/config/tls/` |
| **Extra Files** | | |
| `extraFiles.configMaps` | `[]` | ConfigMaps projected as files into `/app/extra/` |
| `extraFiles.secrets` | `[]` | Secrets projected as files into `/app/extra/` |
| `extraFiles.volumes` | `[]` | Volumes mounted as subdirectories under `/app/extra/<name>/` |
| **Storage** | | |
| `documents.claimName` | `""` | **Optional.** If set, mounts the pre-existing PVC as the documents volume |
| `documents.mountPath` | `/eliassen/documents` | Mount path for the documents volume |
| `archive.claimName` | `""` | **Optional.** If set, mounts the pre-existing PVC as the archive volume |
| `archive.mountPath` | `/eliassen/archive` | Mount path for the archive volume |
| `persistence.retentionPolicy.whenDeleted` | `Retain` | PVC behavior when the StatefulSet is deleted (`Retain` or `Delete`) |
| `persistence.retentionPolicy.whenScaled` | `Retain` | PVC behavior when the StatefulSet is scaled down (`Retain` or `Delete`) |
| **Ingress** | | |
| `ingress.enabled` | `false` | Create an Ingress resource |
| `ingress.ingressClassName` | `""` | Ingress class name (e.g. `nginx`, `alb`) |
| `ingress.annotations` | `{}` | Annotations added to the Ingress resource |
| `ingress.hosts` | see below | List of host/path rules — traffic is forwarded to the `fwportal` service port |
| `ingress.tls` | `[]` | TLS configuration for the Ingress |
| **Scheduling** | | |
| `topologySpreadConstraints` | see below | List of topology spread constraints applied to pods |

### Topology Spread Constraints

By default the chart applies two best-effort spread constraints to distribute pods across zones and nodes:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
```

Both use `ScheduleAnyway`, so pods will still be scheduled even if the cluster nodes don't carry the `topology.kubernetes.io/zone` label (e.g., single-zone or on-prem clusters). To disable spreading entirely:

```yaml
topologySpreadConstraints: []
```

### Ingress

To expose the application via an Ingress, set `ingress.enabled: true` and provide at least one host. Traffic is routed to the `fwportal` named service port (`10680` by default).

```yaml
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  hosts:
    - host: fwportal.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: fwportal-tls
      hosts:
        - fwportal.example.com
```

`ingressClassName` sets `spec.ingressClassName` on the Ingress resource. Leave it empty (`""`) if you rely on an annotation or a cluster-wide default class instead.

### Extra Files (`/app/extra`)

Any ConfigMaps, Secrets, or volumes not known to the chart can be made available to the application under `/app/extra/`. The application must be configured to read files from the full path.

**ConfigMaps and Secrets** are projected flat into `/app/extra/` — each key becomes a file:

```yaml
extraFiles:
  configMaps:
    - name: my-extra-config      # keys appear as /app/extra/<key>
  secrets:
    - name: my-extra-secret      # keys appear as /app/extra/<key>
```

**Volumes** (PVCs, emptyDir, etc.) are mounted as subdirectories at `/app/extra/<name>/`:

```yaml
extraFiles:
  volumes:
    - name: my-data
      persistentVolumeClaim:
        claimName: my-pvc        # files appear under /app/extra/my-data/
    - name: my-scratch
      emptyDir: {}               # empty writable dir at /app/extra/my-scratch/
```

All three can be combined in the same release.

---

## Installing the Chart

### Add the Helm repository

```bash
helm repo add eliassen https://eliasseninnovation.github.io/fwportal-helm
helm repo update
```

### Install

```bash
helm install fwportal eliassen/fwportal \
  --namespace <your-namespace> \
  -f overrides.yaml
```

### Install a specific version

```bash
helm install fwportal eliassen/fwportal \
  --version 1.0.0 \
  --namespace <your-namespace> \
  -f overrides.yaml
```

### Upgrade

```bash
helm upgrade fwportal eliassen/fwportal \
  --namespace <your-namespace> \
  -f overrides.yaml
```

### Uninstall

```bash
helm uninstall fwportal --namespace <your-namespace>
```

