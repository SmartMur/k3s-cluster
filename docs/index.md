# k3s Cluster — Proxmox + TrueNAS

[![Platform](https://img.shields.io/badge/Platform-k3s%20on%20Proxmox-0ea5e9)](#)
[![IaC](https://img.shields.io/badge/IaC-Terraform%20%2B%20Ansible-22c55e)](#)
[![Security](https://img.shields.io/badge/Security-Scrubbed%20%E2%9C%93-22c55e)](#)
[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-1f6feb?logo=pre-commit)](https://pre-commit.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/SmartMur/smartmur-homelab-k3s/blob/main/LICENSE)

Production-oriented automation for provisioning and operating a multi-node k3s cluster across Proxmox hosts, with TrueNAS-backed persistent storage and Traefik ingress.

Beginner journey (from scratch): [Beginner Journey](getting-started/beginner-journey.md)
Quick operator start: [Getting Started](getting-started/getting-started.md)
Kubernetes manager + Dockhand sync: [Dockhand + Headlamp Workflow](guides/dockhand-headlamp-workflow.md)
Notify monitoring + alerting: [Notify Monitoring](operations/notify-monitoring.md)
Latest live ops audit: [Operations Audit](operations/operations-audit.md)
Contributing: [Contributing](reference/contributing.md)
Security policy: [Security](reference/security.md)
Security rulebook: [Security Rulebook](reference/security-rulebook.md)
Deep environment reference: [Stack](guides/stack.md)

## Overview

- Terraform VM provisioning across multiple Proxmox hosts
- Ansible-based cluster bootstrap and app deployment
- Traefik ingress with MetalLB LoadBalancer IP
- Native Kubernetes web manager (Headlamp) exposed via Traefik
- Mixed persistent storage (`truenas-nfs` + `local-path`) based on live workload constraints
- Security-first manifest policy: tracked app manifests contain no plaintext secret values

## Architecture

```text
Mac/Linux control node
  |
  |- Terraform -> Proxmox PVE1 + PVE2
  |               |- k3s-master-01
  |               |- k3s-worker-01
  |               |- k3s-worker-02
  |               '- k3s-worker-03
  |
  |- Ansible -> k3s bootstrap + addons
  |             |- MetalLB
  |             |- Traefik
  |             '- NFS storage class
  |
  '- kubectl -> app manifests -> PVCs on TrueNAS NFS
```

## Repo Structure

```text
.
|- ansible/                # cluster + apps playbooks
|- manifests/              # namespaces, ingress, app manifests
|  |- apps/
|  |- ingress/
|  '- secrets/             # local-only overlays (ignored)
|- scripts/                # end-to-end lifecycle scripts
|- terraform/              # Proxmox infrastructure definitions
|- docs/                   # operator documentation and security runbooks
|- STACK.md                # full environment reference
'- README.md
```

## Requirements

- `bash`, `git`, `python3`
- `terraform`, `ansible`, `kubectl`, `helm`
- Proxmox API access + template VM available
- TrueNAS NFS export for persistent volumes

## Quick Start

```bash
# 1) Install tooling
bash scripts/00-install-tools.sh

# 2) Prepare Proxmox template (one-time)
bash scripts/00-create-proxmox-template.sh

# 3) Configure Terraform vars locally
cp terraform/terraform.tfvars.example terraform/terraform.tfvars
# edit terraform/terraform.tfvars (do not commit)

# 4) Provision VMs
bash scripts/01-provision.sh

# 5) Bootstrap k3s
export K3S_TOKEN="CHANGE_ME_MIN_20_CHARS"
bash scripts/02-cluster-setup.sh

# 6) Configure storage
bash scripts/03-storage-setup.sh

# 7) Create runtime secrets locally (see manifests/secrets/README.md)
#    Then deploy apps
bash scripts/04-deploy-apps.sh

# 8) Sync kube contexts into Dockhand tracked environments
bash scripts/05-sync-dockhand-contexts.sh
```

## Security Model

- Real credentials must never be committed.
- App manifests reference Secrets by name only; they do not carry real secret values.
- Runtime secrets should be created in `manifests/secrets/*.yml` (ignored) or directly with `kubectl create secret ...`.
- Local sensitive artifacts (`.k3s-node-token`, `kubeconfig-raw.yml`, `terraform/terraform.tfvars`, `terraform.tfstate`) are ignored by `.gitignore`.

Example secret creation:

```bash
kubectl create secret generic vaultwarden-secret -n vaultwarden \
  --from-literal=ADMIN_TOKEN='CHANGE_ME' \
  --from-literal=DOMAIN='https://vault.smartmur.ca' \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Professional Checks (Before Push)

```bash
bash -n scripts/*.sh
pre-commit run --all-files
python3 scripts/security_scrub.py
```

Install hooks once per clone:

```bash
brew install pre-commit
pre-commit install
```

## Automation

- CI checks: `.github/workflows/ci.yml`
- Dependabot updates: `.github/dependabot.yml`
- Dependabot auto-merge flow: `.github/workflows/dependabot-automerge.yml`

## Troubleshooting

- Cluster not reachable: `kubectl cluster-info`
- App rollout stuck: `kubectl get pods -A` and `kubectl logs -n <ns> deploy/<app>`
- Traefik VIP missing: `kubectl get svc traefik -n traefik`
- Placeholder guard triggered: replace `CHANGE_ME_*` values via secret creation flow

## Teardown

```bash
bash scripts/99-teardown.sh
```
