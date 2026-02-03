# OpenClaw Helm Chart

A Helm chart for deploying [OpenClaw](https://openclaw.ai/) - an open-source AI personal assistant - to Kubernetes.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- An API key from a supported LLM provider (Anthropic, OpenAI, etc.)

## Installation

### Quick Start

```bash
helm install openclaw . --set credentials.anthropicApiKey=sk-ant-xxx
```

### Using an Existing Secret

Create a secret with your API keys:

```bash
kubectl create secret generic openclaw-credentials \
  --from-literal=ANTHROPIC_API_KEY=sk-ant-xxx \
  --from-literal=OPENAI_API_KEY=sk-xxx
```

Then install the chart:

```bash
helm install openclaw . --set credentials.existingSecret=openclaw-credentials
```

### With Custom Values

```bash
helm install openclaw . -f my-values.yaml
```

## Configuration

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Container image repository | `ghcr.io/openclaw/openclaw` |
| `image.tag` | Container image tag | `2026.1.30` |
| `openclaw.agents.defaults.model` | Primary model (provider/model format) | `anthropic/claude-sonnet-4-20250514` |
| `openclaw.agents.defaults.timeoutSeconds` | Agent timeout in seconds | `600` |
| `openclaw.agents.defaults.thinkingDefault` | Thinking mode (low/high/off) | `low` |
| `openclaw.timezone` | Timezone environment variable | `UTC` |
| `openclaw.bind` | Bind mode (lan/localhost) | `lan` |
| `openclaw.skills` | Skills to install from ClawHub | `[]` |
| `openclaw.configOverrides` | Raw JSON merged into openclaw.json | `{}` |
| `credentials.anthropicApiKey` | Anthropic API key | `""` |
| `credentials.openaiApiKey` | OpenAI API key | `""` |
| `credentials.existingSecret` | Use existing secret | `""` |
| `chromium.enabled` | Enable browser automation | `true` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | Storage size | `5Gi` |
| `ingress.enabled` | Enable ingress | `false` |

### Full Configuration Reference

See [values.yaml](values.yaml) for all available configuration options.

## Architecture

OpenClaw is deployed as a single-instance application (not horizontally scalable) with the following components:

- **Gateway**: Main WebSocket control plane on port 18789
- **Canvas**: HTTP server on port 18793
- **Chromium Sidecar** (optional): Headless browser for automation via CDP on port 9222

### Deployment Strategy

The chart uses `Recreate` strategy since OpenClaw is designed as a single-instance application and cannot be scaled horizontally.

## Storage

By default, the chart creates a PersistentVolumeClaim for storing OpenClaw configuration and state:

- Configuration: `~/.openclaw/openclaw.json`
- State: `~/.openclaw-a`

To disable persistence (data will be lost on pod restart):

```bash
helm install openclaw . --set persistence.enabled=false
```

## Ingress

To expose OpenClaw via Ingress:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  hosts:
    - host: openclaw.example.com
      paths:
        - path: /
          pathType: Prefix
          servicePort: gateway
  tls:
    - secretName: openclaw-tls
      hosts:
        - openclaw.example.com
```

## Security

The chart follows security best practices:

- All containers run as non-root (UID 1000)
- All capabilities are dropped
- Seccomp profiles are enabled
- Read-only root filesystem (where possible)

## Uninstallation

```bash
helm uninstall openclaw
```

Note: The PersistentVolumeClaim is not automatically deleted. To remove it:

```bash
kubectl delete pvc openclaw
```

## Troubleshooting

### Check pod status

```bash
kubectl get pods -l app.kubernetes.io/name=openclaw
```

### View logs

```bash
kubectl logs -l app.kubernetes.io/name=openclaw -c openclaw
```

### Access the gateway locally

```bash
kubectl port-forward svc/openclaw 18789:18789
```

## License

This Helm chart is provided under the MIT License.
