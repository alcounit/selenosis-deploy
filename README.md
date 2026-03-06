# selenosis-deploy Helm chart

## Architecture

![diagram](https://github.com/user-attachments/assets/c483699a-cf94-47a4-a2cf-448f21e39bf3)


## Summary

This chart deploys the full Selenosis stack:
- [selenosis](selenosis) — Selenium hub/proxy for creating sessions and routing traffic.
- [browser-controller](https://github.com/alcounit/browser-controller) — Kubernetes controller that reconciles `Browser` and `BrowserConfig` CRDs into Pods.
- [browser-service](https://github.com/alcounit/browser-service) — REST + event stream API for managing `Browser` resources.
- [browser-ui](https://github.com/alcounit/browser-ui) — UI + VNC WebSocket proxy backed by browser-service.

CRDs for `Browser` and `BrowserConfig` are managed as Helm template resources (`templates/crds/`) and applied on every `helm install` and `helm upgrade`. Installation can be disabled with `--set crds.enabled=false` if you manage CRDs outside of Helm.
Each service is configurable via Helm values that map to the environment variables described in the individual project READMEs.

## Installation

### From Helm Repository

Add the Selenosis Helm repository:

```sh
helm repo add selenosis https://alcounit.github.io/selenosis-deploy/
helm repo update
```

Install the chart:

```sh
helm install selenosis selenosis/selenosis-deploy -n selenosis --create-namespace
```

### From Git Repository

Clone and install directly:

```sh
git clone https://github.com/alcounit/selenosis-deploy.git
cd selenosis-deploy
helm upgrade --install selenosis . -n selenosis --create-namespace --wait
helm status selenosis -n selenosis
```

## CRD Management

CRDs are part of the chart templates and are controlled by the `crds` values block:

```yaml
crds:
  enabled: true  # set to false to skip CRD installation
  keep: true     # adds helm.sh/resource-policy: keep — CRDs are not deleted on helm uninstall
```

**Install without CRDs** (when CRDs are already applied or managed externally):

```sh
helm install selenosis selenosis/selenosis-deploy -n selenosis --create-namespace --set crds.enabled=false
```

**Upgrade without touching CRDs:**

```sh
helm upgrade selenosis selenosis/selenosis-deploy -n selenosis --set crds.enabled=false
```

> Unlike the legacy `crds/` directory approach, CRDs in templates are updated by `helm upgrade` when the schema changes. If you are upgrading from a previous chart version that used `crds/`, follow the [migration guide](https://github.com/alcounit/selenosis-deploy/releases).

## BrowserConfig examples

Ready-to-use `BrowserConfig` manifests are in `examples/`. Apply any of them after deploying the chart:

```sh
kubectl apply -n selenosis -f ./examples/<filename>.yaml
```

### Selenoid (twilio/selenoid)

Images from the Twilio-maintained Selenoid image family. VNC is built into the image and enabled via the `ENABLE_VNC=true` env var. Minimal two-container setup: browser + seleniferous sidecar.

- [browserconfig-selenoid-twilio-chrome-example.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-selenoid-twilio-chrome-example.yaml)
- [browserconfig-selenoid-twilio-firefox-example.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-selenoid-twilio-firefox-example.yaml)

```sh
helm upgrade selenosis . -n selenosis --set browserUI.vncPassword="selenoid"
```

### Selenium Standalone (selenium/standalone)

Official Selenium project standalone images with a built-in VNC server. Password is configured via `SE_VNC_PASSWORD`. Minimal two-container setup: browser + seleniferous sidecar.

- [browserconfig-selenium-standalone-chrome-example.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-selenium-standalone-chrome-example.yaml) 
- [browserconfig-selenium-standalone-firefox-example.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-selenium-standalone-firefox-example.yaml)

```sh
helm upgrade selenosis . -n selenosis --set browserUI.vncPassword="${se_vnc_password}"
```

### Moon (quay.io/browser)

Images from Moon project, distributed via `quay.io/browser`. VNC requires a full X11 sidecar stack: `xvfb` (X server), `openbox` (window manager), and `x11vnc` (VNC server) — all from `quay.io/aerokube`. Also requires a `usergroup` ConfigMap (included in the manifests) to map the `user:4096` identity used by the browser containers.

- [browserconfig-moon-chrome-example.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-moon-chrome-example.yaml)
- [browserconfig-moon-firefox-example.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-moon-firefox-example.yaml)

```sh
helm upgrade selenosis . -n selenosis --set browserUI.vncPassword="selenoid"
```

Playwright-specific Moon images for CDP/BiDi protocol sessions.

- [browserconfig-moon-playwright-chrome-example.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-moon-playwright-chrome-example.yaml)
- [browserconfig-moon-playwright-firefox-example.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browserconfig-moon-playwright-firefox-example.yaml)


## Service types

Each service supports `ClusterIP`, `NodePort`, or `LoadBalancer`.

Example values:

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

Apply:

```sh
helm upgrade --install selenosis . -n selenosis -f values.local.yaml
```

## Ingress Configuration

Expose services via Ingress for external access with TLS and WebSocket support.

Each component (selenosis and browserUI) has its own ingress configuration under its respective section.

### Basic Setup (NGINX Ingress Controller)

```yaml
selenosis:
  ingress:
    enabled: true
    className: nginx
    host: selenosis.example.com

browserUI:
  ingress:
    enabled: true
    className: nginx
    host: ui.example.com
    # CRITICAL: WebSocket support for VNC proxy
    annotations:
      nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
      nginx.ingress.kubernetes.io/websocket-services: "browser-ui"
```

### With TLS/SSL Certificate

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

**Important**: Browser UI requires WebSocket support for the VNC proxy. The above annotations are required for NGINX Ingress Controller. Other ingress controllers may require different annotations.

Apply:

```sh
helm upgrade --install selenosis . -n selenosis -f ingress-values.yaml
```

## Maintainer Release Process

To create a new chart release:

1. Update `Chart.yaml` version:
   ```yaml
   version: 2.0.3
   ```

2. Commit and push:
   ```sh
   git add Chart.yaml
   git commit -m "Bump chart version to 2.0.3"
   git push
   ```

3. Create and push tag:
   ```sh
   git tag v2.0.3
   git push origin v2.0.3
   ```

4. GitHub Actions will automatically:
   - Lint and package the chart
   - Create GitHub Release with chart tarball
   - Publish to Helm repository (GitHub Pages)

5. Users can now install the new version:
   ```sh
   helm repo update
   helm upgrade selenosis selenosis/selenosis-deploy --version 2.0.3
   ```
