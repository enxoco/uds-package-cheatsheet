# UDS Package CR Cheat Sheet

The `Package` custom resource (`kind: Package`, `apiVersion: uds.dev/v1alpha1`) is how you tell the UDS Core operator to manage a workload running in your cluster. It is separate from the Zarf package that deploys the workload — it is a live Kubernetes object that the UDS operator watches and acts on.

When you apply a `Package` CR, the operator:
- Creates Istio `VirtualService` and `ServiceEntry` resources for any declared `expose` entries
- Creates `NetworkPolicy` resources to enforce `allow` rules (all traffic is denied by default)
- Registers SSO clients in Keycloak for any declared `sso` entries
- Deploys authservice chains for any `sso` entries that use `enableAuthserviceSelector`

The CR typically lives in the same namespace as the workload and is usually applied as part of the Zarf package (as a manifest file).

---

## Minimal example

```yaml
apiVersion: uds.dev/v1alpha1
kind: Package
metadata:
  name: my-app
  namespace: my-app
spec: {}
```

An empty spec is valid — it just means "no exposure, no SSO, default-deny networking." Build from there.

---

## `network.expose` — Istio ingress

Exposes a service via one of the UDS Core Istio gateways. The operator creates a `VirtualService` and the associated `NetworkPolicy` to allow ingress traffic.

```yaml
spec:
  network:
    expose:
      - service: my-app          # name of the Kubernetes Service to expose
        selector:
          app: my-app            # pod label selector (used for NetworkPolicy)
        host: my-app             # subdomain — becomes my-app.uds.dev (tenant) or my-app.admin.uds.dev (admin)
        gateway: tenant          # 'tenant' (*.uds.dev) or 'admin' (*.admin.uds.dev)
        port: 443                # port the VirtualService listens on
        targetPort: 8080         # port on the Service/pod (defaults to port if omitted)
```

### Gateways

| Gateway | Domain pattern | Intended for |
|---------|---------------|--------------|
| `tenant` | `<host>.uds.dev` | End-user-facing services |
| `admin` | `<host>.admin.uds.dev` | Operator/admin UIs |
| `passthrough` | `<host>.uds.dev` | TLS passthrough (app terminates TLS itself) |

> **Tip:** To expose the same service on multiple paths or ports, add multiple entries to the `expose` list.

---

## `network.allow` — NetworkPolicy rules

All pod-to-pod and pod-to-external traffic is denied by default. Declare `allow` rules for every traffic flow your workload needs.

```yaml
spec:
  network:
    allow:
      # Allow egress to a database in another namespace
      - direction: Egress
        selector:
          app: my-app             # source pods
        remoteNamespace: postgres
        remoteSelector:
          app: postgres           # destination pods
        port: 5432
        description: "PostgreSQL database"

      # Allow egress to an external hostname (creates a ServiceEntry)
      - direction: Egress
        selector:
          app: my-app
        remoteGenerated: Anywhere # escape hatch — use sparingly
        description: "External API calls"

      # Allow ingress from another namespace
      - direction: Ingress
        selector:
          app: my-app
        remoteNamespace: monitoring
        remoteSelector:
          app: prometheus
        port: 9090
        description: "Prometheus scraping"
```

### `direction` values

| Value | Meaning |
|-------|---------|
| `Ingress` | Allow inbound traffic to the selected pods |
| `Egress` | Allow outbound traffic from the selected pods |

### `remoteGenerated` shortcuts

| Value | Meaning |
|-------|---------|
| `Anywhere` | Allow traffic to/from anywhere (no destination restriction) |
| `IntraNamespace` | Allow traffic within the same namespace |
| `KubeAPI` | Allow traffic to the Kubernetes API server |

> **Gotcha:** If you omit `allow` rules but your app makes outbound calls, those calls will silently fail. Start with `remoteGenerated: Anywhere` to unblock during development, then tighten to specific `remoteNamespace`/`remoteSelector` pairs before shipping.

---

## `sso` — Keycloak client registration

Registers an OIDC client in Keycloak. The operator creates the client and stores the client secret in a Kubernetes `Secret` in the same namespace.

```yaml
spec:
  sso:
    - name: My App                          # display name in Keycloak
      clientId: uds-my-app                  # client ID — must be unique across the realm
      redirectUris:
        - "https://my-app.uds.dev/oauth/callback"
      secretName: my-app-sso-secret        # optional — name of the Secret to create (defaults to sso-client-<clientId>)
```

The generated `Secret` contains `clientId` and `clientSecret` keys. Reference them in your deployment:

