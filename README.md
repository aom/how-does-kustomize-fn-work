# How to update a custom image reference and pass it to Pod

Say we have a custom project `example-nginx` that we need to update to development and integration clusters from a Continuous Delivery service (ArgoCD). Image itself is not run so `kustomize set image ...` does not work. There is a gitops repository that contains latest versions and ArgoCD is hooked to that repo.

## Method 1: Use file as configMap generator

File with custom project name `example-nginx` exists in all sub-directories that need it. File contains the latest tag name. Ie. `container-registry.com/namespace/example-nginx:v1.0.0`.

kustomization.yaml

```
configMapGenerator:
- name: example-nginx
  files:
  - example-nginx
```

deployment.yaml

```
spec:
  template:
    spec:
      containers:
      - env:
          - name: EXAMPLE_NGINX_IMAGE
            valueFrom:
              configMapKeyRef:
                name: example-nginx
                key: example-nginx
```

CI finds any files that match exactly the project image identifier and replaces contents with latest tag. Change is committed and pushed

```
export IMAGE_IDENTIFIER=example-nginx
export IMAGE_TAG=container-registry.com/namespace/example-nginx:v5.0.0
find cluster/services/nginx/dev/ -name ${IMAGE_IDENTIFIER} -type f -print0 | xargs -0 sed -i.bak "s~.*~${IMAGE_TAG}~"
find cluster/services/nginx/dev/ -name ${IMAGE_IDENTIFIER}.bak -type f | xargs rm
```

Bak file is not strictly necessary unless this would ever be run with non GNU sed

ArgoCD notices push and syncs application manifests with CMP including Kustomize and we get ConfigMap with a new tag

```
..
kind: ConfigMap
data:
  example-nginx: container-registry.com/namespace/example-nginx:v5.0.0
metadata:
  name: example-nginx-9695fhgb8m
  namespace: example-dev
---
apiVersion: apps/v1
kind: Deployment
..
spec:
  template:
    spec:
      containers:
      - env:
        - name: EXAMPLE_NGINX_IMAGE
          valueFrom:
            configMapKeyRef:
              key: example-nginx
              name: example-nginx-9695fhgb8m
..
```

## Method 2: Use kustomize functions

ConfigMap with custom project images exists in all sub-directories that need it

images.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: images
data:
  example-nginx: container-registry.com/namespace/example-nginx:v1.0.0
```

deployment.yaml

```
spec:
  template:
    spec:
      containers:
      - env:
          - name: EXAMPLE_NGINX_IMAGE
            valueFrom:
              configMapKeyRef:
                name: images
                key: example-nginx
```

CI runs a kustomize function if `images.yaml` exists and replaces the tag. Change is committed and pushed

```
export IMAGE_IDENTIFIER=example-nginx
export IMAGE_TAG=container-registry.com/namespace/example-nginx:v5.0.0
if [ -f cluster/services/nginx/int/images.yaml ]; then; kustomize fn run cluster/services/nginx/int/images.yaml --enable-exec --exec-path ./bin/image-updater-kustomize-fn; else; echo "No images.yml"; fi
```

ArgoCD notices push and syncs application manifests with CMP including Kustomize and we get ConfigMap with a new tag

```
data:
  example-nginx: container-registry.com/namespace/example-nginx:v5.0.0
kind: ConfigMap
metadata:
  name: images
  namespace: example-int
---
apiVersion: apps/v1
kind: Deployment
..
spec:
  template:
    spec:
      containers:
      - env:
        - name: EXAMPLE_NGINX_IMAGE
          valueFrom:
            configMapKeyRef:
              key: example-nginx
              name: images
..
```
