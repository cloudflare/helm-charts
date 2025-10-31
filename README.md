# Cloudflare Helm Charts

Official Helm charts for deploying Cloudflare Tunnel (`cloudflared`) on Kubernetes.

## About

Cloudflare Tunnel creates secure, outbound-only connections from your infrastructure to Cloudflare's global network without requiring publicly routable IP addresses or opening inbound firewall ports.

**Key Benefits:**
- No inbound firewall rules needed
- Automatic HTTPS with Cloudflare TLS
- Built-in DDoS protection
- Support for HTTP, SSH, RDP, and TCP protocols
- Zero Trust Network Access (ZTNA) via WARP-to-Tunnel

## Quick Start

```bash
# Add the Cloudflare Helm repository
helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm repo update

# Search available charts
helm search repo cloudflare
```

## Available Charts

### `cloudflare-tunnel` - Locally-Managed

Deploy tunnels created and managed via the `cloudflared` CLI.

**Best for:** GitOps workflows, Infrastructure as Code, automated provisioning

**Key Features:**
- Routes configured in Helm values (`ingress`)
- Supports `secretName` for referencing existing credentials
- Full control over tunnel configuration
- Version-controlled infrastructure

**Requirements:**
- `cloudflared` CLI for tunnel creation
- Tunnel credentials (credentials.json)

### `cloudflare-tunnel-remote` - Remote-Managed

Deploy tunnels created in the Cloudflare Zero Trust Dashboard.

**Best for:** Quick deployments, GUI-first workflows, rapid prototyping

**Key Features:**
- Routes configured in Dashboard (not Helm)
- Simple token-based authentication
- No CLI tools required
- Dashboard management

**Requirements:**
- Tunnel created in Dashboard
- Tunnel token (single string)

## Comparison

| Aspect | `cloudflare-tunnel` | `cloudflare-tunnel-remote` |
|--------|---------------------|----------------------------|
| **Tunnel Creation** | CLI: `cloudflared tunnel create` | Dashboard GUI |
| **Credentials** | credentials.json (3 fields) | Token (single string) |
| **Route Config** | Helm values.yaml | Dashboard only |
| **Existing Secret** | ✅ Yes (`secretName`) | ❌ No |
| **CLI Required** | Setup only | Never |
| **Best For** | GitOps, IaC | Quick setup, GUI users |

## Installation Examples

### Locally-Managed Tunnel

```bash
# 1. Create tunnel with cloudflared CLI
cloudflared tunnel login
cloudflared tunnel create my-k8s-tunnel

# 2. Create Kubernetes secret
kubectl create namespace cloudflare
kubectl -n cloudflare create secret generic tunnel-creds \
  --from-file=credentials.json=$HOME/.cloudflared/<TUNNEL_ID>.json

# 3. Create values.yaml
cat > values.yaml <<EOF
cloudflare:
  account: "your-account-id"
  tunnelName: "my-k8s-tunnel"
  tunnelId: "your-tunnel-uuid"
  secretName: "tunnel-creds"
  ingress:
    - hostname: app.example.com
      service: http://my-service.default.svc.cluster.local:80
EOF

# 4. Install chart
helm install my-tunnel cloudflare/cloudflare-tunnel \
  -n cloudflare \
  -f values.yaml
```

### Remote-Managed Tunnel

```bash
# 1. Create tunnel in Dashboard
# - Go to https://one.dash.cloudflare.com
# - Navigate: Networks → Tunnels → Create tunnel
# - Copy the token from installation command

# 2. Install chart with token
kubectl create namespace cloudflare
helm install my-tunnel cloudflare/cloudflare-tunnel-remote \
  -n cloudflare \
  --set cloudflare.tunnel_token="<YOUR_TOKEN>"

# 3. Configure routes in Dashboard
# - Go to your tunnel → Public Hostname tab
# - Add hostname routes
```

## Common Use Cases

### Public Web Application

Expose internal services to the internet with HTTPS:

```yaml
# Locally-managed tunnel values
cloudflare:
  ingress:
    - hostname: app.example.com
      service: http://web-app.default.svc.cluster.local:80
```

### Private Network Access (WARP-to-Tunnel)

Enable Zero Trust access for remote users:

```yaml
cloudflare:
  enableWarp: true
  ingress:
    - service: http://internal-dashboard.default.svc.cluster.local:80
```

Users connect via Cloudflare WARP client for secure access to internal resources.

### Multi-Protocol Support

Route various protocols through the tunnel:

