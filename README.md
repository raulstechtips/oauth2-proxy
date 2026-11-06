# OAuth2 Proxy wrapper chart

## Purpose

Install upstream oauth2-proxy chart 8.5.0, application 7.13.0, without changes to its
templates. Add two InfisicalStaticSecrets and the existing Traefik IngressRoute.
The Service remains `oauth2-proxy` on port `80`. Traefik owns the `oauth-signin`
and `oauth-verify` middlewares that use this Service.

## Before first install

1. Make sure the Infisical operator, its v1beta1 StaticSecret CRD, Traefik, and
   `default/infisical-prod-universal-auth` exist.
2. Make sure the Infisical project has all 11 mapped keys under `/oauth2/config`.
   Make sure `prod-raulstechtips-tls` is available in namespace `oauth2-proxy`.
3. Publish the chart to GHCR through CI. Set `targetRevision` in
   `cluster-config/argo/dev/oauth2-proxy.yaml` to the first CI chart version.
4. Complete the orphan-delete steps in Adoption notes before the first manual sync.
5. Sync the Application manually. Check that both StaticSecrets have synced,
   the Deployment is healthy, and sign-in and forward authentication work.

## Values reference

| Key | Use |
| --- | --- |
| `infisical.authRef` | Existing InfisicalAuth name and namespace |
| `infisical.projectSlug`, `environmentSlug` | Environment connection |
| `clientSecret`, `configSecret` | Enabled switch, target name, path, recursive flag, refresh interval, key mapping |
| `ingressRoute` | Enabled switch, host, TLS Secret, entry points, class, middlewares |
| `oauth2-proxy` | Values passed unchanged to the dependency |

Wrapper features are off by default. `cluster-config/apps/dev/oauth2-proxy.yaml`
enables them and supplies environment values. No secret values are stored in these
files. The CI values use fake connection names and an `example.test` host.

The dependency values contain only differences from upstream defaults. The default
image already resolves to `quay.io/oauth2-proxy/oauth2-proxy:v7.13.0`. Default
Service settings, metrics, autoscaling, session storage, and deprecation checks
are not repeated.

## Upgrades

Change the pinned dependency version in `charts/Chart.yaml`, run
`/opt/homebrew/bin/helm dependency update charts`, and retain `charts/Chart.lock`.
Review the upstream default image and all changed defaults. Run the wrapper checks
with an isolated `TMPDIR`, then render with the apps/dev values and compare to live.
CI publishes to GHCR and updates the Argo chart version. Sync remains manual.

## Adoption notes

The local evidence is in `.omo/evidence/wrappers/oauth2-proxy/` in the HomeLab
workspace. It includes the complete check output, development render, parity
report, and decisions. No cluster changes are part of this work.

Before the first sync, the operator must remove the old controllers with orphan
deletion so that the existing Kubernetes Secrets remain:

```sh
kubectl delete infisicalsecret infisical-secret-oauth2-proxy-client-secret -n oauth2-proxy --cascade=orphan
kubectl delete infisicalsecret infisical-secret-oauth2-proxy-config-secret -n oauth2-proxy --cascade=orphan
```

Do not sync the old Application source again after these steps. The new
InfisicalStaticSecrets use the existing Secret names, the same 3 client keys and
8 configuration keys, the same recursive `/oauth2/config` source, Opaque type,
and a 60-second refresh. `creationPolicy: Orphan` replaces the old Owner policy.
The shared InfisicalAuth replaces inline universal-auth references.

Expected parity differences are two NEW StaticSecrets, two PRUNE old
InfisicalSecrets, and wrapper labels added to the existing IngressRoute. Preserve
the exact live route match, including its host-independent `/oauth2/` alternative.
Four upstream objects are SAME. The Deployment is reported CHANGED only because
both probe `initialDelaySeconds` fields render as `0` but are absent from the
live API response. Upstream 8.5.0 always emits these fields while probes are
enabled. This is a zero-value field-presence difference, not a probe behavior
change. Keep the valid upstream defaults; do not disable probes, use null values,
or fork templates to hide the report.

The recorded summary is `SAME=4 CHANGED=2 NEW=2 PRUNE=2 HOOK=0 UNKNOWN-KIND=0`.
The IngressRoute CHANGED entry contains only the three added
`app.kubernetes.io/*` labels. The two NEW objects are
`oauth2-proxy-client-secret` and `oauth2-proxy-config-secret` StaticSecrets.
The two PRUNE entries are the old `infisical-secret-oauth2-proxy-client-secret`
and `infisical-secret-oauth2-proxy-config-secret` InfisicalSecrets.

The new Application keeps the live name, release name, namespace, and history
limit. It uses project `platform`, GHCR, and main-branch environment values.
The live Application has no sync options or ignore rules to retain. This
dependency has no `crds/` directory, so `helm.skipCrds` is not needed.
