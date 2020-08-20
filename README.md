# Cloudflare Helm Charts

## Note
This project focuses on using a component of Cloudflare Argo Tunnel which is [no longer supported](https://developers.cloudflare.com/argo-tunnel/reference/kubernetes/). Cloudflare is not actively maintaining this project right now.

### About
A convenient location to publish Cloudflare helm charts

### Setup
```bash
helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm repo update
```

### Discovery
```bash
helm search cloudflare
```