```yaml
cloudflare:
  ingress:
    # Web service
    - hostname: web.example.com
      service: http://web-server.default:80
    
    # SSH access
    - hostname: ssh.example.com
      service: ssh://jumpbox.default:22
    
    # Database (TCP)
    - hostname: db.example.com
      service: tcp://postgres.default:5432
```

## Configuration

### Locally-Managed Values

```yaml
replicaCount: 2

cloudflare:
  # Required for auto-created secret
  account: "your-account-id"
  tunnelId: "your-tunnel-uuid"
  tunnelName: "your-tunnel-name"
  secret: "your-tunnel-secret"
  
  # OR reference existing secret (recommended)
  secretName: "tunnel-credentials"
  
  # Enable WARP routing for private access
  enableWarp: false
  
  # Define ingress rules
  ingress:
    - hostname: app.example.com
      service: http://my-app.default.svc.cluster.local:80

image:
  repository: cloudflare/cloudflared
  pullPolicy: IfNotPresent

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

### Remote-Managed Values

```yaml
replicaCount: 2

cloudflare:
  tunnel_token: "your-token-from-dashboard"

image:
  repository: cloudflare/cloudflared
  pullPolicy: IfNotPresent

resources:
  limits:
    cpu: 100m
    memory: 128Mi
```

## Production Best Practices

### High Availability

Run multiple replicas for redundancy:

```yaml
replicaCount: 3

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: cloudflare-tunnel
```

### Secrets Management

**For locally-managed tunnels:**

```yaml
cloudflare:
  secretName: "tunnel-credentials"  # Reference existing secret
```

**For remote-managed tunnels:**

Use CI/CD secret injection:
```bash
helm install my-tunnel cloudflare/cloudflare-tunnel-remote \
  --set cloudflare.tunnel_token="$TUNNEL_TOKEN"
```

Or external secrets tools:
- External Secrets Operator
- Sealed Secrets
- Helm Secrets Plugin

### Resource Limits

```yaml
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### Monitoring

Tunnel pods expose metrics on port 2000:

```bash
kubectl port-forward -n cloudflare <POD_NAME> 2000:2000
curl http://localhost:2000/metrics
```

## Kubernetes Service Discovery

Route to any Kubernetes service using internal DNS:

```
http://<service>.<namespace>.svc.cluster.local:<port>
```

**Examples:**
- `http://nginx.default.svc.cluster.local:80`
- `http://api.production.svc.cluster.local:8080`
- `tcp://postgres.database.svc.cluster.local:5432`

## Verification

Check tunnel connection status:

```bash
# View pod status
kubectl -n cloudflare get pods

# Check logs for connections
kubectl -n cloudflare logs -l app.kubernetes.io/name=cloudflare-tunnel

# Look for successful registration
kubectl -n cloudflare logs -l app.kubernetes.io/name=cloudflare-tunnel | \
  grep "Registered tunnel connection"
```

Healthy tunnels show 4 registered connections to Cloudflare edge locations.

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| `CrashLoopBackOff` | Invalid credentials | Verify secret or token |
| `Error 1033` | No routes configured | Add routes (Dashboard or values) |
| `502 Bad Gateway` | Service unreachable | Check service is running |
| cert.pem warning | Normal for remote tunnels | Ignore (token auth used) |

### Health Check Endpoint

```bash
kubectl -n cloudflare exec <POD_NAME> -- curl http://localhost:2000/ready
```

- HTTP 200 = Healthy
- HTTP 503 = Unhealthy

## Upgrading

```bash
# Update repository
helm repo update cloudflare

# Upgrade release
helm upgrade my-tunnel cloudflare/cloudflare-tunnel \
  -n cloudflare \
  -f values.yaml
```

## Cleanup

```bash
# Uninstall chart
helm uninstall my-tunnel -n cloudflare

# Delete namespace
kubectl delete namespace cloudflare

# Delete tunnel (locally-managed)
cloudflared tunnel delete <TUNNEL_NAME>

# Delete tunnel (remote-managed)
# Dashboard: Networks → Tunnels → Delete
```

## Support & Resources

- **Cloudflare Tunnel Docs**: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- **Zero Trust Platform**: https://developers.cloudflare.com/cloudflare-one/
- **Community Forum**: https://community.cloudflare.com/
- **Dashboard**: https://dash.cloudflare.com/
- **Zero Trust Dashboard**: https://one.dash.cloudflare.com/

## License

Apache 2.0 - See [LICENSE](LICENSE) for details.
