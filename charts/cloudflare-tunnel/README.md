# cloudflare-tunnel

Helm chart for deploying locally-managed Cloudflare Tunnels on Kubernetes.

## Prerequisites

- Kubernetes cluster
- Helm 3.x
- `cloudflared` CLI for tunnel creation
- Cloudflare account with a domain

## Installation

### 1. Create Tunnel

```bash
# Authenticate with Cloudflare
cloudflared tunnel login

# Create a tunnel
cloudflared tunnel create my-k8s-tunnel

# Note the Tunnel ID from output
```

### 2. Create Kubernetes Secret

```bash
# Create namespace
kubectl create namespace cloudflare

# Create secret from credentials file
kubectl -n cloudflare create secret generic tunnel-credentials \
  --from-file=credentials.json=$HOME/.cloudflared/<TUNNEL_ID>.json
```

### 3. Install Chart

```bash
# Add repository
helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm repo update

# Install with values
helm install my-tunnel cloudflare/cloudflare-tunnel \
  -n cloudflare \
  --set cloudflare.tunnelName="my-k8s-tunnel" \
  --set cloudflare.secretName="tunnel-credentials" \
  --set cloudflare.ingress[0].hostname="app.example.com" \
  --set cloudflare.ingress[0].service="http://my-app.default.svc.cluster.local:80"
```

## Configuration

### Required Values

| Parameter | Description | Example |
|-----------|-------------|---------|
| `cloudflare.tunnelName` | Tunnel name | `my-k8s-tunnel` |
| `cloudflare.secretName` | Existing secret name | `tunnel-credentials` |
| `cloudflare.ingress` | Ingress rules array | See below |

### Optional Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cloudflare.account` | Account ID (if creating secret) | `""` |
| `cloudflare.tunnelId` | Tunnel UUID (if creating secret) | `""` |
| `cloudflare.secret` | Tunnel secret (if creating secret) | `""` |
| `cloudflare.enableWarp` | Enable WARP routing | `false` |
| `replicaCount` | Number of replicas | `2` |
| `image.repository` | Image repository | `cloudflare/cloudflared` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `resources` | Resource limits/requests | `{}` |

### Ingress Configuration

```yaml
cloudflare:
  ingress:
    # Route public traffic
    - hostname: app.example.com
      service: http://web-app.default.svc.cluster.local:80
    
    # SSH access
    - hostname: ssh.example.com
      service: ssh://jumpbox.default.svc.cluster.local:22
    
    # TCP service
    - hostname: db.example.com
      service: tcp://postgres.default.svc.cluster.local:5432
```

## Examples

### Example 1: Public Web Application

```yaml
# values.yaml
replicaCount: 2

cloudflare:
  tunnelName: "web-app-tunnel"
  secretName: "tunnel-credentials"
  ingress:
    - hostname: app.example.com
      service: http://nginx.default.svc.cluster.local:80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
```

### Example 2: Private Network Access (WARP)

```yaml
# values.yaml
cloudflare:
  tunnelName: "internal-access"
  secretName: "tunnel-credentials"
  enableWarp: true
  ingress:
    - service: http://internal-dashboard.default.svc.cluster.local:80
    - service: http://api.internal.svc.cluster.local:8080
```

### Example 3: Auto-Created Secret

```yaml
# values.yaml
cloudflare:
  account: "your-account-id"
  tunnelId: "your-tunnel-uuid"
  tunnelName: "my-tunnel"
  secret: "your-tunnel-secret"
  ingress:
    - hostname: app.example.com
      service: http://app.default:80
```

## Secret Format

When using `secretName`, the secret must contain a `credentials.json` key:

```json
{
  "AccountTag": "your-account-id",
  "TunnelID": "your-tunnel-uuid",
  "TunnelSecret": "base64-secret-string"
}
```

Create with:
```bash
kubectl create secret generic tunnel-credentials \
  --from-file=credentials.json=/path/to/credentials.json
```

## Verification

```bash
# Check pod status
kubectl -n cloudflare get pods

# View logs
kubectl -n cloudflare logs -l app.kubernetes.io/name=cloudflare-tunnel

# Check for successful connections (should see 4)
kubectl -n cloudflare logs -l app.kubernetes.io/name=cloudflare-tunnel | \
  grep "Registered tunnel connection"
```

## Upgrading

```bash
helm repo update
helm upgrade my-tunnel cloudflare/cloudflare-tunnel \
  -n cloudflare \
  -f values.yaml
```

## Uninstallation

```bash
# Uninstall chart
helm uninstall my-tunnel -n cloudflare

# Delete secret
kubectl -n cloudflare delete secret tunnel-credentials

# Delete tunnel
cloudflared tunnel delete my-k8s-tunnel
```

## Troubleshooting

### Pod in CrashLoopBackOff

Check logs for credential errors:
```bash
kubectl -n cloudflare logs <pod-name>
```

Common causes:
- Invalid credentials.json
- Secret not found
- Missing ingress rules

### No Tunnel Connections

Verify:
```bash
# Check secret exists
kubectl -n cloudflare get secret tunnel-credentials

# Verify credentials format
kubectl -n cloudflare get secret tunnel-credentials -o jsonpath='{.data.credentials\.json}' | base64 -d | jq .
```

### Service Not Accessible

- Verify service exists: `kubectl get svc <service-name>`
- Check ingress rules in values
- Verify DNS configuration

## Support

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Community Forum](https://community.cloudflare.com/)
