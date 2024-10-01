# Cloudflare Helm Charts

### About
A convenient location to publish Cloudflare helm charts

### Setup
```bash
helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm repo update
```

### Discovery
```bash
helm search repo cloudflare
```

### Contents

- `charts/cloudflare-tunnel`: Helm 3 chart for creating local cloudflared tunnels
- `charts/cloudflare-tunnel-remote`: Helm 3 chart for deploying remotely managed cloudflared tunnels that have already been provisioned in Cloudflare
