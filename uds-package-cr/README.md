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

## Troubleshooting

### Check Package CR status

The quickest way to see whether the operator finished reconciling and whether anything failed:

```shell
kubectl describe package <name> -n <namespace>
```

Look at the `Status.Conditions` block. A healthy Package shows `Ready: True`. If reconciliation failed, the `Reason` and `Message` fields say why.

Watch reconciliation in real time:

```shell
kubectl get package <name> -n <namespace> -w
```

---

### Verify generated resources

When the operator reconciles a Package CR it creates several resources. If something isn't working, confirm each was actually created.

**NetworkPolicies** — one per `allow` rule, plus ingress for each `expose` entry:

```shell
kubectl get netpol -n <namespace>
kubectl describe netpol -n <namespace>
```

**VirtualServices** — one per `expose` entry:

```shell
kubectl get vs -n <namespace>
kubectl describe vs -n <namespace>
```

**AuthorizationPolicies** — Istio-level access control, created alongside NetworkPolicies:

```shell
kubectl get authorizationpolicies -n <namespace>
kubectl describe authorizationpolicies -n <namespace>
```

If any of these are missing after a successful reconcile, the operator likely didn't see the CR. See the namespace labeling issue below.

---

### Common issues

#### Package CR is ignored — namespace missing Pepr labels

UDS Core (Pepr) only manages namespaces that carry specific labels. Namespaces created via a Zarf component get these labels automatically through Pepr's namespace mutation webhook. If you deploy into an existing namespace (e.g. `default`) or one created manually, those labels will be absent and the Package CR will never be reconciled — no NetworkPolicies, VirtualServices, or SSO clients will be created.

Check what labels are present:

```shell
kubectl get namespace <namespace> --show-labels
```

The safest fix is to let Zarf create and own the namespace by declaring it in the component's `namespace:` field. This ensures the mutation webhook runs and the labels are applied.

Confirm the Pepr watcher saw the CR at all:

```shell
kubectl logs -n pepr-system -l app=pepr-uds-core-watcher --tail=100
```

---

#### App returns 503 after adding an `expose` entry

A 503 usually means the VirtualService was created but traffic can't reach the pod. Check in order:

1. **`targetPort` matches the container port** — `targetPort` in the `expose` entry must match what the container listens on, not the Service port
2. **NetworkPolicy allows the ingress** — `kubectl get netpol -n <namespace>` should show an ingress policy for the pod selector
3. **Service is selecting pods** — `kubectl get endpoints <service> -n <namespace>` should list pod IPs; an empty endpoints list means the Service selector doesn't match the pods

---

#### App can't reach an external service (silent failures / timeouts)

All egress is blocked by default. Missing `allow` rules cause connections to time out with no error in the app logs.

To unblock everything temporarily for debugging:

```yaml
allow:
  - direction: Egress
    selector:
      app: my-app
    remoteGenerated: Anywhere
    description: "Debug: allow all egress"
```

Then tighten to specific `remoteNamespace`/`remoteSelector` pairs once you know exactly what's needed.

---

#### authservice redirect loop or 401 after login

If `enableAuthserviceSelector` is set but users loop at the login page or get a 401 after authenticating:

1. **Selector mismatch** — the label in `enableAuthserviceSelector` must exactly match the pod labels. Check with `kubectl get pods -n <namespace> --show-labels`.
2. **Redirect URI mismatch** — the `redirectUris` in the `sso` entry must match what Keycloak registered for the client. Verify in the Keycloak admin console at `https://keycloak.admin.uds.dev`.
3. **SSO secret missing** — if the app also reads OIDC credentials directly, confirm the secret exists: `kubectl get secret -n <namespace>`.
