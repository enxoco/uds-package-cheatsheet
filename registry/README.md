# Zarf Internal Registry Cheat Sheet

Zarf ships with an internal OCI-compliant container registry deployed into the cluster. This is used to store images in air-gapped environments so workloads don't need to reach the internet.

All examples use `uds zarf tools registry` (a bundled crane-based tool). When run without a server address, it automatically discovers the registry address and credentials from the `zarf-state` secret — **no login, no port-forward, and no address argument required** for most operations.

---

## End-to-end example: inspect and pull an image

`library/registry` (the Docker registry image) is present in all UDS Core installations, making it a reliable image to use for testing registry operations.

**1. Confirm the image is present**
```shell
$ uds zarf tools registry catalog
...
library/registry
...
```

**2. List available tags**
```shell
$ uds zarf tools registry ls library/registry
2.8.3
```

**3. Inspect the digest**
```shell
$ uds zarf tools registry digest library/registry:2.8.3
sha256:3c7ca5d56ebbf0d446633decd5a97b31e7c0613a07e1b4f5c5e3d9eba5f4a0c8
```

**4. Pull the image to a local tarball**
```shell
$ uds zarf tools registry pull library/registry:2.8.3 registry.tar
```

The result is an OCI-layout tarball you can inspect, re-push to another registry, or load into Docker with `docker load`.

---

## End-to-end example: build a local image and deploy it via a Zarf package

> **Production note:** Running custom or unvetted images in a cluster is discouraged in production environments. Prefer hardened, minimal base images from a trusted source, or explore alternatives like `kubectl debug` (ephemeral containers) before adding a custom image to the registry.

A common development/debugging need is a custom image — one with tools like `curl`, `dig`, `netcat`, `tcpdump`, etc. — that you can `kubectl exec` into alongside other workloads.

The correct way to deploy a custom image into a Zarf-managed cluster is through a Zarf package. Pushing directly to the registry bypasses Zarf's tag-transformation and credential-injection lifecycle, causing image pull failures. A Zarf package handles all of this automatically.

**1. Write a Dockerfile** (see [`Dockerfile`](Dockerfile) in this directory for a fuller example)
```dockerfile
FROM alpine:3.19
RUN apk add --no-cache curl bind-tools netcat-openbsd tcpdump bash jq
```

**2. Build the image locally**
```shell
docker build -t debug-tools:latest .
```

The image only needs to exist in the local Docker daemon — Zarf will pull it from there during package creation.

**3. Write a `zarf.yaml`** (see [`zarf.yaml`](zarf.yaml) in this directory)
```yaml
kind: ZarfPackageConfig
metadata:
  name: debug-tools
  version: 0.0.1

components:
  - name: debug-tools
    required: true
    images:
      - debug-tools:latest
    manifests:
      - name: debug-pod
        namespace: debug-tools
        files:
          - manifests/debug-pod.yaml
```

> **Do not use `namespace: default`**: The `default` namespace carries the label `zarf.dev/agent=ignore`, which prevents the `zarf-agent` mutating webhook from rewriting image references and injecting pull credentials. Pods deployed there will fail to pull from the internal registry. Always use a dedicated namespace (e.g. `debug-tools`) — Zarf creates it automatically.

**4. Write the pod manifest** (see [`manifests/debug-pod.yaml`](manifests/debug-pod.yaml))
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-tools
spec:
  containers:
    - name: debug-tools
      image: debug-tools:latest
      command: ["sleep", "infinity"]
  restartPolicy: Never
```

The image reference here uses the short form (`debug-tools:latest`). The `zarf-agent` webhook rewrites it to the full internal registry address and appends a deterministic hash suffix at deploy time — this is expected Zarf behavior.

**5. Create the Zarf package**
```shell
uds zarf package create . --confirm
# produces zarf-package-debug-tools-amd64-0.0.1.tar.zst
```

This pulls `debug-tools:latest` from the local Docker daemon and bundles it into a self-contained tarball.

> **Expected warning during package create:** You may see a `WRN unable to find image, attempting pull from docker daemon as fallback` message referencing `docker.io/library/debug-tools`. This is normal — Zarf first tries to fetch the image from a remote registry, fails (the image doesn't exist on Docker Hub), then falls back to the local Docker daemon where the image does exist. The package is created successfully regardless.


**6. Deploy the package**
```shell
uds zarf package deploy zarf-package-debug-tools-*.tar.zst --confirm
```

Zarf pushes the image to the internal registry with the correct tag, deploys the pod manifest, and the `zarf-agent` webhook handles image reference rewriting and pull credential injection automatically.

**7. Exec into the pod**
```shell
kubectl exec -n debug-tools -it debug-tools -- bash
```

**8. Clean up**
```shell
kubectl -n debug-tools delete pod debug-tools
```

---

## Listing images

### List all repositories

```shell
uds zarf tools registry catalog
```

### List tags for a specific image

```shell
uds zarf tools registry ls <image-name>
```

Example:
```shell
uds zarf tools registry ls library/nginx
```

### Inspect an image's digest

```shell
uds zarf tools registry digest <image-name>:<tag>
```

---

## Pulling images

Pulls the image as an OCI tarball — useful for inspection or re-pushing elsewhere:

```shell
uds zarf tools registry pull <image>:<tag> <output-file.tar>
```

---

## Pushing images

Push an OCI tarball into the registry (the format produced by `docker save` or `uds zarf tools registry pull`).

> **Important:** The destination must include the full `host:port` prefix (e.g. `127.0.0.1:31999`). Without it the tool pushes to Docker Hub instead.

```shell
uds zarf tools registry push <local-image.tar> 127.0.0.1:31999/<image>:<tag>
```

---

## Copying images between registries

`registry copy` mirrors an image from one registry to another without pulling it locally first. Useful for seeding the internal registry from an external source:

```shell
uds zarf tools registry copy <source-image>:<tag> <image>:<tag>
```

Example — copy from Docker Hub:
```shell
uds zarf tools registry copy docker.io/library/nginx:latest library/nginx:latest
```

---

## Finding the registry address manually

If you need the registry address or credentials directly (e.g. for use with another tool), they are stored in the `zarf-state` secret:

```shell
kubectl get secret -n zarf zarf-state -o jsonpath='{.data.state}' | base64 -d | jq '.registryInfo'
```

Output:
```json
{
  "address": "127.0.0.1:31999",
  "internalRegistry": true,
  "nodePort": 31999,
  "pullPassword": "<pull-password>",
  "pullUsername": "zarf-pull",
  "pushPassword": "<push-password>",
  "pushUsername": "zarf-push"
}
```

The registry NodePort is directly accessible — no port-forward needed.

---

## Quick reference

| Task | Command |
|------|---------|
| List all images | `uds zarf tools registry catalog` |
| List tags for image | `uds zarf tools registry ls <image>` |
| Get image digest | `uds zarf tools registry digest <image>:<tag>` |
| Pull image to tarball | `uds zarf tools registry pull <image>:<tag> <output.tar>` |
| Push tarball to registry | `uds zarf tools registry push <input.tar> <image>:<tag>` |
| Copy image from external registry | `uds zarf tools registry copy <src> <image>:<tag>` |
| Get registry address & credentials | `kubectl get secret -n zarf zarf-state -o jsonpath='{.data.state}' \| base64 -d \| jq '.registryInfo'` |
