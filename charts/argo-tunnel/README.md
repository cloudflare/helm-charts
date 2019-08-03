# Argo Tunnel Ingress Controller

### TL;DR;
```console
$ helm install --name anydomain cloudflare/argo-tunnel
```
> **Tip**: See [Your First Tunnel][guide-first-tunnel].

### About
Argo Tunnel Ingress Controller provides Kubernetes Ingress via Argo Tunnels.
The controller establishes or destroys tunnels by monitoring changes to resources.

Argo Tunnel offers an easy way to expose web servers securely to the internet,
without opening up firewall ports and configuring ACLs. Argo Tunnel also ensures
requests route through Cloudflare before reaching the web server so you can be 
sure attack traffic is stopped with Cloudflare’s WAF and Unmetered DDoS mitigation
and authenticated with Access if you’ve enabled those features for your account.

- [Argo Smart Routing][argo-smart-routing]
- [Argo Tunnel: Reference][argo-tunnel-reference]
- [Argo Tunnel: Quick Start][argo-tunnel-quick-start]
- [Argo Tunnel Ingress: Quick Start][argo-tunnel-ingress-quick-start]

### Installing the Chart
To install the chart with the release name `argo-mydomain`:
```console
$ helm install --name anydomain cloudflare/argo-tunnel
```
> **Tip**: See [Your First Tunnel][guide-first-tunnel].

The command deploys the controller on the Kubernetes cluster in the default configuration.
The [configuration](#configuration) section lists the parameters that can be configured
during installation.

> **Tip**: List all releases using `helm list`

### Uninstalling the Chart
To uninstall/delete the `anydomain` deployment:
```console
$ helm delete anydomain
```

### Configuration
The following table lists the configurable parameters of the chart and their default values.

Parameter | Description | Default
--- | --- | ---
`controller.name` | name of the controller component | `controller`
`controller.defaultOriginSecret` | default origin tunnel certificate secret `<namespace>/<name>` | `""`
`controller.image.repository` | controller container image repository | `gcr.io/cloudflare-registry/argo-tunnel`
`controller.image.tag` | controller container image tag | `0.6.5`
`controller.image.pullPolicy` | controller container image pull policy | `Always`
`controller.ingressClass` | name of the ingress class to route through this controller | `argo-tunnel`
`controller.logLevel` | log-level for this controller | `3`
`controller.replicaCount` | desired number of controller pods ([load-balancers][argo-tunnel-load-balancing] are required for values larger than 1). | `1`
`controller.resyncPeriod` | period between kubernetes resource synchronization | `5m`
`controller.watchNamespace` | restrict resource watches to a single namespace | `""`
`controller.workers` | number of workers processing updates | `2`
`loadBalancing.enabled` | if `true`, replicaCount may be >1, requires [load balancing][argo-tunnel-load-balancing] enabled on account | `false`
`rbac.create` | if `true`, create & use RBAC resources | `true`
`rbac.name` | The name of the role binding to use. If not set and `create` is `true`, a name is generated using the fullname template. | ``
`secretMapping.create` | if `true`, create a secret mapping config map | `true`
`secretMapping.content` | The secret mapping config map YAML content. | `{}`
`serviceAccount.create` | if `true`, create a service account | `true`
`serviceAccount.name` | The name of the service account to use. If not set and `create` is `true`, a name is generated using the fullname template. | ``

A useful trick to debug issues with ingress is to increase the logLevel.
```console
$ helm install --name anydomain cloudflare/argo-tunnel --set controller.logLevel=6
```

> **Warn**: replicaCount >1 requires [load-balancers][argo-tunnel-load-balancing]

[argo-smart-routing]: https://www.cloudflare.com/products/argo-smart-routing/
[argo-tunnel-load-balancing]: https://developers.cloudflare.com/argo-tunnel/reference/load-balancing/
[argo-tunnel-reference]: https://developers.cloudflare.com/argo-tunnel/reference/
[argo-tunnel-quick-start]: https://developers.cloudflare.com/argo-tunnel/quickstart/
[argo-tunnel-ingress-quick-start]: https://github.com/cloudflare/cloudflare-ingress-controller/
[guide-first-tunnel]: https://github.com/cloudflare/cloudflare-ingress-controller/blob/master/docs/guide_first_tunnel.md
