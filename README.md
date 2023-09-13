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

### Usage

#### Cloudflare Tunnel

If you are using the Cloudflare Tunnel without an existing secret you have to set the following values:

```yaml
cloudflare:
  account: <your-cloudflare-account-id>
  tunnelName: <your-cloudflare-tunnel-name>
  tunnelId: <your-cloudflare-tunnel-id>
  secret: <your-cloudflare-tunnel-token>
```

If you are using the Cloudflare Tunnel with an existing secret you have to set the following values:

```yaml
cloudflare:
  tunnelName: <your-cloudflare-tunnel-name>
  secretName: <your-existing-cloudflare-tunnel-secret> # It must be created in the namespace that your are deploying your cloudflare-tunnel in.
```

#### Cloudflare Tunnel Remote

Docs are coming soon...

### Contents

- `charts/cloudflare-tunnel`: Helm 3 chart using cloudflared best practices
