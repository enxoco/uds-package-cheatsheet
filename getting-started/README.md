## Getting Started with UDS Core
This doc serves as a sort of cheat sheet/living doc for me as I work through understanding the lay of the land with UDS Core, as well as creating/deploying my first package.  I will try to reference official docs sections as appropriate.

## Assumptions
This guide assumes that you are setting up a cluster for the first time, or redeploying UDS core from scratch.  To be clear something like the following:
```shell
$ uds deploy k3d-core-slim-dev:latest
```

This command will bootstrap a k3d cluster for you and install only basic components such as Keycloak, Istio and Pepr. [source](https://github.com/defenseunicorns/uds-core/tree/main/bundles/k3d-slim-dev)

### Local Networking & Traffic Flow
When you hit `https://sso.uds.dev` in a browser, the request travels through several layers:

1. **DNS** — `*.uds.dev` is a real public wildcard DNS record that resolves to `127.0.0.1`. No `/etc/hosts` edits needed.
2. **Host ports 80/443** — k3d maps these into the cluster via Docker. The Docker daemon (`dockerd`) binds the ports as root on your behalf; you never need `sudo` yourself.
3. **NGINX (inside k3d)** — receives the traffic and uses ACL rules to forward it to the correct [MetalLB](https://metallb.universe.tf/) load balancer IP.
4. **MetalLB** — provides a stable in-cluster IP for each Istio Gateway service.
5. **Istio Gateways** — terminate TLS (using bundled `*.uds.dev` wildcard certs) and route to the appropriate service via VirtualService rules.

Three gateways handle different domains:
- `*.uds.dev` — tenant gateway (user-facing apps, e.g. `sso.uds.dev`)
- `keycloak.uds.dev` — also tenant gateway
- `*.admin.uds.dev` — admin gateway (operator/admin UIs)

> **Port binding & Docker daemon privileges**
> Ports 80 and 443 are bound by the Docker daemon, not by `k3d` or your shell directly. The daemon gains the necessary privilege to bind these ports when it is installed/configured — for example, Colima requires `sudo` during setup for exactly this reason. Once that is in place, day-to-day `k3d` usage needs no elevated permissions.
>
> If cluster creation succeeds but you can't reach `*.uds.dev` in a browser, check:
> - The Docker daemon is running: `docker info`
> - On Linux, your user is in the `docker` group: `groups $USER`. If not: `sudo usermod -aG docker $USER`, then log out and back in.
> - Nothing else is already bound to port 80 or 443: `sudo lsof -i :80 -i :443`

### SSO Integration
UDS Core comes with Keycloak integration out of the box.

> **DNS & TLS tip:** `uds.dev` and all its subdomains (`sso.uds.dev`, `keycloak.admin.uds.dev`, etc.) are real public DNS records that resolve to `127.0.0.1`. UDS Core also bundles wildcard TLS certificates for `*.uds.dev`, so local k3d clusters work over HTTPS with valid certs out of the box — no `/etc/hosts` editing, no local DNS setup, and no cert warnings.  On a brand new deployment you will first need/want to create a user before signing into any services.  The operator will create an initial Keycloak admin user for you and store the credentials in a secret named `keycloak-admin-password` which resides in the keycloak namespace.  

#### Keycloak admin user/password
A quick way to get the initial admin password from the secret: (adding ; echo to the end of the command ensures that the shell will print a new line, otherwise you may see a rogue % on the end of the string.). The default username is admin, however you can check that by extracting the username from the same secret.


```shell
uds zarf tools kubectl get secret -n keycloak keycloak-admin-password -o jsonpath='{.data.password}' | base64 -d; echo

<your-admin-password>
```

**Alternate** At the time of this edit you can also simply run `setup:print-keycloak-admin-password` like so:
```shell
$ uds run setup:print-keycloak-admin-password
     !!! Please ensure you're not running this in CI !!!                                                                                                                                                                               
     Keycloak Admin Username:  admin
     Keycloak Admin Password:  <your-admin-password>                                                                                                                                                                                        
  ✔  Completed "Print the default keycloak admin password to standard out (if available)"
```

#### Creating a test user
If you need a user to test applications with, there is a task to boostrap the creation of a `doug@uds.dev` user.

```shell
$ uds run setup:keycloak-user --set KEYCLOAK_USER_GROUP="/UDS Core/Admin"
```
The command will create a user with the following credentials that can then be used to login to `https://sso.uds.dev`
```
username: doug@uds.dev
password: unicorn123!@#UN
```
[docs source](https://uds.defenseunicorns.com/tutorials/deploy-uds-on-rke2/#configuring-keycloak-sso)

Note: This command assumes that you have a `tasks.yaml` file in your working directory.  This file should be included by default with all UDS packages but if you don't have one, this is what it should look like:

```yaml
# Copyright 2024 Defense Unicorns
# SPDX-License-Identifier: AGPL-3.0-or-later OR LicenseRef-Defense-Unicorns-Commercial
# yaml-language-server: $schema=https://raw.githubusercontent.com/defenseunicorns/uds-cli/refs/heads/main/tasks.schema.json
includes:
  - test: ./tasks/test.yaml
  - create: https://raw.githubusercontent.com/defenseunicorns/uds-common/v1.24.1/tasks/create.yaml
  - lint: https://raw.githubusercontent.com/defenseunicorns/uds-common/v1.24.1/tasks/lint.yaml
  - pull: https://raw.githubusercontent.com/defenseunicorns/uds-common/v1.24.1/tasks/pull.yaml
  - deploy: https://raw.githubusercontent.com/defenseunicorns/uds-common/v1.24.1/tasks/deploy.yaml
  - setup: https://raw.githubusercontent.com/defenseunicorns/uds-common/v1.24.1/tasks/setup.yaml
  ###Truncated
```

  You can view the [tasks/setup.yaml](https://raw.githubusercontent.com/defenseunicorns/uds-common/v1.24.1/tasks/setup.yaml) file to see exactly what it is doing and view other convience wrappers.  


  With that user created, you should now be able to login to `https://sso.uds.dev` with that information.  If you need to login to the admin side of keycloak you will need to visit [https://keycloak.admin.uds.dev](https://keycloak.admin.uds.dev)

  ** Note ** To see a list of services that are available externally you can run:
  ```shell
    $ uds zarf tools kubectl get vs -A
NAMESPACE     NAME                                                                  GATEWAYS                                  HOSTS                        AGE
keycloak      keycloak-admin-admin-access-with-optional-client-certificate          ["istio-admin-gateway/admin-gateway"]     ["keycloak.admin.uds.dev"]   22h
keycloak      keycloak-tenant-public-auth-access-with-optional-client-certificate   ["istio-tenant-gateway/tenant-gateway"]   ["sso.uds.dev"]              22h
keycloak      keycloak-tenant-remove-private-paths-from-public-gateway              ["istio-tenant-gateway/tenant-gateway"]   ["sso.uds.dev"]              22h
  ```

  or alternatively `k get vs -A` (vs short for virtualservices)

### Quick port forwards
If you need to access a service via port-forward, zarf offers `zarf connect list` which will list the services that are available as well as the commands to connect.  For example:
```shell
zarf connect registry
```
will connect you to the internal zarf registry.

### Bundled CLI tools
Both `uds` and `zarf` ship with several tools bundled directly into the binary so you don't need to install them separately. This is especially useful in air-gapped environments. Access them via `uds zarf tools <tool>`.

Notable bundled tools:
- **`kubectl`** (`uds zarf tools kubectl`) — bundled Kubernetes CLI. Use this instead of your system `kubectl` to ensure the version matches your cluster.
- **`helm`** (`uds zarf tools helm`) — bundled Helm. Same version-pinning benefit as kubectl.
- **`registry`** (`uds zarf tools registry`) — crane-based OCI registry tool for pushing/pulling/copying images to the internal Zarf registry. Useful for air-gap workflows without Docker.
- **`sbom`** (`uds zarf tools sbom`) — Syft-based SBOM generator. Every Zarf package carries a machine-readable SBOM out of the box.
- **`monitor`** (`uds zarf tools monitor`) — launches a bundled k9s-style cluster dashboard. Immediate cluster visibility with no extra install.
- **`yq`** (`uds zarf tools yq`) — YAML/JSON query and edit tool. Handy for inspecting `zarf.yaml`, `uds-bundle.yaml`, and Helm values files in-place.
- **`wait-for`** (`uds zarf tools wait-for`) — blocks until a Kubernetes resource or URL reaches a ready state. Used in `onDeploy.after` actions for reliable sequencing.
- **`archiver`** (`uds zarf tools archiver`) — compress/decompress archives (tar, zip, zst) without needing system tools installed.
- **`clear-cache`** (`uds zarf tools clear-cache`) — clears the local Zarf build/image cache (`~/.zarf-cache`) to free disk space or force fresh pulls.
- **`gen-pki`** (`uds zarf tools gen-pki`) — generates a self-signed CA and TLS cert/key pair for bootstrapping internal services.
- **`gen-key`** (`uds zarf tools gen-key`) — generates a cosign-compatible signing key pair for package signing and verification.

To see the full list:
```shell
uds zarf tools --help
```

### `uds deploy` vs `uds zarf deploy`
These are easy to confuse:
- **`uds deploy`** — deploys a **UDS bundle** (a `.tar.zst` or OCI reference produced by `uds create`). This is the typical top-level entry point.
- **`uds zarf deploy`** — deploys an individual **Zarf package**. Bundles are composed of one or more Zarf packages under the hood.

If you're getting "not a valid bundle" or "not a valid package" errors, you're likely using the wrong command for what you built.

### Cluster teardown
To fully tear down the k3d cluster and start fresh:
```shell
k3d cluster delete
```
If you also want to clear the local Zarf package/image cache:
```shell
uds zarf tools clear-cache
```

### Pepr debugging
Pepr runs as an admission webhook and mutates/validates resources as they are applied. If a package deployment stalls or resources behave unexpectedly, Pepr is a common culprit. First steps:
```shell
# Check what webhooks are registered
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Check Pepr logs
kubectl logs -n pepr-system -l app=pepr-uds-core-watcher --tail=100
kubectl logs -n pepr-system -l app=pepr-uds-core --tail=100
```

#### Stuck namespaces after package removal
`zarf package remove <package-name>` will sometimes complete successfully but leave behind a namespace stuck in `Terminating` state. This typically happens when a pod or container failed to start cleanly and Pepr's finalizer annotation is still present on the namespace, blocking deletion.

To fix it, patch the namespace to remove the finalizer:
```shell
kubectl get namespace <stuck-namespace> -o json \
  | jq 'del(.spec.finalizers[] | select(. == "pepr.dev/uds"))' \
  | kubectl replace --raw "/api/v1/namespaces/<stuck-namespace>/finalize" -f -
```
Or edit it directly and remove the `pepr.dev/uds` entry from `.spec.finalizers`:
```shell
kubectl edit namespace <stuck-namespace>
```
After removing the finalizer the namespace should delete itself immediately.

### `uds run` and tasks.yaml
`uds run` always looks for `tasks.yaml` in the **current working directory**. Running it from a different directory will either fail silently or pick up a different project's tasks. Always confirm your CWD before running tasks.