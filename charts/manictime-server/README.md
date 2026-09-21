# manictime-server

A Helm chart for [ManicTime Server](https://www.manictime.com/teams), the
self-hosted / on-premise time-tracking server that ManicTime desktop clients
sync to. Once installed, it can be paired with ManicTime's MCP integration so
an AI assistant can query activity across all devices in the tenant.

## Installation

```bash
helm repo add d4rkeagle65 https://d4rkeagle65.github.io/helm-charts
helm install manictime-server d4rkeagle65/manictime-server \
  --namespace manictime --create-namespace \
  --set settings.database.host=manictime-pg-rw \
  --set settings.database.passwordSecret.name=manictime-pg-app
```

## Prerequisites

- Kubernetes 1.19+ (Ingress uses `networking.k8s.io/v1`)
- PostgreSQL 15 reachable from the pod, with two empty databases and a role
  that owns both (see the CNPG section below for the way this chart expects
  to be run in the target homelab)
- A `ReadWriteOnce` StorageClass for the `/app/Data` PVC

## How ManicTime Server is actually configured

Unlike most modern container images, `manictime/manictimeserver` does not
accept database credentials via environment variables. It reads them from
`/app/Data/ManicTimeServerSettings.json`, which must be present on the data
volume before the server starts. The file looks like this:

```json
{
  "database": {
    "provider": "PostgreSql",
    "connectionString": "Host=...; Port=5432; Database=ManicTimeCore; SSL Mode=Prefer; User Id=...; Password=...;"
  },
  "reporting": {
    "database": {
      "provider": "PostgreSql",
      "connectionString": "Host=...; Port=5432; Database=ManicTimeReports; SSL Mode=Prefer; User Id=...; Password=...;"
    }
  }
}
```

When `settings.seed` is true (the default), this chart renders that JSON into
a Secret and an init container copies it to the PVC on first launch. The init
container is idempotent: it only writes the file if it is missing, so Setup.dll
or manual edits on the PVC are never clobbered.

If you would rather manage the file yourself, set `settings.seed=false` and
either `kubectl cp` the file into `/app/Data` or point `settings.existingSecret`
at a Secret with a `ManicTimeServerSettings.json` key.

## First-run setup (admin account and license)

After the pod comes up with a valid settings file, run Setup.dll once to
create the admin account and enter your license or start the trial:

```bash
kubectl -n manictime exec -it deploy/manictime-server -- \
  dotnet Setup.dll -publicurl
```

The port question during setup should be left at 8080 (that is what the
container listens on internally).

## Using your existing CNPG cluster

This chart does not manage Postgres. The intended flow in a homelab that
already runs CloudNativePG is:

1. Create a `Cluster` for ManicTime in your existing CNPG install. Minimal
   shape:

   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Cluster
   metadata:
     name: manictime-pg
     namespace: manictime
   spec:
     instances: 1
     imageName: ghcr.io/cloudnative-pg/postgresql:15
     storage:
       size: 10Gi
       storageClass: nfs-retain-rwo
     bootstrap:
       initdb:
         database: ManicTimeCore
         owner: manictime
         postInitApplicationSQL:
           - CREATE DATABASE "ManicTimeReports" OWNER "manictime";
   ```

   CNPG will generate a Secret named `manictime-pg-app` in the same
   namespace with `username` / `password` / `uri` keys for the `manictime`
   role. It also creates:

   - `manictime-pg-rw` service for read/write traffic (use this)
   - `manictime-pg-ro` and `manictime-pg-r` services for replicas

2. Point the chart at that cluster. Two knobs matter:

   ```yaml
   settings:
     database:
       host: manictime-pg-rw
       user: manictime
       coreDatabase: ManicTimeCore
       reportsDatabase: ManicTimeReports
       passwordSecret:
         name: manictime-pg-app
         key: password
   ```

   The chart reads the password out of that Secret at template time (via
   `lookup`) and embeds it in the rendered `ManicTimeServerSettings.json`
   Secret. Because this happens at `helm install` / `helm upgrade` time,
   rerun `helm upgrade` if CNPG ever rotates the credential.

3. If you want the ManicTime chart to own its Postgres instead (single
   command `helm install`), the file `templates/cnpg-cluster.yaml` in
   this chart carries a commented-out CNPG `Cluster` + Secret that you
   can uncomment and gate behind a values flag.

## Prerequisites: MCP integration

ManicTime's MCP server sits alongside the Server install and exposes tools
like `get_environments` (list devices) and `get_combined_activities` (query
activity across devices). See
https://docs.manictime.com/ai-mcp-server/available-tools for the current
tool catalog.

## Source Code

* <https://www.manictime.com/teams/download>
* <https://hub.docker.com/r/manictime/manictimeserver>
* <https://docs.manictime.com/server>

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| additionalVolumeMounts | list | `[]` | Add custom volume mounts to the deployment |
| additionalVolumes | list | `[]` | Add custom volumes to the deployment (may need to match `additionalVolumeMounts`) |
| affinity | object | `{}` |  |
| env | object | `{}` | Extra environment variables for the ManicTime Server container |
| fullnameOverride | string | `""` |  |
| image.pullPolicy | string | `"IfNotPresent"` | Container image pull policy |
| image.registry | string | `"docker.io"` | Container image registry |
| image.repository | string | `"manictime/manictimeserver"` | Location of the container image |
| image.tag | string | `"2026.2.2"` | Container image tag |
| imagePullSecrets | list | `[]` | List of image pull secrets if you use a privately hosted image |
| ingress.annotations | object | `{}` | Additional annotations for the Ingress object |
| ingress.enabled | bool | `false` | Control whether ingress is created |
| ingress.hosts | list | `[]` | See Kubernetes Docs for a guide to setup Ingress hosts |
| ingress.tls | list | `[]` | See Kubernetes Docs for a guide to setup TLS on Ingress |
| livenessProbe.enabled | bool | `true` | Whether to enable the liveness probe |
| nameOverride | string | `""` |  |
| nodeSelector | object | `{}` |  |
| persistence.annotations | object | `{}` | Additional annotations to add to the PVC |
| persistence.enabled | bool | `true` | Whether to enable the PVC and mount for /app/Data |
| persistence.mountPath | string | `"/app/Data"` | Mount path inside the container |
| persistence.selector | object | `{}` | PV selector |
| persistence.size | string | `"5Gi"` | Requested storage size |
| persistence.storageClass | string | `""` | Storage Class name of the PV |
| podAnnotations | object | `{}` |  |
| podSecurityContext | object | `{}` |  |
| readinessProbe.enabled | bool | `true` | Whether to enable the readiness probe |
| replicaCount | int | `1` |  |
| resources.limits.memory | string | `"1Gi"` |  |
| resources.requests.cpu | string | `"50m"` |  |
| resources.requests.memory | string | `"256Mi"` |  |
| securityContext | object | `{}` |  |
| service.httpNodePort | int | `0` | Node port number if `service.type` is `NodePort` |
| service.httpPort | int | `8080` | Service http port number |
| service.type | string | `"ClusterIP"` | Service type |
| serviceAccount.annotations | object | `{}` | Annotations to add to the service account |
| serviceAccount.create | bool | `true` | Specifies whether a service account should be created |
| serviceAccount.name | string | `""` | The name of the service account to use. If not set and `create` is `true`, a name is generated using the fullname template |
| settings.seed | bool | `true` | Render and seed ManicTimeServerSettings.json into the PVC on first launch |
| settings.existingSecret | string | `""` | Use this Secret (key `ManicTimeServerSettings.json`) instead of rendering one from `settings.database.*` |
| settings.database.host | string | `""` | Postgres host (e.g. CNPG `rw` service) |
| settings.database.port | int | `5432` | Postgres port |
| settings.database.user | string | `"manictime"` | Postgres user |
| settings.database.password | string | `""` | Postgres password (prefer `passwordSecret`) |
| settings.database.passwordSecret.name | string | `""` | Existing Secret to read the password from at template time |
| settings.database.passwordSecret.key | string | `"password"` | Key in that Secret holding the password |
| settings.database.sslMode | string | `"Prefer"` | Npgsql SSL Mode |
| settings.database.coreDatabase | string | `"ManicTimeCore"` | Core database name |
| settings.database.reportsDatabase | string | `"ManicTimeReports"` | Reports database name |
| settings.extra | object | `{}` | Extra top-level keys merged into ManicTimeServerSettings.json |
| tolerations | list | `[]` |  |

## Credits

Chart layout modeled on the sibling `emby` chart in this repo, which is
itself based on <https://ccremer.github.io/charts>.
