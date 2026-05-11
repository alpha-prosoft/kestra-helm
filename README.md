# kestra-helm

Thin umbrella chart on top of the upstream [kestra](https://github.com/kestra-io/helm-charts)
chart, wired for our cluster as described in
[kestra.io/docs/installation/kubernetes](https://kestra.io/docs/installation/kubernetes).
The upstream chart is pulled in unmodified as a Helm dependency; this repo only
adds:

- a `DatabaseInstance` CR ([postgres-operator](https://github.com/alpha-prosoft/postgres-operator))
  so the Postgres database, role and credentials Secret are provisioned for us
- a `Bucket` CR (ACK S3 controller, `s3.services.k8s.aws/v1alpha1`) for Kestra's
  internal storage backend
- a Gateway API `HTTPRoute` (exposed at `kestra.${PublicHostedZoneName}`)
- wiring for Kestra's mandatory basic auth (and an optional `SealedSecret` for it)
- an optional **git source for workflows** — a bootstrap sidecar plus a
  `SyncFlows` flow that keeps a namespace in sync with a git repo
- defaults that point Kestra at our mirrored images and at the Postgres/S3 above
- a CI workflow that mirrors the upstream chart + the `kestra/kestra` and
  `docker:dind-rootless` images to our registry, then republishes the umbrella
  as an OCI artifact

Kestra runs as a single **standalone** server with the dind sidecar, backed by
Postgres (queue + repository) and S3 (storage).

## Layout

- `helm/kestra/Chart.yaml` — declares the upstream `kestra` chart as an OCI dependency on our mirror (alias `upstream`).
- `helm/kestra/values.yaml` — `gatewayApi.*`, `database.*`, `aws.*` blocks plus `upstream.*` overrides forwarded to the kestra chart.
- `helm/kestra/templates/databaseinstance.yaml` — postgres-operator `DatabaseInstance`.
- `helm/kestra/templates/bucket.yaml` — ACK S3 `Bucket`.
- `helm/kestra/templates/kestra-extra-config.yaml` — `kestra-extra-config` ConfigMap holding the S3 storage block (rendered from `aws.*`, mounted into Kestra via `upstream.configurations.configmaps`).
- `helm/kestra/templates/git-sync.yaml` — `kestra-git-sync` ConfigMap holding the `system/git-flow-sync` flow (rendered from `git.*`); only when `git.enabled`.
- `helm/kestra/templates/git-auth-sealedsecret.yaml` — `kestra-git-auth` `SealedSecret` (rendered from `git.auth.encrypted*`); only when those are set.
- `helm/kestra/templates/basic-auth-sealedsecret.yaml` — `kestra-basic-auth` `SealedSecret` (rendered from `basicAuth.encrypted*`); only when those are set.
- `helm/kestra/templates/httproute.yaml` — the Gateway API route.
- `.github/workflows/build.yml` — mirrors the upstream chart + images, then publishes the umbrella.

The dependency `tgz` is **not** vendored. CI mirrors both the chart and the
images from upstream into our registry; the umbrella depends on
`oci://${DOCKER_PUSH_URL}/kestra` so deploys never reach `helm.kestra.io` or
upstream Docker Hub repos.

## How the wiring fits together

- `database.create: true` renders a `DatabaseInstance`. postgres-operator
  creates the `kestra` database + role and writes a Secret named
  `database.targetSecretName` (default `kestra-db`) with keys
  `host/port/database/username/password/url`.
- `upstream.common.extraEnv` maps those Secret keys to `KESTRA_PG_*` env vars,
  and `upstream.configurations.application.datasources.postgres` references them
  via `${KESTRA_PG_*}` placeholders (resolved by Micronaut at runtime — the
  password never lands in a ConfigMap). **If you change `targetSecretName`,
  update the `secretKeyRef.name` entries under `upstream.common.extraEnv`.**
- `aws.s3.create: true` renders a `Bucket` (the ACK controller creates the real
  bucket). The same `aws.s3.bucketName` / `aws.region` are rendered into the
  `kestra-extra-config` ConfigMap as Kestra's `kestra.storage.s3` config so the
  two stay in sync.
- S3 credentials use the default AWS credential chain. For IRSA, set the role
  ARN on the service account: `upstream.serviceAccount.annotations.eks\.amazonaws\.com/role-arn`.

`DatabaseInstance` and `Bucket` carry `argocd.argoproj.io/sync-wave: "-1"` so
ArgoCD provisions (and waits for) them before the Kestra workload that consumes
the generated Secret.

## Authentication — the `kestra-auth` Secret

All four credentials Kestra needs live in a single Secret named **`kestra-auth`**
in the release namespace, with these keys (all plaintext):

| key | used as |
|---|---|
| `basicAuthEmail` | Kestra server basic auth username (must be a valid email) |
| `basicAuthPassword` | Kestra server basic auth password |
| `gitUser` | git username for the SyncFlows task and any imported flow |
| `gitToken` | git password / PAT (same) |

The chart wires them into env vars (all `optional: true`, so a missing Secret or
missing key only breaks the corresponding feature, not the pod):

| env var on the container | Secret key | used by |
|---|---|---|
| `KESTRA_SERVER_BASIC_AUTH_USERNAME` | `basicAuthEmail` | `kestra.server.basic-auth.username` |
| `KESTRA_SERVER_BASIC_AUTH_PASSWORD` | `basicAuthPassword` | same, password |
| `ENV_GIT_USERNAME` | `gitUser` | `{{ envs.git_username }}` in any flow (parent sync or imported) |
| `ENV_GIT_PASSWORD` | `gitToken` | `{{ envs.git_password }}` in any flow |

The `git-sync-bootstrap` sidecar `envFrom`s the same Secret and uses
`$basicAuthEmail` / `$basicAuthPassword` to authenticate its own API calls.

Bring the Secret yourself with all four keys, or set the kubeseal-encrypted
plaintext values below to have the chart render a single `kestra-auth`
SealedSecret for you. Only keys with non-empty values are rendered, so partial
configurations (e.g. basic auth only) work.

| value | → key in the Secret |
|---|---|
| `auth.encryptedBasicAuthEmail` | `basicAuthEmail` |
| `auth.encryptedBasicAuthPassword` | `basicAuthPassword` |
| `auth.encryptedGitUser` | `gitUser` |
| `auth.encryptedGitToken` | `gitToken` |

Seal the **plaintext** of each, scoped to `kestra-auth` in the release namespace:

```sh
printf %s 'admin@example.com' | kubeseal --raw -n kestra --name kestra-auth   # -> auth.encryptedBasicAuthEmail
printf %s "$KESTRA_PASSWORD"  | kubeseal --raw -n kestra --name kestra-auth   # -> auth.encryptedBasicAuthPassword
printf %s 'x-access-token'    | kubeseal --raw -n kestra --name kestra-auth   # -> auth.encryptedGitUser
printf %s "$GIT_PAT"          | kubeseal --raw -n kestra --name kestra-auth   # -> auth.encryptedGitToken
```

For a GitHub personal access token, `gitUser` is typically `x-access-token`.

> Basic auth is mandatory since Kestra 0.24 — without it the API returns `401`
> and the UI shows a setup page (config-file credentials take precedence over
> anything entered there).

## Arbitrary sealed config files — `secrets`

The `secrets` value is a map of `filename -> kubeseal-encrypted content`. Each
entry becomes one key in a SealedSecret named **`kestra-secrets`** and lands
as a file at `/secrets/<filename>` inside the Kestra container (mounted
read-only via `upstream.common.extraVolumes` / `extraVolumeMounts`; the Secret
reference is `optional: true` so an empty map is fine).

```yaml
secrets:
  application.properties.prod: AgB...
  extra.yml:                   AgB...
```

Seal each file's plaintext, scoped to `kestra-secrets` in the release namespace:

```sh
printf %s "$(cat application.properties.prod)" \
  | kubeseal --raw -n kestra --name kestra-secrets   # -> secrets."application.properties.prod"
```

Use the files from inside Kestra however you'd like — e.g. add
`/secrets/application.properties.prod` to a flow's command, or reference it
from `upstream.configurations` if it's a Micronaut config file.

## Git source for workflows

With `git.enabled: true`:

- the `kestra-git-sync` ConfigMap renders a Kestra flow `system/git-flow-sync`
  using `io.kestra.plugin.git.SyncFlows` with a `Schedule` trigger (cron
  `git.syncInterval`), pulling from `git.url` (`git.branch`, subdir
  `git.gitDirectory`) into `git.targetNamespace` (`includeChildNamespaces` /
  `delete` configurable);
- the `git-sync-bootstrap` sidecar (always present in the pod — it costs no
  extra scheduling slot, just a container) polls `:8081/health/readiness`, then
  `POST`s that flow to the Kestra API (`PUT` to update if it already exists,
  authenticating with the `basicAuthEmail` / `basicAuthPassword` keys above) and
  triggers one immediate sync. When `git.enabled` is false no ConfigMap is
  mounted and the sidecar just idles; override `upstream.common.extraContainers: []`
  to drop it entirely.

The git plugin ships in the `kestra/kestra` image, so nothing extra is needed.

### Private repo

Set `git.auth: true` (a plain boolean). The `SyncFlows` task then passes
`{{ envs.git_username }}` / `{{ envs.git_password }}`, resolved from the
`gitUser` / `gitToken` keys of `kestra-auth`. Imported flows that themselves do
git work reference the same `envs.*` variables:

```yaml
- id: clone
  type: io.kestra.plugin.git.Clone
  url: https://github.com/your-org/your-repo
  username: "{{ envs.git_username }}"
  password: "{{ envs.git_password }}"
```

Note: `envs.*` values are **not** redacted in the Kestra UI/logs. If that's a
concern for you, configure a real Kestra secret backend (AWS Secrets Manager,
Vault, etc.) instead of the env-var bridge.

## Required values

- `aws.region` — AWS region for the bucket and Kestra's S3 client.
- `aws.s3.bucketName` — globally unique S3 bucket name.
- `gatewayApi.hostedZoneName` — only when `gatewayApi.enabled: true`.
- `git.url` — only when `git.enabled: true`.

## Local render

After CI has run at least once (so the OCI mirror is populated):

```sh
helm dependency build helm/kestra
helm template test helm/kestra \
  --set aws.region=eu-west-1 \
  --set aws.s3.bucketName=my-kestra-storage \
  --set gatewayApi.enabled=true \
  --set gatewayApi.hostedZoneName=example.com
```

## Bumping the upstream version

Change `appVersion` (the `kestra/kestra` image tag) and
`dependencies[0].version` (the upstream chart version) in
`helm/kestra/Chart.yaml` to a release listed on
[ArtifactHub](https://artifacthub.io/packages/helm/kestra/kestra) and let CI
re-mirror and re-publish.

## Install

Deployed as an ArgoCD `Application` pointing at the OCI chart:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kestra
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: docker.io/alphaprosoft
    chart: kestra-helm
    targetRevision: 1.X-main
    helm:
      releaseName: kestra
      values: |
        aws:
          region: eu-west-1
          s3:
            bucketName: my-kestra-storage
        gatewayApi:
          enabled: true
          hostedZoneName: example.com
  destination:
    server: https://kubernetes.default.svc
    namespace: kestra
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Requires postgres-operator and the ACK S3 controller running in the cluster, and
a Gateway named per `gatewayApi.parentRef` if `gatewayApi.enabled`.

## CI secrets / variables

- `secrets.DOCKER_PUSH_USERNAME` — registry username; doubles as the namespace when `DOCKER_PUSH_URL` is host-only.
- `secrets.DOCKER_PUSH_PASSWORD` — registry access token.
- `vars.DOCKER_PUSH_URL` — registry. Either host-only (`docker.io`) or host+namespace (`docker.io/myorg`). Host-only means artifacts push to `docker.io/{username}/{name}`.
