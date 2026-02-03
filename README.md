# ⚓ OpenClaw Helm Chart

[![Helm 3](https://img.shields.io/badge/Helm-3.0+-0f1689?logo=helm&logoColor=white)](https://helm.sh/)
[![Kubernetes 1.19+](https://img.shields.io/badge/Kubernetes-1.19+-326ce5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Helm chart for deploying [OpenClaw](https://openclaw.ai/) — an open-source AI personal assistant — to Kubernetes.

📖 [Documentation](https://openclaw.ai/docs) • 🐛 [Issues](https://github.com/openclaw/openclaw-helm/issues) • 💬 [Discussions](https://github.com/openclaw/openclaw-helm/discussions)

---

## 🚀 Quick Start

```bash
helm install openclaw . --set credentials.anthropicApiKey=sk-ant-xxx
```

---

## 📋 Table of Contents

- [Installation](#-installation)
- [Configuration](#️-configuration)
- [Architecture](#️-architecture)
- [Storage](#-storage)
- [Configuration Mode](#-configuration-mode)
- [Ingress](#-ingress)
- [Security](#-security)
- [Uninstallation](#️-uninstallation)
- [Troubleshooting](#-troubleshooting)
- [License](#-license)

---

## 📦 Installation

### Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- An API key from a supported LLM provider (Anthropic, OpenAI, etc.)

### Quick Start

```bash
helm install openclaw . --set credentials.anthropicApiKey=sk-ant-xxx
```

<details>
<summary><b>Using an Existing Secret</b></summary>

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

</details>

<details>
<summary><b>With Custom Values</b></summary>

```bash
helm install openclaw . -f my-values.yaml
```

</details>

---

## ⚙️ Configuration

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
| `credentials.anthropicApiKey` | Anthropic API key | `""` |
| `credentials.existingSecret` | Use existing secret | `""` |
| `chromium.enabled` | Enable browser automation | `true` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `ingress.enabled` | Enable ingress | `false` |

<details>
<summary><b>Full Configuration Reference</b></summary>

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Container image repository | `ghcr.io/openclaw/openclaw` |
| `image.tag` | Container image tag | `2026.1.30` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `openclaw.agents.defaults.model` | Primary model (provider/model format) | `anthropic/claude-sonnet-4-20250514` |
| `openclaw.agents.defaults.timeoutSeconds` | Agent timeout in seconds | `600` |
| `openclaw.agents.defaults.thinkingDefault` | Thinking mode (low/high/off) | `low` |
| `openclaw.timezone` | Timezone environment variable | `UTC` |
| `openclaw.bind` | Bind mode (lan/localhost) | `lan` |
| `openclaw.skills` | Skills to install from ClawHub | `[]` |
| `openclaw.configOverrides` | Raw JSON merged into openclaw.json | `{}` |
| `openclaw.configMode` | Config management mode (merge/overwrite) | `merge` |
| `credentials.anthropicApiKey` | Anthropic API key | `""` |
| `credentials.openaiApiKey` | OpenAI API key | `""` |
| `credentials.existingSecret` | Use existing secret | `""` |
| `chromium.enabled` | Enable browser automation | `true` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | Storage size | `5Gi` |
| `ingress.enabled` | Enable ingress | `false` |

See [values.yaml](values.yaml) for all available configuration options.

</details>

---

## 🏗️ Architecture

OpenClaw is deployed as a single-instance application with the following components:

- **Gateway** — Main WebSocket control plane on port `18789`
- **Canvas** — HTTP server on port `18793`
- **Chromium Sidecar** *(optional)* — Headless browser for automation via CDP on port `9222`

> **Note:** The chart uses `Recreate` deployment strategy since OpenClaw is designed as a single-instance application and cannot be scaled horizontally.

---

## 💾 Storage

By default, the chart creates a PersistentVolumeClaim for storing OpenClaw configuration and state:

- Configuration: `~/.openclaw/openclaw.json`
- State: `~/.openclaw-a`

<details>
<summary><b>Disable Persistence</b></summary>

To disable persistence (data will be lost on pod restart):

```bash
helm install openclaw . --set persistence.enabled=false
```

</details>

---

## 🔀 Configuration Mode

OpenClaw is inherently stateful and updates its own configuration file at runtime (e.g., when installing skills or changing settings via the UI). By default, the chart uses `merge` mode to preserve these runtime changes.

| Mode | Behavior |
|------|----------|
| `merge` *(default)* | Merges Helm values with existing config. Runtime changes are preserved, Helm values take precedence on conflicts. |
| `overwrite` | Completely replaces config on every pod restart. Runtime changes are lost. |

```yaml
openclaw:
  configMode: "merge"  # or "overwrite"
```

> **Tip:** Use `overwrite` mode if you want strict GitOps where Helm is the single source of truth.

---

## 🌐 Ingress

<details>
<summary><b>Full Ingress Example</b></summary>

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

</details>

---

## 🔒 Security

The chart follows security best practices:

- ✅ All containers run as non-root (UID 1000)
- ✅ All capabilities are dropped
- ✅ Seccomp profiles are enabled
- ✅ Read-only root filesystem (where possible)

---

## 🗑️ Uninstallation

```bash
helm uninstall openclaw
```

> **Note:** The PersistentVolumeClaim is not automatically deleted. To remove it:

```bash
kubectl delete pvc openclaw
```

---

## 🔧 Troubleshooting

<details>
<summary><b>Debug Commands</b></summary>

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

</details>

---

## 📄 License

This Helm chart is provided under the [MIT License](LICENSE).
