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

## Git source for workflows

With `git.enabled: true`:

- the `kestra-git-sync` ConfigMap renders a Kestra flow `system/git-flow-sync`
  using `io.kestra.plugin.git.SyncFlows` with a `Schedule` trigger (cron
  `git.syncInterval`), pulling from `git.url` (`git.branch`, subdir
  `git.gitDirectory`) into `git.targetNamespace` (`includeChildNamespaces` /
  `delete` configurable);
- the `git-sync-bootstrap` sidecar (always present in the pod — it costs no
  extra scheduling slot, just a container) polls `:8081/health/readiness`, then
  `POST`s that flow to the Kestra API (`PUT` to update if it already exists) and
  triggers one immediate sync. When `git.enabled` is false no ConfigMap is
  mounted and the sidecar just idles; override `upstream.common.extraContainers: []`
  to drop it entirely.

The plugin ships in the `kestra/kestra` image, so nothing extra is needed.

### Private repo

Kestra flows can't read a Kubernetes Secret directly, so credentials are bridged
through env vars on the Kestra container. Set `git.auth.enabled: true`; the
`SyncFlows` task then uses `{{ envs.git_username }}` / `{{ envs.git_password }}`,
which Kestra resolves from the `ENV_GIT_USERNAME` / `ENV_GIT_PASSWORD` env vars.
The chart wires those from a Secret named **`kestra-git-auth`** in the release
namespace (keys **`username`** and **`password`**, plain values) with
`optional: true`, so a missing Secret only breaks git sync, not the pod.

Provide that Secret either yourself, or by setting these two values so the chart
renders a `SealedSecret` for it:

| value | becomes | key in the Secret |
|---|---|---|
| `git.auth.encryptedUsername` | `SealedSecret.spec.encryptedData.username` | `username` |
| `git.auth.encryptedPassword` | `SealedSecret.spec.encryptedData.password` | `password` |

Seal the **plaintext** credentials, scoped to that name and namespace:

```sh
printf %s 'x-access-token' | kubeseal --raw -n kestra --name kestra-git-auth   # -> git.auth.encryptedUsername
printf %s "$GIT_PAT"       | kubeseal --raw -n kestra --name kestra-git-auth   # -> git.auth.encryptedPassword
```

For a GitHub personal access token, `username` is typically `x-access-token`.

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
