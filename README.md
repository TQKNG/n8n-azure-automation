# N8N Azure Automation 🚀

Automated Infrastructure as Code deployment (IAC) for a production-ready n8n instance on Azure.

This repository provisions:
- Azure networking, NSG, and Ubuntu VM via Bicep
- Dockerized n8n + Caddy reverse proxy on the VM
- HTTPS with automatic TLS certificates (Let's Encrypt)
- Operational scripts for deployment, health checks, and backups
- GitHub Actions workflows for setup, deploy, post-DNS start, teardown, and automated releases

## Architecture

The deployment uses a modular Bicep template and cloud-init bootstrap:

1. `infrastructure/main.bicep` orchestrates modules and outputs deployment metadata.
2. `infrastructure/modules/nsg.bicep` creates inbound/outbound firewall rules.
3. `infrastructure/modules/network.bicep` creates VNet, subnet, public IP, and NIC.
4. `infrastructure/modules/cloud-init.bicep` renders container templates and systemd timers.
5. `infrastructure/modules/vm.bicep` provisions Ubuntu 22.04 with trusted launch.
6. VM cloud-init writes files under `/opt/n8n`, installs Docker, and starts containers.

Runtime stack:
- `n8n` container on internal port `5678`
- `caddy` container terminating TLS on `80/443` and proxying to n8n
- Persistent Docker volumes for n8n data and Caddy state/logs

## Repository Structure

```text
.
├─ .github/workflows/
│  ├─ 00-setup-repository.yml
│  ├─ 01-bootstrap-azure-oidc.yml
│  ├─ 02-verify-azure-oidc.yml
│  ├─ 03-deploy.yml
│  ├─ 04-start-n8n.yml
│  ├─ 05-teardown.yml
│  ├─ 06-pr-title-lint.yml
│  ├─ 07-release-please.yml
│  └─ 08-sync-version-file.yml
├─ infrastructure/
│  ├─ main.bicep
│  ├─ modules/
│  │  ├─ cloud-init.bicep
│  │  ├─ network.bicep
│  │  ├─ nsg.bicep
│  │  └─ vm.bicep
│  ├─ containers/
│  │  ├─ docker-compose.yml
│  │  ├─ caddy/
│  │  │  ├─ caddy.template
│  │  │  └─ caddy.env.template
│  │  ├─ n8n/
│  │  │  └─ n8n.env.template
│  │  └─ scripts/
│  │     ├─ deploy.sh
│  │     ├─ backup.sh
│  │     └─ health-check.sh
│  └─ scripts/
│     └─ cloud-init-template.yml
├─ .release-please-config.json
├─ .release-please-manifest.json
├─ CHANGELOG.md
├─ LICENSE
└─ README.md
```

## Prerequisites

- Azure subscription with permissions to create:
  - resource groups
  - virtual machines
  - networking resources
- GitHub repository admin access
- Domain name with ability to update DNS A records
- SSH public key for VM access
- GitHub Personal Access Token with `repo` scope (used in setup workflow)

## GitHub Actions Workflow Order

Run these workflows in order:

1. **00 - Setup Repository**
	- Creates repository secrets/variables
	- Creates `production` environment and generates `N8N_ENCRYPTION_KEY`
2. **01 - Bootstrap Azure OIDC**
	- Validates setup and prints local Azure CLI bootstrap commands if needed
3. **02 - Verify Azure OIDC**
	- Confirms OIDC authentication and basic Azure permissions
4. **03 - Deploy n8n Infrastructure**
	- Deploys Bicep templates and waits for cloud-init completion
	- Pauses for manual DNS update
5. **04 - Start N8N**
	- Use after DNS points to VM public IP
	- Restarts Caddy and verifies HTTPS endpoint
6. **05 - Teardown Infrastructure** (optional)
	- Deletes resource group after explicit `DELETE` confirmation

Workflow files:
- `.github/workflows/00-setup-repository.yml`
- `.github/workflows/01-bootstrap-azure-oidc.yml`
- `.github/workflows/02-verify-azure-oidc.yml`
- `.github/workflows/03-deploy.yml`
- `.github/workflows/04-start-n8n.yml`
- `.github/workflows/05-teardown.yml`

Release and versioning workflows:
- `.github/workflows/06-pr-title-lint.yml`
- `.github/workflows/07-release-please.yml`
- `.github/workflows/08-sync-version-file.yml`

Release configuration:
- `.release-please-config.json`
- `.release-please-manifest.json`

## Required Secrets and Variables

### Repository Secrets

- `AZURE_SUBSCRIPTION_ID`
- `AZURE_TENANT_ID`
- `AZURE_CLIENT_ID` (set after OIDC bootstrap)
- `SSH_PUBLIC_KEY`
- `GH_PAT`

### Environment Secrets (`production` by default)

- `N8N_ENCRYPTION_KEY` (auto-generated in setup workflow)

### Repository Variables

- `AZURE_REGION` (default: `eastus`)
- `VM_SIZE` (default: `Standard_B2ms`)
- `RESOURCE_GROUP_NAME` (default: `rg-n8n-prod`)
- `N8N_VERSION` (default: `latest`)

## Deployment Inputs

`03-deploy.yml` supports:
- `domain_name` (required)
- `vm_size`
- `azure_region`
- `resource_group_name`
- `n8n_version`
- `base_name` (3-10 chars)
- `environment` (`production`, `staging`, `development`)

Core Bicep parameters are defined in `infrastructure/main.bicep`.

## DNS and Go-Live

After `03 - Deploy n8n Infrastructure` completes:

1. Point your domain A record to the VM public IP from workflow output.
2. Wait for DNS propagation.
3. Run `04 - Start N8N` with:
	- `resource_group_name`
	- `environment`
4. Open `https://<your-domain>`.

## Operations on the VM

Cloud-init writes assets to `/opt/n8n`:
- Docker compose file
- Caddy and n8n env/config files
- operational scripts

Useful scripts:
- `/opt/n8n/scripts/deploy.sh`
- `/opt/n8n/scripts/backup.sh`
- `/opt/n8n/scripts/health-check.sh`
- `/opt/n8n/scripts/monitor.sh`

Systemd units/timers created by cloud-init:
- `n8n.service`
- `n8n-backup.timer` (daily, 02:00)
- `n8n-health.timer` (every 5 minutes)

## Security Notes

- NSG allows inbound `443`, `80`, and `22`.
- NSG explicitly denies direct inbound access to `5678`.
- Caddy handles TLS and security headers.
- VM uses SSH key auth only (password auth disabled).
- n8n encryption key is stored as a GitHub environment secret.

## Local Bicep Validation (Optional)

If you want to validate templates locally:

```bash
az bicep build --file infrastructure/main.bicep
for module in infrastructure/modules/*.bicep; do
  az bicep build --file "$module"
done
```



## Versioning and License

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution and pull request guidance.

- Current version: `0.1.0` (see `.version`)
- Changelog: `CHANGELOG.md`
- License: MIT (`LICENSE`)
