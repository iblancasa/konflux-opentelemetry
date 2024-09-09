# konflux-opentelemetry

This repository contains Konflux configuration to build Red Hat build of OpenTelemetry.

## Build locally

```bash
docker login brew.registry.redhat.io -u
docker login registry.redhat.io -u

podman build -t docker.io/user/otel-operator:$(date +%s) -f Dockerfile.operator 
```

## Release

Open PR `Release - update bundle version` and update [patch_csv.yaml](./bundle-patch/patch_csv.yaml) by submitting a PR with follow-up changes:
1. `spec.version` with the current version e.g. `opentelemetry-operator.v0.108.0`
1. `spec.name` with the current version e.g. `opentelemetry-operator.v0.108.0`
1. `spec.replaces` with [the previous shipped version](https://catalog.redhat.com/software/containers/rhosdt/opentelemetry-operator-bundle/615618406feffc5384e84400) of CSV e.g. `opentelemetry-operator.v0.107.0-4`
1. `metadata.extra_annotations.olm.skipRange` with the version being productized e.g. `'>=0.33.0 <0.108.0'`

Once the PR is merged and bundle is built. Open another PR `Release - update catalog` with:
 * Updated [catalog template](./catalog/catalog-template.json) with the new bundle (get the bundle pullspec from [Konflux](https://console.redhat.com/application-pipeline/workspaces/rhosdt/applications/otel/components/otel-bundle)):
    ```bash
    opm alpha render-template basic catalog/catalog-template.json > catalog/opentelemetry-product/catalog.json && \
    opm validate catalog/opentelemetry-product/ 

    # This does not generate valid catalog, e.g. it is smaller and missing relatedImages
    docker run --rm -it -v $(pwd)/catalog:/tmp:Z  --entrypoint /bin/bash registry.redhat.io/openshift4/ose-operator-registry-rhel9:v4.16
    opm alpha render-template basic /tmp/catalog-template.json > /tmp/opentelemetry-product/catalog-ose-operator.json
    ```

(TODO verify) After konflux builds the bundle create one more PR to change the registry to `registry.redhat.io` see https://konflux-ci.dev/docs/advanced-how-tos/releasing/maintaining-references-before-release/

## Test locally

Images can be found at https://quay.io/organization/redhat-user-workloads

### Deploy bundle

```bash
operator-sdk olm install 
operator-sdk run bundle quay.io/redhat-user-workloads/rhosdt-tenant/otel/otel-bundle@sha256:a09e1fa7c42b3f89b8a74e83d9d8c5b501ef9cd356612d6e146646df1f3d5800
operator-sdk cleanup opentelemetry-product
```

### Deploy catalog

Get catalog for specific version from [Konflux](https://console.redhat.com/application-pipeline/workspaces/rhosdt/applications/otel-fbc-v4-15/components/otel-fbc-v4-15)

```yaml
kubectl apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
   name: konflux-catalog-otel
   namespace: openshift-marketplace
spec:
   sourceType: grpc
   image: quay.io/redhat-user-workloads/rhosdt-tenant/otel/otel-fbc-v4-15@sha256:337009c69204eed22bd90acf5af45f3db678bd65531c8847c59e9532f8427d29
   displayName: Konflux Catalog OTEL
   publisher: grpc
EOF

kubectl get pods -w -n openshift-marketplace
kubectl delete CatalogSource konflux-catalog-otel -n openshift-marketplace
```

`Konflux catalog OTEL` menu should appear in the OCP console under Operators->OperatorHub.

### Inspect bundle image

```bash
mkdir /tmp/bundle
docker image save -o /tmp/bundle/image.tar quay.io/redhat-user-workloads/rhosdt-tenant/otel/otel-bundle@sha256:193358e912cd6a1d06eacf27363d85f2082c21596084110f026f43682ca3cecf
tar xvf /tmp/bundle/image.tar -C /tmp/bundle
tar xvf /tmp/bundle/c6f6e1b5441a6acfc03bb40f4b2d47b98dcfca1761e77e47fba004653eb596d7/layer.tar -C /tmp/bundle/c6f6e1b5441a6acfc03bb40f4b2d47b98dcfca1761e77e47fba004653eb596d7
```