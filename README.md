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

## Install and first deploy

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
- [playwright-config-multisidecar.yaml](https://github.com/alcounit/selenosis-deploy/blob/main/examples/playwright-config-multisidecar.yaml) is a standalone example.


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
