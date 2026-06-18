# selenosis-deploy

**Helm chart that deploys the full [Selenosis](https://github.com/alcounit/selenosis) stack on Kubernetes** — CRDs, RBAC, all services, and ingress, in one command.

[![Helm Chart Version](https://img.shields.io/github/v/release/alcounit/selenosis-deploy?label=helm)](https://github.com/alcounit/selenosis-deploy/releases)
[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/selenosis)](https://artifacthub.io/packages/search?repo=selenosis)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](./LICENSE)

---

## What it deploys

| Component | Role |
| --- | --- |
| **[selenosis](https://github.com/alcounit/selenosis)** | Stateless Selenium / Playwright / MCP hub. |
| **[seleniferous](https://github.com/alcounit/seleniferous)** | Sidecar proxy inside each browser pod (added to pods via `BrowserConfig`). |
| **[browser-controller](https://github.com/alcounit/browser-controller)** | Operator that reconciles `Browser` / `BrowserConfig` CRDs into pods. |
| **[browser-service](https://github.com/alcounit/browser-service)** | REST + SSE facade over `Browser` and `BrowserConfig` resources. |
| **[browser-ui](https://github.com/alcounit/browser-ui)** | Web dashboard with live sessions + VNC. |

`Browser` and `BrowserConfig` CRDs ship as Helm templates (`templates/crds/`) and are
applied on every `helm install`/`upgrade`. Each service is configured through Helm values
that map to the environment variables documented in the individual project READMEs. For the
architecture, see [selenosis → How it works](https://github.com/alcounit/selenosis#how-it-works).

---

## Quick start

```bash
# 1. Add the Helm repository
helm repo add selenosis https://alcounit.github.io/selenosis-deploy/
helm repo update

# 2. Install the full stack
helm install selenosis selenosis/selenosis-deploy -n selenosis --create-namespace

# 3. Apply a ready-made BrowserConfig (defines which browser images to run)
kubectl apply -n selenosis \
  -f https://raw.githubusercontent.com/alcounit/selenosis-deploy/main/examples/browserconfig-selenium-standalone-chrome-example.yaml
```

<details>
<summary><b>Install from the git repository instead</b></summary>

```bash
git clone https://github.com/alcounit/selenosis-deploy.git
cd selenosis-deploy
helm upgrade --install selenosis . -n selenosis --create-namespace --wait
helm status selenosis -n selenosis
```

</details>

---

## CRD management

CRDs are part of the chart templates, controlled by the `crds` values block:

```yaml
crds:
  enabled: true  # set to false to skip CRD installation (managed externally)
  keep: true     # adds helm.sh/resource-policy: keep — CRDs survive `helm uninstall`
```

```bash
# Install / upgrade without touching CRDs
helm upgrade --install selenosis selenosis/selenosis-deploy -n selenosis --set crds.enabled=false
```

> Because CRDs are templated (not in the legacy `crds/` directory), `helm upgrade` updates
> them when the schema changes. Migrating from an older `crds/`-based chart? See the
> [release notes](https://github.com/alcounit/selenosis-deploy/releases).

---

## BrowserConfig examples

Ready-to-use `BrowserConfig` manifests live in [`examples/`](https://github.com/alcounit/selenosis-deploy/tree/main/examples). Apply any after deploying the chart:

```bash
kubectl apply -n selenosis -f ./examples/<filename>.yaml
```

<details>
<summary><b>Per-image families: Selenoid, Selenium Standalone, Moon, Playwright, Playwright MCP</b></summary>

### Selenoid (`twilio/selenoid`)
Community-maintained Selenoid image family. VNC is built in, enabled via `ENABLE_VNC=true`.
Minimal two-container setup: browser + seleniferous sidecar.
- [chrome](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-selenoid-twilio-chrome-example.yaml) · [firefox](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-selenoid-twilio-firefox-example.yaml)
```bash
helm upgrade selenosis . -n selenosis --set browserUI.vncPassword="selenoid"
```

### Selenium Standalone (`selenium/standalone`)
Official Selenium standalone images with a built-in VNC server (`SE_VNC_PASSWORD`).
Minimal two-container setup: browser + seleniferous sidecar.
- [chrome](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-selenium-standalone-chrome-example.yaml) · [firefox](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-selenium-standalone-firefox-example.yaml)
```bash
helm upgrade selenosis . -n selenosis --set browserUI.vncPassword="${se_vnc_password}"
```

### Moon (`quay.io/browser`)
Moon images via `quay.io/browser`. VNC needs a full X11 sidecar stack (`xvfb`, `openbox`,
`x11vnc` from `quay.io/aerokube`) plus a `usergroup` ConfigMap (included) mapping the
`user:4096` identity.
- [chrome](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-moon-chrome-example.yaml) · [firefox](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-moon-firefox-example.yaml)
- Playwright CDP/BiDi variants: [chrome](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-moon-playwright-chrome-example.yaml) · [firefox](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-moon-playwright-firefox-example.yaml)
```bash
helm upgrade selenosis . -n selenosis --set browserUI.vncPassword="selenoid"
```

### Playwright Standalone (`mcr.microsoft.com/playwright`)
Official Microsoft Playwright base image (Chromium, Firefox, WebKit). `playwright-core` is
installed via an init container; `run-server` starts a multi-browser WebSocket server and
the client picks the browser at connect time.
- [example](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-playwright-example.yaml)

### Playwright MCP (`mcr.microsoft.com/playwright/mcp`)
Microsoft Playwright MCP server image — browser automation over MCP Streamable HTTP,
built-in MCP server, no init container.
- [example](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-playwright-mcp-example.yaml)

</details>

---

## Service types & ingress

Each service (`selenosis`, `browserService`, `browserUI`) supports `ClusterIP`,
`NodePort`, or `LoadBalancer`, and `selenosis`/`browserUI` each have their own ingress
block.

<details>
<summary><b>Service type values</b></summary>

```yaml
browserUI:
  service:
    type: NodePort
    port: 8080
    nodePort: 30080

browserService:
  service:
    type: LoadBalancer
    port: 8080

selenosis:
  service:
    type: ClusterIP
    port: 4444
```

```bash
helm upgrade --install selenosis . -n selenosis -f values.local.yaml
```

</details>

<details>
<summary><b>Ingress with TLS and WebSocket (NGINX)</b></summary>

Browser UI **requires WebSocket support** for the VNC proxy — the annotations below are
required for the NGINX Ingress Controller (other controllers differ).

```yaml
selenosis:
  ingress:
    enabled: true
    className: nginx
    host: selenosis.example.com
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    tls:
      secretName: selenosis-tls

browserUI:
  ingress:
    enabled: true
    className: nginx
    host: ui.example.com
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
      nginx.ingress.kubernetes.io/websocket-services: "browser-ui"
    tls:
      secretName: browser-ui-tls
```

```bash
helm upgrade --install selenosis . -n selenosis -f ingress-values.yaml
```

</details>

---

## Maintainer release process

<details>
<summary><b>How to cut a new chart release</b></summary>

1. Bump `version` in `Chart.yaml`.
2. Commit and push.
3. Tag and push: `git tag vX.Y.Z && git push origin vX.Y.Z`.
4. GitHub Actions lints, packages, creates the GitHub Release, and publishes to the Helm
   repository (GitHub Pages).
5. Users upgrade: `helm repo update && helm upgrade selenosis selenosis/selenosis-deploy --version X.Y.Z`.

</details>

---

## License

[Apache-2.0](./LICENSE)
