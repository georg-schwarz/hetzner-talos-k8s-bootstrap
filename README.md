# Talos Linux Kubernetes on Hetzner Cloud

Deploy a multi-node Talos Linux Kubernetes cluster on Hetzner Cloud using Pulumi.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![CI Status](https://github.com/georg-schwarz/pulumi-hetzner-k8s-demo/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/python-3.11+-blue.svg)
![Code Style](https://img.shields.io/badge/code%20style-ruff-blue.svg)
![Type Checked](https://img.shields.io/badge/type%20checked-ty-blue.svg)

## Why This Stack?

- **Talos Linux:** Immutable, API-driven OS designed specifically for Kubernetes - no SSH, no shell, just pure Kubernetes
- **Hetzner Cloud:** Cost-effective European hosting with excellent price/performance ratio
- **Pulumi (Python):** Infrastructure-as-code with real programming languages instead of YAML templates
- **Multi-node Architecture:** Demonstrates real-world cluster patterns with single-node control plane and multiple worker nodes

> **⚠️  EDUCATIONAL PURPOSE ONLY**
>
> This repository is designed for learning and demonstration purposes. It is **not production-ready** and requires additional hardening and configuration for production use. See [Production Considerations](#production-considerations) for what would need to be added.

## Prerequisites

1. Install [talosctl](https://docs.siderolabs.com/talos/latest/getting-started/talosctl)
2. Install [kubectl](https://kubernetes.io/docs/tasks/tools/)
3. Install [Pulumi](https://www.pulumi.com/docs/get-started/download-install/)
4. Install [uv](https://docs.astral.sh/uv/getting-started/installation/) (Python package and project manager)
5. Create a Pulumi Cloud account (free)
6. Create a Hetzner Cloud account (state Feb 2026: there are referral codes to get 20€ free credit)

## Setup

1. **Login to Pulumi**
   ```bash
   pulumi login
   ```

2. **Initialize or select a stack**
   ```bash
   pulumi stack init dev
   # or select existing: pulumi stack select dev
   ```

3. **Install Python dependencies**
   ```bash
   uv sync
   ```

4. **Get your Hetzner Cloud API token** from [console.hetzner.cloud](https://console.hetzner.cloud/) → Security → API Tokens (Read & Write permissions)

5. **Configure the Hetzner token**
   ```bash
   pulumi config set hcloud:token YOUR_HETZNER_TOKEN --secret
   ```

6. **Create an SSH key** (used for potential rescue mode operations)
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_hetzner_pulumi -C "YOUR_EMAIL"
   ```

7. **Edit the config** at `src/config.py`, e.g., how many worker nodes you want

## Use

Deploy the Talos cluster:
```bash
pulumi up
```

The deployment will:
1. Create Hetzner Cloud server(s) with official Talos ISO (122630)
   - 1 control plane node
   - N worker nodes (configurable, default: 1)
2. Boot Talos Linux 1.11.2 from ISO on all nodes
3. Generate secrets and apply machine configuration to all nodes
4. Install Talos to disk and bootstrap Kubernetes on control plane
5. Save talosconfig to `~/.talos/config`
6. Save kubeconfig to `~/.kube/config-talos`

## Cleanup

Destroy all resources:
```bash
pulumi destroy
```

## Accessing the Cluster

### Using Makefile shortcuts (recommended)

```bash
# Check cluster health
make health      

# Export kubeconfig
make kubeconfig    

# Open Talos dashboard (control plane)   
make talos-dashboard  
```

### Using talosctl

```bash
export TALOSCONFIG=~/.talos/config

# Check cluster health
talosctl --nodes $(pulumi stack output control_plane_ip) --endpoints $(pulumi stack output control_plane_ip) health

# Open Talos dashboard (control plane)
talosctl --nodes $(pulumi stack output control_plane_ip) --endpoints $(pulumi stack output control_plane_ip) dashboard

# View logs from control plane
talosctl --nodes $(pulumi stack output control_plane_ip) --endpoints $(pulumi stack output control_plane_ip) logs

# View logs from a specific worker node
WORKER_IP=$(pulumi stack output worker_ips --json | jq -r '.[0]')
talosctl --nodes $WORKER_IP --endpoints $(pulumi stack output control_plane_ip) logs
```

### Using kubectl

```bash
export KUBECONFIG=~/.kube/config-talos

# Check cluster info
kubectl cluster-info

# Get nodes (shows control plane + workers)
kubectl get nodes

# Get all pods
kubectl get pods -A
```

## Development

### Linting and type checking

```bash
# Run linting and type checking
make lint       

# Apply auto-fixes to linting and formatting
make format     
```

## Troubleshooting

### Server not ready for configuration
If `ConfigurationApply` fails with connection timeout, nodes may still be installing Talos. Wait a few minutes and try again.

### Bootstrap timeout
Bootstrap can take 5-10 minutes. Monitor with:
```bash
make health
```

### Kubeconfig not available
If kubeconfig retrieval fails, ensure the API server is responsive:
```bash
talosctl --nodes $(pulumi stack output control_plane_ip) --endpoints $(pulumi stack output control_plane_ip) service kubelet status
```

### Worker nodes not joining
Check worker node status individually:
```bash
# Get first worker IP
WORKER_IP=$(pulumi stack output worker_ips --json | jq -r '.[0]')
talosctl --nodes $WORKER_IP --endpoints $(pulumi stack output control_plane_ip) get members

# Or check logs
talosctl --nodes $WORKER_IP --endpoints $(pulumi stack output control_plane_ip) logs kubelet
```

## Important Notes

- **No SSH Access:** Talos doesn't support SSH by design. All management is done via the `talosctl` API.
- **Security:** All communication is encrypted by default with mTLS for API access.
- **Multi-node support:** Configure worker nodes in `src/config.py`
- **Control plane scheduling:** With 0 workers, make sure to allow workload scheduling (`allow_scheduling`) on the control plane node

## Production Considerations

This setup is educational and **not production-ready**. For production use, you would need:

- **Private Networks:** Isolate VMs in Hetzner private networks instead of exposing them publicly
- **Firewall Rules:** Restrict access to only necessary ports
- **High Availability Control Plane:** Multi-control-plane (3+ nodes) with load balancer
- **Monitoring & Backups:** Prometheus/Grafana monitoring and etcd backup strategy
- **Persistent Storage:** Configure persistent storage solution (Hetzner CSI driver, Longhorn, etc.)
- ... and much more!

## 🤝 Need help?

Need help transitioning to a production deployment? 

**[Contact me on LinkedIn](https://www.linkedin.com/in/schwargeo/)** | [Learn more at georg-schwarz.com](https://georg-schwarz.com)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
