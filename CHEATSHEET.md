# UDS / Zarf / Pepr — Quick Reference

Single-page command reference. See each section's `README.md` for full explanations.

---

## Getting Started

| Task | Command |
|------|---------|
| Deploy k3d UDS Core (slim dev) | `uds deploy k3d-core-slim-dev:latest` |
| Tear down k3d cluster | `k3d cluster delete` |
| Clear Zarf build/image cache | `uds zarf tools clear-cache` |
| Get Keycloak admin password | `uds zarf tools kubectl get secret -n keycloak keycloak-admin-password -o jsonpath='{.data.password}' \| base64 -d; echo` |
| Get Keycloak admin password (task shortcut) | `uds run setup:print-keycloak-admin-password` |
| Create test SSO user (`doug@uds.dev`) | `uds run setup:keycloak-user --set KEYCLOAK_USER_GROUP="/UDS Core/Admin"` |
| List all exposed services (VirtualServices) | `kubectl get vs -A` |
| List available port-forward shortcuts | `zarf connect list` |
| Connect to internal registry via port-forward | `zarf connect registry` |

### Pepr debugging

| Task | Command |
|------|---------|
| List registered admission webhooks | `kubectl get mutatingwebhookconfigurations` |
| Pepr watcher logs | `kubectl logs -n pepr-system -l app=pepr-uds-core-watcher --tail=100` |
| Pepr admission logs | `kubectl logs -n pepr-system -l app=pepr-uds-core --tail=100` |
| Fix stuck namespace (remove Pepr finalizer) | `kubectl get namespace <ns> -o json \| jq 'del(.spec.finalizers[] \| select(. == "pepr.dev/uds"))' \| kubectl replace --raw "/api/v1/namespaces/<ns>/finalize" -f -` |

---

## Bundle & Package Deployment

| Task | Command |
|------|---------|
| Deploy a UDS bundle | `uds deploy <bundle>.tar.zst --confirm` |
| Deploy a Zarf package (standalone) | `uds zarf package deploy <pkg>.tar.zst --confirm` |
| Override a variable at deploy time | `uds deploy <bundle> --set db_host=prod.internal` |
| Override a variable scoped to a package | `uds deploy <bundle> --set my-package.db_host=prod.internal` |
| Override via environment variable | `UDS_DB_HOST=prod.internal uds deploy <bundle>` |

### Variable override precedence (highest → lowest)

```
--set flag  →  UDS_* env vars  →  uds-config.yaml  →  default in uds-bundle.yaml
```

---

## UDS Package CR

| Task | Command |
|------|---------|
| List all Package CRs | `kubectl get packages -A` |
| Describe a Package CR (check status/conditions) | `kubectl describe package <name> -n <namespace>` |
| Watch Package CR reconciliation | `kubectl get package <name> -n <namespace> -w` |
| List VirtualServices (one per `expose` entry) | `kubectl get vs -n <namespace>` |
| List NetworkPolicies (one per `allow` rule) | `kubectl get netpol -n <namespace>` |
| List AuthorizationPolicies (Istio-level access control) | `kubectl get authorizationpolicies -n <namespace>` |
| Check namespace labels (Pepr management) | `kubectl get namespace <namespace> --show-labels` |
| Check SSO client secret | `kubectl get secret -n <namespace> -o jsonpath='{.data.clientSecret}' <secret-name> \| base64 -d` |
| Check Service endpoints (is the selector matching pods?) | `kubectl get endpoints <service> -n <namespace>` |

---

## Registry

| Task | Command |
|------|---------|
| List all images in internal registry | `uds zarf tools registry catalog` |
| List tags for an image | `uds zarf tools registry ls <image>` |
| Get image digest | `uds zarf tools registry digest <image>:<tag>` |
| Pull image to OCI tarball | `uds zarf tools registry pull <image>:<tag> output.tar` |
| Push OCI tarball to registry | `uds zarf tools registry push input.tar 127.0.0.1:31999/<image>:<tag>` |
| Copy image from external registry | `uds zarf tools registry copy docker.io/library/nginx:latest library/nginx:latest` |
| Get registry address and credentials | `kubectl get secret -n zarf zarf-state -o jsonpath='{.data.state}' \| base64 -d \| jq '.registryInfo'` |
| Build local image into Zarf package | `uds zarf package create . --confirm` |
| Deploy Zarf package (pushes image to internal registry) | `uds zarf package deploy zarf-package-*.tar.zst --confirm` |

---

## Bundled CLI Tools (`uds zarf tools <tool>`)

| Tool | Purpose |
|------|---------|
| `kubectl` | Bundled Kubernetes CLI — version-matched to the cluster |
| `helm` | Bundled Helm |
| `registry` | crane-based OCI registry tool |
| `sbom` | Syft-based SBOM generator |
| `monitor` | k9s-style cluster dashboard |
| `yq` | YAML/JSON query and edit |
| `wait-for` | Block until a resource or URL is ready |
| `archiver` | Compress/decompress tar/zip/zst archives |
| `clear-cache` | Clear `~/.zarf-cache` |
| `gen-pki` | Generate self-signed CA + TLS cert/key pair |
| `gen-key` | Generate cosign-compatible package signing key pair |

```shell
uds zarf tools --help   # full list
```
