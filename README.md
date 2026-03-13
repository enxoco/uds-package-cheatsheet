# UDS / Zarf / Pepr — The Missing Handbook

Quick-reference cheat sheets for developers working with [UDS](https://uds.defenseunicorns.com), [Zarf](https://zarf.dev), and [Pepr](https://pepr.dev). Not a replacement for official docs — just some things that weren't obvious to me getting started and took time to figure out.

For a single-page command reference, see [CHEATSHEET.md](CHEATSHEET.md).

---

## Terminology

These three terms overlap in naming but are distinct concepts:

| Term | Kind | What it is |
|------|------|-----------|
| **Zarf package** | `ZarfPackageConfig` (`zarf.yaml`) | Deployment artifact — bundles images, Helm charts, and manifests into a portable `.tar.zst` for air-gapped delivery |
| **UDS Bundle** | `UDSBundle` (`uds-bundle.yaml`) | Composition layer — groups multiple Zarf packages into a single deployable unit, wires variables across them |
| **UDS Package CR** | `kind: Package` (`uds-package.yaml`) | Live cluster resource — tells the UDS Core operator how to expose, protect, and network-policy a running workload |

A common source of confusion: "package" appears in both "Zarf package" and "UDS Package CR" but means completely different things. The Zarf package is a build artifact; the UDS Package CR is a Kubernetes object that exists in the cluster after deploy.

---

## Sections

| Section | What's inside |
|---|---|
| [Getting Started](getting-started/README.md) | Cluster bootstrap, DNS & TLS, Keycloak, bundled CLI tools |
| [Bundle Authoring](bundle-authoring/README.md) | Variable wiring across the bundle → zarf → helm stack, flavors, common/zarf.yaml |
| [UDS Package CR](uds-package-cr/README.md) | Writing `kind: Package` — network exposure, network policies, SSO clients, authservice protection |
| [Registry](registry/README.md) | Zarf internal registry operations — listing, pulling, pushing, copying images |
