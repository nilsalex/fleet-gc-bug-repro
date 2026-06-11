# Fleet GC bug reproduction

Minimal reproduction for a bug in Fleet's garbage collector where a valid helm
release is repeatedly uninstalled when the `BundleDeployment` name exceeds 53
characters.

## Bug summary

When a bundle's `fleet.yaml` sets an explicit `name:` field longer than 53
characters, Fleet's agent deploys the helm release under a truncated name (via
`names.HelmReleaseName()`), but the garbage collector computes the expected
release name from the raw `BundleDeployment` name without applying the same
truncation. The resulting mismatch causes the GC to treat the release as
unknown and uninstall it on every GC interval.

The agent logs the following repeatedly:

```
{"level":"info","logger":"bundledeployment.garbage-collect",
 "msg":"Deleting unknown bundle ID, helm uninstall",
 "bundleID":"repro-bundle-name-that-exceeds-the-53-char-helm-limit-x",
 "release":"default/repro-bundle-name-that-exceeds-the-53-char-helm-98fdb",
 "expectedRelease":"default/repro-bundle-name-that-exceeds-the-53-char-helm-limit-x"}
```

## Repository structure

```
manifests/                         <- affected bundle
  fleet.yaml                       <- explicit name: >53 chars
  templates/
    configmap.yaml

manifests-unaffected/              <- unaffected bundle (no explicit name)
  fleet.yaml
  templates/
    configmap.yaml
```

## Prerequisites

- `docker`
- `k3d` v5.8.3
- `helm`
- `kubectl`

## Reproduction steps

```bash
# 1. Create a k3d cluster
k3d cluster create fleet-bug-repro --servers 1 --network fleet

# 2. Install Fleet v0.14.3
host=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' k3d-fleet-bug-repro-server-0)
ca=$(kubectl config view --flatten -o jsonpath='{.clusters[?(@.name=="k3d-fleet-bug-repro")].cluster.certificate-authority-data}' | base64 -d)
helm repo add fleet https://rancher.github.io/fleet-helm-charts/
helm upgrade --install fleet-crd fleet/fleet-crd --version 0.14.3 \
  -n cattle-fleet-system --create-namespace --wait
helm upgrade --install fleet fleet/fleet --version 0.14.3 \
  -n cattle-fleet-system --wait \
  --set apiServerCA="$ca" \
  --set apiServerURL="https://$host:6443" \
  --set bootstrap.agentNamespace=cattle-fleet-local-system \
  --set garbageCollectionInterval=5s \
  --set debug=true --set debugLevel=1

# 3. Create a GitRepo pointing at this repository
kubectl apply -f - <<EOF
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: gc-bug-repro
  namespace: fleet-local
spec:
  repo: https://github.com/nilsalex/fleet-gc-bug-repro
  branch: main
  paths:
    - manifests
    - manifests-unaffected
EOF

# 4. Wait for both bundles to deploy (~30s), then watch the GC logs
kubectl -n cattle-fleet-local-system logs deployment/fleet-agent -f | grep "garbage-collect"
```

## Expected behaviour

Both ConfigMaps remain stable. No `garbage-collect` log lines appear.

## Actual behaviour

`fleet-gc-bug-repro` (from the affected bundle) is uninstalled and redeployed
repeatedly every few seconds. `fleet-gc-bug-repro-unaffected` (from the bundle
without an explicit name) is stable.

```
affected:   EXISTS (age=4s)  |  unaffected: EXISTS (age=51s)
affected:   EXISTS (age=2s)  |  unaffected: EXISTS (age=59s)
affected:   EXISTS (age=5s)  |  unaffected: EXISTS (age=67s)
affected:   EXISTS (age=0s)  |  unaffected: EXISTS (age=75s)
```

## Root cause

`releaseKey()` in
`internal/cmd/agent/deployer/cleanup/cleanup.go` computes the expected release
name as `namespace + "/" + bd.Name`. When deploying, `getOpts()` in
`internal/helmdeployer/deployer.go` passes the bundle ID through
`names.HelmReleaseName()`, which truncates names longer than 53 characters and
appends a 5-character hash. The two values diverge for any bundle whose
`fleet.yaml` sets a `name:` field exceeding 53 characters.

## Fix

Apply `names.HelmReleaseName()` in `releaseKey()` when no explicit
`helm.releaseName` is set, mirroring what `getOpts()` does:

```go
// Before
return ns + "/" + bd.Name

// After
return ns + "/" + names.HelmReleaseName(bd.Name)
```

PR with fix: https://github.com/nilsalex/fleet/pull/new/fix/cleanup-helm-release-name-truncation
