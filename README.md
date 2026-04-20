# OpenClaw Helm Chart

A Helm chart for deploying [OpenClaw](https://openclaw.ai) AI Gateway on OpenShift with optional [ClawSuite](https://github.com/outsourc-e/clawsuite) sidecar.

## Features

- **OpenClaw Gateway**: WebSocket control plane for AI agents (port 18789)
- **ClawSuite Sidecar**: Mission control dashboard for OpenClaw agents (optional, enabled by default)
- **OpenShift OAuth Proxy**: Protects web UIs with OpenShift authentication
- **Persistent Storage**: Configurable PVC for OpenClaw data
- **Network Isolation**: NetworkPolicy allowing egress to WAN while restricting cross-namespace traffic
- **Locked-down Security**: Minimal ServiceAccount with no RBAC bindings

## Prerequisites

- OpenShift 4.12+ cluster
- Helm 3.8+
- Access to OpenShift internal registry for oauth-proxy image

## Installation

### Install from OCI Registry (Recommended)

```bash
# Install specific version from GitHub Container Registry
helm install openclaw oci://ghcr.io/rhai-code/openclaw --version 1.1.0

# Or with custom values
helm install openclaw oci://ghcr.io/rhai-code/openclaw --version 1.1.0 -f custom-values.yaml
```

### Install from Local Chart

```bash
# Clone the repository
git clone https://github.com/rhai-code/openclaw-helm.git
cd openclaw-helm

# Install from current directory (chart is at repo root)
helm install openclaw .

# Or with custom values
helm install openclaw . -f custom-values.yaml
```

### Quick Start on OpenShift

```bash
# Create a new project
oc new-project openclaw

# Install from OCI registry
helm install openclaw oci://ghcr.io/rhai-code/openclaw --version 1.1.0 \
  --set route.host=openclaw.apps.your-cluster.com \
  --set openclaw.config.agents.defaults.model.primary="openai/gpt-4o"

# Wait for the deployment to be ready
oc wait --for=condition=available --timeout=300s deployment/openclaw

# Get the route URL
oc get route openclaw -o jsonpath='{.spec.host}'
```

## Configuration

### Required Configuration

Before using OpenClaw, you need to configure:

1. **AI Model Provider**: Set your preferred model and API keys
2. **Channels**: Configure messaging channels (WhatsApp, Telegram, etc.)
3. **Authentication**: Set gateway auth tokens

Example `custom-values.yaml`:

```yaml
openclaw:
  config:
    agents:
      defaults:
        model:
          primary: "openai/gpt-4o"
    gateway:
      port: 18789
      auth:
        mode: "token"
        token: "your-secure-token-here"
    channels:
      telegram:
        botToken: "${TELEGRAM_BOT_TOKEN}"

  # env:
  #   - name: OPENAI_API_KEY
  #     valueFrom:
  #       secretKeyRef:
  #         name: your-secret-name
  #         key: openai-api-key
```

### OAuth SAR Configuration

Control who can access the web UI via Subject Access Review:

```yaml
oauthProxy:
  # Default: requires access to read services in the deployment namespace
  sar: '{"namespace": "{{ .Release.Namespace }}", "resource": "services", "verb": "get"}'

  # Example: Allow any authenticated user
  # sar: '{"resource": "users", "verb": "get"}'

  # Example: Require specific group
  # sar: '{"resource": "groups", "name": "openclaw-users", "verb": "get"}'

  # Example: Cluster admin only
  # sar: '{"resource": "clusterrolebindings", "verb": "list"}'
```

### Storage Configuration

```yaml
persistence:
  enabled: true
  size: 20Gi
  storageClass: "gp3-csi" # Specify your StorageClass
  accessMode: ReadWriteOnce
```

### vLLM / OpenAI-Compatible API Configuration

To connect OpenClaw to a vLLM-hosted or other OpenAI-compatible API:

```yaml
# 1. Define secrets (creates Kubernetes Secrets)
secrets:
  vllm-api-key:
    api-key: "dummy-key" # or your actual key if required

# 2. Configure OpenClaw to use vLLM
openclaw:
  config:
    agents:
      defaults:
        model:
          primary: vllm/model-id-goes-here
    models:
      providers:
        vllm:
          baseUrl: "http://vllm.your-namespace.svc.cluster.local:8000/v1" # Point to your vLLM endpoint
          apiKey: "${VLLM_API_KEY}"
          api: openai-completions
          models:
            - id: model-id-goes-here
              name: Your open-weight model running in vLLM
              reasoning: true
              input: ["text"]
              cost:
                input: 0
                output: 0
                cacheRead: 0
                cacheWrite: 0
              contextWindow: 262144
              maxTokens: 65536
    gateway:
      auth:
        mode: "token"
        token: "your-secure-token-here" # Chart automatically shares this with ClawSuite

  env:
    # Reference the secret (secure - not in deployment manifest)
    - name: VLLM_API_KEY
      valueFrom:
        secretKeyRef:
          name: vllm-api-key
          key: api-key
```

**Note:** If using External Secrets Operator or other external secret management, skip the `secrets:` section and ensure the referenced secrets exist before deployment.

### ClawSuite Build Configuration

When ClawSuite is enabled without a pre-built image, a BuildConfig is created. This BuildConfig includes some patches to make ClawSuite play nice in our OAuth-proxied environment. You can change this behavior a bit:

```yaml
clawsuite:
  enabled: true
  # Uncomment to use pre-built image instead of building
  # image:
  #   repository: quay.io/your-org/clawsuite
  #   tag: v4.0.0

  build:
    git:
      repository: https://github.com/outsourc-e/clawsuite.git
      ref: main # or a specific tag/commit, defaults to tagged release
```

### Resource Limits

Default resources (generous for AI workloads):

```yaml
openclaw:
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "4"
      memory: "8Gi"
```

## Values Reference

### Global Settings

| Parameter          | Description                  | Default    |
| ------------------ | ---------------------------- | ---------- |
| `replicaCount`     | Number of replicas           | `1`        |
| `strategy.type`    | Deployment strategy          | `Recreate` |
| `imagePullSecrets` | Image pull secrets           | `[]`       |
| `secrets`          | Kubernetes Secrets to create | `{}`       |

### OpenClaw Configuration

| Parameter                   | Description           | Default                                 |
| --------------------------- | --------------------- | --------------------------------------- |
| `openclaw.image.repository` | OpenClaw image        | `ghcr.io/openclaw/openclaw`             |
| `openclaw.image.tag`        | Image tag             | `""` (defaults to AppVersion + `-slim`) |
| `openclaw.port`             | WebSocket port        | `18789`                                 |
| `openclaw.resources`        | Resource limits       | See values.yaml                         |
| `openclaw.config`           | openclaw.json content | Minimal default                         |
| `openclaw.env`              | Extra env vars        | `[]`                                    |

### ClawSuite Configuration

| Parameter                        | Description                | Default                                       |
| -------------------------------- | -------------------------- | --------------------------------------------- |
| `clawsuite.enabled`              | Enable ClawSuite           | `true`                                        |
| `clawsuite.image.repository`     | Pre-built image (optional) | `""`                                          |
| `clawsuite.build.git.repository` | Source git URL             | `https://github.com/outsourc-e/clawsuite.git` |
| `clawsuite.build.git.ref`        | Git ref to build           | `v4.0.0`                                      |
| `clawsuite.port`                 | ClawSuite port             | `3000`                                        |
| `clawsuite.allowedHosts`         | Allowed hosts              | `""` (allows all)                             |
| `clawsuite.service.enabled`      | Enable separate service    | `true`                                        |
| `clawsuite.route.enabled`        | Enable separate route      | `true`                                        |
| `clawsuite.oauthProxy.sar`       | Subject Access Review      | `'{"namespace": "{{ .Release.Namespace }}", "resource": "services", "verb": "get"}'` |
| `clawsuite.resources`            | Resource limits            | See values.yaml                               |
| `clawsuite.env`                  | Extra env vars             | `[]`                                          |

### OAuth Proxy Configuration

| Parameter                     | Description                             | Default                                                                              |
| ----------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------ |
| `oauthProxy.image.repository` | OAuth proxy image                       | `image-registry.openshift-image-registry.svc:5000/openshift/oauth-proxy`             |
| `oauthProxy.image.tag`        | Image tag                               | `v4.4`                                                                               |
| `oauthProxy.sar`              | Subject Access Review                   | `'{"namespace": "{{ .Release.Namespace }}", "resource": "services", "verb": "get"}'` |
| `oauthProxy.cookieSecret`     | Cookie secret (auto-generated if empty) | `""`                                                                                 |

### Persistence

| Parameter                  | Description       | Default                |
| -------------------------- | ----------------- | ---------------------- |
| `persistence.enabled`      | Enable PVC        | `true`                 |
| `persistence.size`         | PVC size          | `10Gi`                 |
| `persistence.storageClass` | StorageClass name | `""` (default)         |
| `persistence.mountPath`    | Mount path        | `/home/node/.openclaw` |

### Network Policy

| Parameter                         | Description          | Default |
| --------------------------------- | -------------------- | ------- |
| `networkPolicy.enabled`           | Enable NetworkPolicy | `true`  |
| `networkPolicy.allowWanEgress`    | Allow egress to WAN  | `true`  |
| `networkPolicy.additionalIngress` | Custom ingress rules | `[]`    |
| `networkPolicy.additionalEgress`  | Custom egress rules  | `[]`    |

### Route

| Parameter                                 | Description          | Default        |
| ----------------------------------------- | -------------------- | -------------- |
| `route.enabled`                           | Enable Route         | `true`         |
| `route.host`                              | Route hostname       | Auto-generated |
| `route.tls.termination`                   | TLS termination type | `edge`         |
| `route.tls.insecureEdgeTerminationPolicy` | Insecure policy      | `Redirect`     |

## Security Considerations

### ServiceAccount

The chart creates a ServiceAccount configured for OpenShift OAuth integration:

```yaml
serviceAccount:
  create: true
  automountServiceAccountToken: true # Required for OpenShift OAuth proxy
```

The ServiceAccount is automatically annotated with:

- `serviceaccounts.openshift.io/oauth-redirectreference.primary` - References the Route for OAuth redirect URI validation

**RBAC Requirements:** The oauth-proxy requires read access to the `oauth-serving-cert` ConfigMap in the `openshift-config-managed` namespace, plus cluster-scoped permissions to create SubjectAccessReviews and TokenReviews. These permissions are automatically granted by the chart.

### Network Policy

The NetworkPolicy:

- **Denies** all ingress from outside the namespace except OpenShift router
- **Allows** egress to WAN (0.0.0.0/0 excluding RFC1918 ranges)
- **Allows** DNS resolution
- **Allows** intra-namespace communication

### Container Security

All containers run with:

- Non-root user (UID 1000)
- Read-only root filesystem (OAuth proxy)
- Dropped capabilities
- No privilege escalation

## Troubleshooting

### Check Pod Status

```bash
oc get pods -n openclaw
oc describe pod -n openclaw -l app.kubernetes.io/name=openclaw
```

### View Logs

```bash
# OpenClaw logs
oc logs -n openclaw deployment/openclaw -c openclaw

# ClawSuite logs
oc logs -n openclaw deployment/openclaw -c clawsuite

# OAuth proxy logs
oc logs -n openclaw deployment/openclaw -c oauth-proxy
```

### Check Build Status (ClawSuite)

```bash
oc get builds -n openclaw
oc logs -n openclaw build/{{ include "openclaw.fullname" . }}-clawsuite-1
```

### Common Issues

**Issue**: OAuth proxy fails with "Unauthorized"

- **Solution**: Check SAR configuration matches your OpenShift RBAC

**Issue**: ClawSuite can't connect to OpenClaw

- **Solution**: Verify pods are in same namespace (NetworkPolicy allows intra-namespace)

**Issue**: BuildConfig fails

- **Solution**: Ensure cluster can reach GitHub or configure a mirror

## Upgrading

```bash
# Upgrade from OCI registry
helm upgrade openclaw oci://ghcr.io/rhai-code/openclaw --version 1.1.0

# Upgrade from local chart
helm upgrade openclaw .

# Upgrade with new values
helm upgrade openclaw oci://ghcr.io/rhai-code/openclaw --version 1.1.0 -f new-values.yaml
```

## Uninstallation

```bash
# Uninstall release
helm uninstall openclaw

# Clean up PVC (data will be lost!)
oc delete pvc -n openclaw -l app.kubernetes.io/name=openclaw
```

## Contributing

Contributions are welcome! Please submit issues and pull requests to the chart repository.

## License

This chart is released under the MIT License. OpenClaw and ClawSuite have their own licenses.

## References

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [ClawSuite GitHub](https://github.com/outsourc-e/clawsuite)
- [OpenShift OAuth Proxy Documentation](https://docs.openshift.com/container-platform/latest/authentication/oauth-proxy.html)
