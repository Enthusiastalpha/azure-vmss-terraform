# Azure VMSS — Highly Available Web Infrastructure with Terraform

A production-grade Azure Virtual Machine Scale Set infrastructure built with Terraform, featuring Azure Key Vault integration, remote state management, and a real-time monitoring dashboard.

---

## Architecture

```
Internet
    │
    ▼
Azure Standard Load Balancer (dev-web-lb)
    │  zone-redundant · health probes · NAT rules
    ▼
Network Security Group (dev-web-nsg)
    │  HTTP 80 · HTTPS 443 · SSH 22
    ▼
Virtual Network (dev-web-vnet) · Subnet 10.0.1.0/24
    │
    ├── Zone 1: dev-web-vmss instance
    ├── Zone 2: dev-web-vmss instance
    └── Zone 3: dev-web-vmss instance
         │
         ├── Apache + PHP metrics API
         ├── Real-time VMSS Monitor dashboard
         └── NAT Gateway (outbound traffic)

Security Layer
    ├── Azure Key Vault (tfsecretdev) — SPN, SSH key, storage key
    └── Remote State — Azure Blob Storage (tfstate container)
```

---

## Features

- **High Availability** — 3 VMSS instances spread across Availability Zones 1, 2 and 3
- **Zero secrets on disk** — all credentials fetched from Azure Key Vault at runtime via OAuth2
- **Remote Terraform state** — stored in Azure Blob Storage, state key loaded from Key Vault
- **Live monitoring dashboard** — real CPU, memory, disk, network and Apache logs served from the VM
- **Dynamic NSG rules** — driven from a locals map, no hardcoded rules in resource blocks
- **Variable validation** — environment and instance count validated at plan time
- **Consistent naming** — every resource follows `${environment}-web-<resource>` pattern

---

## Project Structure

```
├── backend.tf          # Remote state configuration (Azure Blob Storage)
├── provider.tf         # Azure provider configuration
├── variables.tf        # Input variables with type constraints and validations
├── locals.tf           # Computed values and dynamic NSG rules map
├── vnet.tf             # VNet, Subnet, NSG, Load Balancer, NAT Gateway
├── vmss.tf             # Virtual Machine Scale Set
├── keyvault.tf         # Key Vault data sources for secrets
├── autoscale.tf        # Autoscale settings (disabled — sandbox policy)
├── outputs.tf          # LB public IP, FQDN, SSH instructions
├── terraform.tfvars    # Non-sensitive variable values
├── userdata.sh         # Bootstrap script — Apache, PHP metrics API, dashboard
└── .gitignore          # Excludes state files, credentials, tfvars secrets
```

---

## Prerequisites

- Terraform >= 1.5.0
- Azure subscription
- Azure Key Vault with the following secrets:
  - `spn-client-id`
  - `spn-client-secret`
  - `spn-tenant-id`
  - `spn-subscription-id`
  - `ssh-public-key`
  - `storage-account-key`
- Azure Storage Account with a container named `tfstate`
- Service Principal with Contributor role on the resource group

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/azure-vmss-terraform.git
cd azure-vmss-terraform
```

### 2. Create `set-env.ps1` (gitignored — never committed)

```powershell
# set-env.ps1 — fill in your values, keep this file local only
$kvName       = "your-keyvault-name"
$tenantId     = "your-tenant-id"
$clientId     = "your-client-id"
$clientSecret = "your-client-secret"

# Fetches all secrets from Key Vault into environment variables
$token = (Invoke-RestMethod -Method Post `
  -Uri "https://login.microsoftonline.com/$tenantId/oauth2/token" `
  -Body @{ grant_type="client_credentials"; client_id=$clientId; client_secret=$clientSecret; resource="https://vault.azure.net" }).access_token

function Get-KVSecret($name) {
  (Invoke-RestMethod -Uri "https://$kvName.vault.azure.net/secrets/$name/?api-version=7.4" `
    -Headers @{ Authorization = "Bearer $token" }).value
}

$env:ARM_CLIENT_ID       = Get-KVSecret "spn-client-id"
$env:ARM_CLIENT_SECRET   = Get-KVSecret "spn-client-secret"
$env:ARM_TENANT_ID       = Get-KVSecret "spn-tenant-id"
$env:ARM_SUBSCRIPTION_ID = Get-KVSecret "spn-subscription-id"
$env:ARM_ACCESS_KEY      = Get-KVSecret "storage-account-key"
```

### 3. Update `terraform.tfvars`

```hcl
environment         = "dev"
location            = "East US 2"
resource_group_name = "your-resource-group"
key_vault_name      = "your-keyvault-name"
```

### 4. Update `backend.tf`

```hcl
backend "azurerm" {
  resource_group_name  = "your-resource-group"
  storage_account_name = "your-storage-account"
  container_name       = "tfstate"
  key                  = "vmss/terraform.tfstate"
}
```

### 5. Deploy

```powershell
# Load credentials from Key Vault
.\set-env.ps1

# Initialise Terraform with remote backend
terraform init

# Preview changes
terraform plan

# Deploy
terraform apply
```

---

## Accessing the Dashboard

After `terraform apply` completes:

```powershell
# Get the public URL
terraform output lb_public_ip_fqdn

# Or direct IP
terraform output lb_public_ip
```

Open the URL in your browser — you'll see the **VMSS Monitor** dashboard showing real-time metrics from whichever instance the load balancer routes you to. Refresh to hit a different zone.

---

## SSH Access

```powershell
# Instance 0 (Zone 1) — port 50000
ssh -i ~/.ssh/your-key azureuser@<LB_PUBLIC_IP> -p 50000

# Instance 1 (Zone 2) — port 50001
ssh -i ~/.ssh/your-key azureuser@<LB_PUBLIC_IP> -p 50001

# Instance 2 (Zone 3) — port 50002
ssh -i ~/.ssh/your-key azureuser@<LB_PUBLIC_IP> -p 50002
```

---

## Security Notes

- SPN credentials are never stored on disk — loaded into environment variables via `set-env.ps1` for the session only
- SSH public key is read directly from Key Vault by Terraform at apply time
- `terraform.tfvars` contains no sensitive values — safe to commit
- `set-env.ps1` is gitignored — never committed
- Terraform state is stored remotely in Azure Blob Storage — not locally

---

## Known Limitations

- Autoscale (`azurerm_monitor_autoscale_setting`) is disabled — blocked by the Azure playground subscription policy. The configuration is written and ready in `autoscale.tf` — uncomment to enable in a real subscription.
- Manual scaling via CLI: `az vmss scale --name dev-web-vmss --resource-group <rg> --new-capacity <n>`

---


## Resources

- [Terraform Azure Provider Docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Azure VMSS Documentation](https://docs.microsoft.com/azure/virtual-machine-scale-sets/)
- [Azure Key Vault](https://docs.microsoft.com/azure/key-vault/)

---

## License

MIT
