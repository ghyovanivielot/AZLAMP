# Azure LAMP Stack with Terraform + GitHub Actions (OIDC)

## Prerequisites
- Azure subscription
- GitHub account
- Azure CLI installed locally

## Setup Instructions

### 1. Create Azure Service Principal with OIDC

```bash
# Login to Azure
az login

# Get your subscription ID
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
echo "Subscription ID: $SUBSCRIPTION_ID"

# Create the app registration
APP_ID=$(az ad app create --display-name "github-oidc-lamp" --query appId -o tsv)
echo "App ID: $APP_ID"

# Create service principal
az ad sp create --id $APP_ID

# Assign Contributor role
az role assignment create \
  --role "Contributor" \
  --subscription $SUBSCRIPTION_ID \
  --assignee $APP_ID

# Get tenant ID
TENANT_ID=$(az account show --query tenantId -o tsv)
echo "Tenant ID: $TENANT_ID"

# Create federated credential for OIDC
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-oidc-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:YOUR_GITHUB_USERNAME/AZLAMP:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**IMPORTANT:** Replace `YOUR_GITHUB_USERNAME` with your actual GitHub username in the command above.

### 2. Generate SSH Keys (if you don't have them)

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

### 3. Configure GitHub Secrets

Go to your GitHub repo → Settings → Secrets and variables → Actions → New repository secret

Add these secrets:

| Secret Name | Value | How to Get |
|------------|-------|------------|
| `AZURE_CLIENT_ID` | Your App ID | From step 1 output |
| `AZURE_TENANT_ID` | Your Tenant ID | From step 1 output |
| `AZURE_SUBSCRIPTION_ID` | Your Subscription ID | From step 1 output |
| `SSH_PUB_KEY` | Your public key | `cat ~/.ssh/id_rsa.pub` |
| `SSH_PRIVATE_KEY` | Your private key | `cat ~/.ssh/id_rsa` |

### 4. Push to GitHub

```bash
git add .
git commit -m "Initial LAMP stack setup"
git push -u origin main
```

The GitHub Action will automatically trigger and deploy your infrastructure.

## What Gets Deployed

- Resource Group
- Virtual Network (10.0.0.0/16)
- Subnet (10.0.1.0/24)
- Network Security Group (allows SSH + HTTP)
- Ubuntu 22.04 VM (Standard_B1ms)
- Apache, MySQL, PHP installed via Ansible

## Customization

Edit `infra/terraform/variables.tf` to change:
- `prefix`: Resource naming prefix (default: "lamp")
- `location`: Azure region (default: "eastus")

## Testing

After deployment completes, get the public IP from GitHub Actions output and visit:
```
http://<PUBLIC_IP>
```

You should see the Apache default page.

## Clean Up

To destroy all resources:
```bash
cd infra/terraform
terraform destroy -auto-approve -var="ssh_pub_key=$(cat ~/.ssh/id_rsa.pub)"
```