```yaml
env:
  - name: OIDC_CLIENT_ID
    valueFrom:
      secretKeyRef:
        name: my-app-sso-secret
        key: clientId
  - name: OIDC_CLIENT_SECRET
    valueFrom:
      secretKeyRef:
        name: my-app-sso-secret
        key: clientSecret
```

### Common SSO options

```yaml
spec:
  sso:
    - name: My App
      clientId: uds-my-app
      redirectUris:
        - "https://my-app.uds.dev/oauth/callback"
      # Optional fields:
      groups:                               # restrict login to specific Keycloak groups
        anyOf:
          - /UDS Core/Admin
      standardFlowEnabled: true            # authorization code flow (default true)
      publicClient: false                  # set true for SPAs that can't keep a secret
      rootUrl: "https://my-app.uds.dev"
```

---

## `enableAuthserviceSelector` — authservice protection

Normally an SSO client registration just creates the Keycloak client — your app must implement its own OIDC login flow. `enableAuthserviceSelector` changes this: the UDS operator deploys an [authservice](https://github.com/istio-ecosystem/authservice) chain that transparently intercepts requests and requires Keycloak authentication before they reach your pods. Your app does not need to handle OIDC at all.

```yaml
spec:
  sso:
    - name: My App
      clientId: uds-my-app
      redirectUris:
        - "https://my-app.uds.dev/oauth/callback"
      enableAuthserviceSelector:
        app: my-app                         # label selector — pods matching this get the authn proxy
```

### What happens

1. A request arrives at `my-app.uds.dev`.
2. Istio's `EnvoyFilter` (configured by authservice) checks for a valid session cookie.
3. If no session exists, the user is redirected to `sso.uds.dev` (Keycloak) to authenticate.
4. After successful login, Keycloak redirects back with a code; authservice exchanges it for tokens and sets a session cookie.
5. The request is forwarded to your app — which never sees the OIDC exchange at all.

### When to use it

| Situation | Recommendation |
|-----------|---------------|
| App has no built-in auth and you want SSO protection | Use `enableAuthserviceSelector` |
| App implements its own OIDC flow | Omit `enableAuthserviceSelector`, use the SSO secret directly |
| App is an API consumed by other services (not a browser UI) | Omit — authservice is session-cookie-based and not suited for M2M flows |

> **Note:** The pod label in `enableAuthserviceSelector` must match the pods that serve the exposed traffic. If the selector doesn't match any running pods, authservice will be configured but requests won't be intercepted.

---

## Full worked example

A complete `Package` CR wiring together exposure, network policies, and authservice-protected SSO:

```yaml
apiVersion: uds.dev/v1alpha1
kind: Package
metadata:
  name: my-app
  namespace: my-app
spec:
  network:
    expose:
      - service: my-app
        selector:
          app: my-app
        host: my-app
        gateway: tenant
        port: 443
        targetPort: 8080

    allow:
      - direction: Egress
        selector:
          app: my-app
        remoteNamespace: postgres
        remoteSelector:
          app: postgres
        port: 5432
        description: "PostgreSQL"

      - direction: Egress
        selector:
          app: my-app
        remoteNamespace: monitoring
        remoteSelector:
          app: prometheus
        port: 9090
        description: "Metrics scrape (outbound to prometheus pushgateway)"

  sso:
    - name: My App
      clientId: uds-my-app
      redirectUris:
        - "https://my-app.uds.dev/oauth/callback"
      enableAuthserviceSelector:
        app: my-app
```

With this CR applied:
- `https://my-app.uds.dev` routes to the `my-app` service on port 8080
- Unauthenticated requests are redirected to Keycloak; authenticated sessions pass through
- Pods can reach the `postgres` namespace on 5432, and push metrics on 9090; all other outbound traffic is blocked
- A Keycloak client `uds-my-app` is registered; the client secret is in a `Secret` in the `my-app` namespace

---

## Quick reference

| Task | Command |
|------|---------|
| List all Package CRs | `kubectl get packages -A` |
| Describe a Package CR | `kubectl describe package <name> -n <namespace>` |
| Watch Package CR reconciliation | `kubectl get package <name> -n <namespace> -w` |
| List VirtualServices created by UDS | `kubectl get vs -A` |
| List NetworkPolicies created by UDS | `kubectl get netpol -A` |
| Check SSO client secret | `kubectl get secret sso-client-<clientId> -n <namespace> -o jsonpath='{.data.clientSecret}' \| base64 -d` |
