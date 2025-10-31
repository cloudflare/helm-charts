# cloudflare-tunnel-remote

Helm chart for deploying remote-managed Cloudflare Tunnels on Kubernetes.

## Prerequisites

- Kubernetes cluster
- Helm 3.x
- Cloudflare account with a domain
- Access to Cloudflare Zero Trust Dashboard

## Installation

### 1. Create Tunnel in Dashboard

1. Navigate to https://one.dash.cloudflare.com/
2. Go to **Networks** → **Tunnels**
3. Click **Create a tunnel**
4. Select **Cloudflared** connector
5. Enter tunnel name
6. Click **Save tunnel**
7. Copy the token from the installation command

### 2. Install Chart

```bash
# Add repository
helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm repo update

# Install with token
helm install my-tunnel cloudflare/cloudflare-tunnel-remote \
  -n cloudflare \
  --create-namespace \
  --set cloudflare.tunnel_token="<YOUR_TOKEN>"
```

### 3. Configure Routes in Dashboard

1. In Dashboard, go to your tunnel
2. Click **Public Hostname** tab
3. Click **Add a public hostname**
4. Configure:
   - Subdomain: `app`
   - Domain: Select your domain
   - Service: `http://my-service.default.svc.cluster.local:80`
5. Click **Save hostname**

## Configuration

### Required Values

| Parameter | Description | Example |
|-----------|-------------|---------|
| `cloudflare.tunnel_token` | Token from Dashboard | `eyJhIjoiYTMx...` |

### Optional Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `2` |
| `image.repository` | Image repository | `cloudflare/cloudflared` |
| `image.tag` | Image tag override | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `resources` | Resource limits/requests | `{}` |

### Important Note

**Routes cannot be configured in Helm values** for remote-managed tunnels. All route configuration must be done in the Cloudflare Dashboard.

## Examples

### Example 1: Basic Deployment

```yaml
# values.yaml
replicaCount: 2

cloudflare:
  tunnel_token: "eyJhIjoiYTMxMjQw..."

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

```bash
helm install my-tunnel cloudflare/cloudflare-tunnel-remote \
  -n cloudflare \
  -f values.yaml
```

### Example 2: High Availability

```yaml
# values.yaml
replicaCount: 3

cloudflare:
  tunnel_token: "eyJhIjoiYTMxMjQw..."

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app.kubernetes.io/name: cloudflare-tunnel-remote
        topologyKey: kubernetes.io/hostname

resources:
  limits:
    cpu: 200m
    memory: 256Mi
```

### Example 3: CI/CD Integration

```bash
# Store token in CI/CD secrets
helm install my-tunnel cloudflare/cloudflare-tunnel-remote \
  -n cloudflare \
  --set cloudflare.tunnel_token="$TUNNEL_TOKEN" \
  --set replicaCount=2
```

## Production Recommendations

### Secrets Management

**DO NOT** commit tokens to Git. Use one of these approaches:

1. **CI/CD Injection**
   ```bash
   --set cloudflare.tunnel_token="$TUNNEL_TOKEN"
   ```

2. **External Secrets Operator**
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: tunnel-token
   spec:
     target:
       template:
         data:
           tunnel_token: "{{ .token }}"
   ```

3. **Sealed Secrets**
   ```bash
   echo -n "token" | kubectl create secret generic tunnel-token \
     --dry-run=client --from-file=tunnel_token=/dev/stdin -o yaml | \
     kubeseal -o yaml
   ```

4. **Helm Secrets Plugin**
   ```bash
   helm secrets install my-tunnel cloudflare/cloudflare-tunnel-remote \
     -f secrets.yaml
   ```

## Verification

```bash
# Check pod status
kubectl -n cloudflare get pods

# View logs
kubectl -n cloudflare logs -l app.kubernetes.io/name=cloudflare-tunnel-remote

# Check for successful connections (should see 4)
kubectl -n cloudflare logs -l app.kubernetes.io/name=cloudflare-tunnel-remote | \
  grep "Registered tunnel connection"
```

Expected output:
```
INF Registered tunnel connection connIndex=0 location=...
INF Registered tunnel connection connIndex=1 location=...
INF Registered tunnel connection connIndex=2 location=...
INF Registered tunnel connection connIndex=3 location=...
```

## Configure Routes (Dashboard)

After deployment, configure routes in the Dashboard:

1. Go to https://one.dash.cloudflare.com/
2. **Networks** → **Tunnels** → Click your tunnel
3. **Public Hostname** tab → **Add a public hostname**
4. Configure route:
   - **Subdomain**: Your subdomain
   - **Domain**: Your domain
   - **Service Type**: HTTP/HTTPS/TCP/SSH
   - **URL**: Kubernetes service URL

### Service URL Format

```
http://<service-name>.<namespace>.svc.cluster.local:<port>
```

**Examples:**
- `http://nginx.default.svc.cluster.local:80`
- `http://api.production.svc.cluster.local:8080`
- `tcp://postgres.database.svc.cluster.local:5432`

## Upgrading

```bash
helm repo update
helm upgrade my-tunnel cloudflare/cloudflare-tunnel-remote \
  -n cloudflare \
  --set cloudflare.tunnel_token="$TUNNEL_TOKEN"
```

## Uninstallation

```bash
# Uninstall chart
helm uninstall my-tunnel -n cloudflare

# Delete tunnel in Dashboard
# Go to Networks → Tunnels → Click tunnel → Delete
```

## Troubleshooting

### Error 1033

**Cause:** No routes configured in Dashboard  
**Solution:** Add public hostname routes in Dashboard

### Pod in CrashLoopBackOff

**Cause:** Invalid or expired token  
**Solution:**
1. Generate new token in Dashboard
2. Update Helm release with new token

```bash
helm upgrade my-tunnel cloudflare/cloudflare-tunnel-remote \
  --set cloudflare.tunnel_token="$NEW_TOKEN" \
  --reuse-values
```

### cert.pem Warning in Logs

```
ERR Cannot determine default origin certificate path
```

**This is normal and can be ignored.** Remote-managed tunnels use token authentication, not certificate files.

### Service Not Accessible (502)

**Verify service exists:**
```bash
kubectl get svc <service-name> -n <namespace>
```

**Test internal connectivity:**
```bash
kubectl run test -it --rm --image=curlimages/curl -- \
  curl http://<service>.<namespace>.svc.cluster.local:<port>
```

## Comparison with cloudflare-tunnel

| Feature | cloudflare-tunnel-remote | cloudflare-tunnel |
|---------|-------------------------|-------------------|
| **Tunnel Creation** | Dashboard | CLI |
| **Route Config** | Dashboard only | Helm values |
| **Token Type** | Single string | credentials.json |
| **CLI Required** | No | Yes (for setup) |
| **Best For** | Quick setup, GUI users | GitOps, IaC |

## Support

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Zero Trust Dashboard](https://one.dash.cloudflare.com/)
- [Community Forum](https://community.cloudflare.com/)
