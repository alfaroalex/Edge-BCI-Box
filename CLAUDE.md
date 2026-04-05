# CLAUDE.md

## Project Overview

Edge infrastructure platform on a 3-node Dell Tracewell XR4000 chassis. Runs a K3s cluster that hosts both Dell Federal partner demo workloads and BCI/neuroscience data processing workloads. Everything is managed as code — Ansible for bare metal, Terraform for the application stack, GitLab CI for pipelines, Bazel for builds.

## Hardware

- **XR4520** — K3s control plane + KVM host. Rocky Linux. Runs Vcinity VM under KVM.
- **XR4510 #1** — K3s worker. Fedora. Primary container workload node.
- **XR4510 #2** — K3s worker. Fedora. REDCOM Sigma host (KVM VM or bare metal) + container capacity.
- **Network switch** — VLAN segmentation between production and platform networks.

## Repo Structure

```
├── CLAUDE.md
├── README.md
├── flake.nix                  # Nix dev environment
├── ansible/
│   ├── inventory/
│   ├── playbooks/
│   │   ├── provision-rocky.yml
│   │   ├── provision-fedora.yml
│   │   ├── harden.yml
│   │   └── k3s-bootstrap.yml
│   └── roles/
├── packer/
│   ├── rocky.pkr.hcl
│   └── fedora.pkr.hcl
├── kickstart/
│   ├── rocky-ks.cfg
│   └── fedora-ks.cfg
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── modules/
│   │   ├── namespaces/
│   │   ├── helm-releases/
│   │   ├── networking/
│   │   ├── certificates/
│   │   ├── kvm-vms/            # libvirt provider for Vcinity, REDCOM
│   │   └── aws/                # VPC, VPN, S3, Lambda
│   └── environments/
│       ├── conference/
│       └── dev/
├── helm/
│   ├── tak-server/
│   ├── versa-sdwan/
│   ├── gitlab/
│   ├── monitoring/             # Prometheus, Grafana, Alertmanager
│   ├── mne-python/
│   ├── eegnet-inference/
│   ├── neural-telemetry/
│   └── openclaw/
├── rust/
│   └── telemetry-agent/        # DaemonSet — node metrics → Prometheus
│       ├── src/
│       ├── Cargo.toml
│       ├── Dockerfile
│       └── BUILD
├── python/
│   ├── health-check/           # Post-deploy stack validation
│   ├── neural-pipeline/        # Synthetic spike train → NATS → Grafana
│   └── eegnet-api/             # FastAPI wrapper for EEGNet inference
├── docker/
│   └── Dockerfiles for custom images
├── .gitlab-ci.yml
├── BUILD                       # Bazel root
├── WORKSPACE                   # Bazel workspace
└── .gitignore
```

## K8s Namespaces

- `dell-partners` — TAK Server, Versa SD-WAN (partner demo workloads)
- `platform` — GitLab, Prometheus, Grafana, Alertmanager, cert-manager, MinIO
- `bci-workloads` — MNE-Python, EEGNet inference, neural telemetry pipeline
- `infra-tools` — Rust telemetry DaemonSet, Python health checks, OpenClaw

## Tech Stack

- **Provisioning:** Kickstart, Packer, Cloud-Init, Ansible
- **Security:** Secure Boot, UEFI, TPM 2.0, cert-manager, mTLS, VLANs, 802.1x
- **Orchestration:** K3s, Helm, Docker
- **IaC:** Terraform (kubernetes, helm, libvirt, aws providers)
- **CI/CD:** GitLab (self-hosted on cluster), GitHub (public repo)
- **Build:** Bazel, Nix
- **Languages:** Python, Rust
- **Cloud:** AWS (VPC, VPN, S3, Lambda, EC2)
- **Monitoring:** Prometheus, Grafana, Alertmanager
- **Messaging:** NATS or Redis
- **Storage:** MinIO or Longhorn
- **Virtualization:** KVM (for Vcinity and REDCOM)
- **AI/Agent:** OpenClaw (ops alerts via Telegram)

## Conventions

- All secrets in `.gitignore` — no credentials in the repo ever.
- Partner-specific configs use Terraform variables, not hardcoded values.
- Commits should be descriptive — the Git history is part of the portfolio.
- Ansible handles everything below K3s. Terraform handles everything above.
- Bazel builds all container images. GitLab CI calls Bazel.
- Nix flake defines the dev environment — all tools versioned in `flake.nix`.

## Non-Containerizable Workloads

- **Vcinity ULT X** — KVM VM on XR4520. Managed via Terraform libvirt provider.
- **REDCOM Sigma** — KVM VM on XR4510 #2, or standalone hardware appliance via Ethernet. Not containerizable.

## Key Design Decisions

- **K3s over K8s** — lightweight, single-binary, appropriate for 3-node edge hardware.
- **Mixed OS (Rocky + Fedora)** — intentional, demonstrates heterogeneous fleet management.
- **Hybrid deployment (containers + VMs + appliances)** — reflects real-world infrastructure, not a container-only utopia.
- **GitHub + GitLab** — GitHub is public portfolio/source-of-truth. GitLab on the cluster runs CI/CD pipelines.
- **Terraform for everything above the OS** — single `terraform apply` deploys the full stack in ~15 minutes.
