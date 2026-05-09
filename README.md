# aviowiki-cluster-sealed-secrets-web

Thin umbrella chart on top of [bakito/sealed-secrets-web](https://github.com/bakito/sealed-secrets-web).
The upstream chart is pulled in unmodified as a helm dependency; this repo only
adds:

- a Gateway API `HTTPRoute` template (exposed at `secret.${PublicHostedZoneName}`)
- defaults that point the deployment at our mirrored image
- a CI workflow that mirrors the upstream image to DockerHub and republishes
  the umbrella chart as an OCI artifact

## Layout

- `helm/sealed-secrets-web/Chart.yaml` — declares the bakito chart as an OCI dependency on our DockerHub mirror.
- `helm/sealed-secrets-web/values.yaml` — `gatewayApi.*` block and `upstream.*` overrides forwarded to the bakito chart.
- `helm/sealed-secrets-web/templates/httproute.yaml` — the only template we own.
- `.github/workflows/build.yml` — mirrors the upstream image and chart, then publishes the umbrella.

The dependency `tgz` is **not** vendored. CI mirrors both the image and the
chart from upstream into our DockerHub registry; the umbrella chart depends on
`oci://${DOCKER_PUSH_URL}/sealed-secrets-web` so deploys never need to reach
`charts.bakito.net` or `ghcr.io`.

To bump the upstream version, change `appVersion` and `dependencies[0].version`
in `Chart.yaml` to a release listed at
[charts.bakito.net](https://charts.bakito.net) and let CI re-publish.

## Local render

After CI has run at least once (so the OCI mirror is populated):

```sh
helm dependency build helm/sealed-secrets-web
helm template test helm/sealed-secrets-web \
  --set gatewayApi.enabled=true \
  --set gatewayApi.hostedZoneName=example.com
```

## Install

Deployed as an ArgoCD `Application` pointing at the OCI chart:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets-web
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: docker.io/alphaprosoft
    chart: sealed-secrets-web-helm
    targetRevision: 1.X-main
    helm:
      releaseName: sealed-secrets-web
      values: |
        gatewayApi:
          enabled: true
          hostedZoneName: example.com
  destination:
    server: https://kubernetes.default.svc
    namespace: sealed-secrets-web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

## CI secrets / variables

- `secrets.DOCKER_PUSH_USERNAME` — DockerHub username; doubles as the
  namespace when `DOCKER_PUSH_URL` is host-only.
- `secrets.DOCKER_PUSH_PASSWORD` — DockerHub access token.
- `vars.DOCKER_PUSH_URL` — registry. Either host-only (`docker.io`) or
  host+namespace (`docker.io/myorg`). Host-only means artifacts push to
  `docker.io/{username}/{name}`.
