# selenosis-deploy Helm chart

## Architecture

![diagram](https://github.com/user-attachments/assets/c483699a-cf94-47a4-a2cf-448f21e39bf3)


## Summary

This chart deploys the full Selenosis stack:
- [selenosis](selenosis) — Selenium hub/proxy for creating sessions and routing traffic.
- [browser-controller](https://github.com/alcounit/browser-controller) — Kubernetes controller that reconciles `Browser` and `BrowserConfig` CRDs into Pods.
- [browser-service](https://github.com/alcounit/browser-service) — REST + event stream API for managing `Browser` resources.
- [browser-ui](https://github.com/alcounit/browser-ui) — UI + VNC WebSocket proxy backed by browser-service.

CRDs for `Browser` and `BrowserConfig` are shipped in `crds/` and installed automatically.
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

## BrowserConfig examples

Examples are in `examples/`:
- [browser-config-multisidecar.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browser-config-multisidecar.yaml) multi sidecar example.
- [browser-config-singlesidecar.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/browser-config-singlesidecar.yaml) is a standalone example.
- [playwright-config-multisidecar.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/playwright-config-multisidecar.yaml) multi sidecar example.


```sh
helm upgrade selenosis . -n selenosis --set browserUI.vncPassword="selenoid"
kubectl apply -n selenosis -f ./examples/browser-config-multisidecar.yaml
```

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
