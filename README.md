# Azure LAMP Stack - Automated Deployment with Terraform + GitHub Actions

## 🎯 What This Project Does

This project automatically deploys a complete LAMP (Linux, Apache, MySQL, PHP) stack on Azure using:
- **Terraform** for infrastructure provisioning
- **Ansible** for application configuration
- **GitHub Actions** for CI/CD automation
- **OIDC** for secure, keyless authentication

When you push code to GitHub, it automatically:
1. Creates Azure infrastructure (VM, networking, security)
2. Installs and configures Apache, MySQL, and PHP
3. Makes your web server publicly accessible

---

## 🏗️ Architecture Overview

```
GitHub Actions (CI/CD)
    ↓
Azure OIDC Authentication (no secrets!)
    ↓
Terraform (Infrastructure as Code)
    ↓
Azure Resources Created:
    - Resource Group
    - Virtual Network (10.0.0.0/16)
    - Subnet (10.0.1.0/24)
    - Network Security Group (SSH + HTTP)
    - Public IP (Standard SKU)
    - Network Interface
    - Ubuntu 22.04 VM (Standard_B1ms)
    ↓
Ansible (Configuration Management)
    ↓
LAMP Stack Installed:
    - Apache2 web server
    - MySQL database
    - PHP runtime
```

---

## 🔐 How OIDC Authentication Works

**Traditional Method (BAD):**
- Store Azure credentials as GitHub secrets
- Credentials can be stolen/leaked
- Must manually rotate secrets
- Long-lived access

**OIDC Method (GOOD - What We Use):**
1. GitHub generates a temporary token when workflow runs
2. Azure validates the token against federated credential
3. Azure grants temporary access (no secrets stored!)
4. Token expires automatically after job completes

**AWS Equivalent:** This is like using IAM Roles for Service Accounts (IRSA) in EKS or GitHub OIDC with AWS.

---

## 📋 What We Built - Step by Step

### **Step 1: Created Azure Service Principal with OIDC**
```bash
# Created an app registration in Azure AD
APP_ID=$(az ad app create --display-name "github-oidc-lamp" --query appId -o tsv)

# Created service principal (like an IAM role in AWS)
az ad sp create --id $APP_ID

# Gave it Contributor permissions on subscription
az role assignment create \
  --role "Contributor" \
  --assignee $APP_ID \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# Set up OIDC trust relationship (federated credential)
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:ghyovanivielot/AZLAMP:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**What this does:** Tells Azure "trust tokens from GitHub Actions for this specific repo"

---

### **Step 2: Created Terraform Infrastructure Code**

**Files:**
- `infra/terraform/main.tf` - Defines all Azure resources
- `infra/terraform/variables.tf` - Configurable parameters
- `infra/terraform/outputs.tf` - Exports public IP for Ansible

**Key Resources:**
- **Resource Group:** Container for all resources
- **VNet + Subnet:** Network isolation (like AWS VPC)
- **NSG:** Firewall rules (like AWS Security Groups)
- **Public IP:** Standard SKU (required by subscription limits)
- **VM:** Ubuntu 22.04 with SSH key authentication

---

### **Step 3: Created Ansible Configuration**

**File:** `infra/ansible/playbook.yml`

Installs:
- Apache2 (web server)
- MySQL (database)
- PHP + php-mysql (application runtime)

**Why Ansible after Terraform?**
- Terraform = infrastructure (servers, networks)
- Ansible = configuration (software, settings)
- Separation of concerns (like AWS: CloudFormation + Systems Manager)

---

### **Step 4: Created GitHub Actions Workflow**

**File:** `.github/workflows/deploy.yml`

**Two Jobs:**

**Job 1 - Terraform:**
1. Checkout code
2. Login to Azure using OIDC (no secrets!)
3. Install Terraform
4. Import existing resources (if any)
5. Run `terraform apply` to create/update infrastructure
6. Output public IP address

**Job 2 - Ansible:**
1. Wait for Terraform to complete
2. Install Ansible
3. Create inventory file with VM IP
4. Add SSH private key
5. Run playbook to install LAMP stack

---

### **Step 5: Configured GitHub Secrets**

**Secrets Added:**
- `AZURE_CLIENT_ID` - App registration ID
- `AZURE_TENANT_ID` - Azure AD tenant
- `AZURE_SUBSCRIPTION_ID` - Target subscription
- `SSH_PUB_KEY` - Public key for VM access
- `SSH_PRIVATE_KEY` - Private key for Ansible connection

**Note:** No Azure credentials stored! Only metadata for OIDC.

---

## 🔧 Troubleshooting We Fixed

### **Issue 1: SSH Key Format**
**Problem:** Terraform tried to read SSH key from file path
**Fix:** Changed to accept key content directly from GitHub secret
```hcl
# Before: public_key = file(var.ssh_pub_key)
# After:  public_key = var.ssh_pub_key
```

### **Issue 2: Client Secret Authentication**
**Problem:** Workflow used old authentication method
**Fix:** Updated to OIDC with `id-token: write` permission

### **Issue 3: Terraform Not Installed**
**Problem:** GitHub runner didn't have Terraform
**Fix:** Added `hashicorp/setup-terraform@v3` action

### **Issue 4: Duplicate Output**
**Problem:** Output defined in both `main.tf` and `outputs.tf`
**Fix:** Removed from `main.tf`, kept in `outputs.tf`

### **Issue 5: Basic SKU Public IP Limit**
**Problem:** Subscription blocked Basic SKU public IPs
**Fix:** Changed to Standard SKU: `sku = "Standard"`

### **Issue 6: Resource Already Exists**
**Problem:** Previous failed run left resources in Azure
**Fix:** Added import step to bring existing resources into Terraform state

---

## 🚀 How to Use This Project

### **Deploy Changes:**
```bash
# Make changes to code
git add .
git commit -m "Your changes"
git push origin main
```

GitHub Actions automatically deploys!

### **View Deployment:**
1. Go to: https://github.com/ghyovanivielot/AZLAMP/actions
2. Click latest workflow run
3. Watch Terraform and Ansible jobs
4. Get public IP from Terraform output

### **Test Your Server:**
```bash
# Get the public IP from GitHub Actions output
curl http://<PUBLIC_IP>
```

You should see Apache's default page.

### **SSH into VM:**
```bash
ssh -i ~/.ssh/id_rsa azureuser@<PUBLIC_IP>
```

---

## 🧹 Clean Up Resources

To delete everything and avoid charges:

```bash
cd infra/terraform
terraform destroy -auto-approve -var="ssh_pub_key=$(cat ~/.ssh/id_rsa.pub)"
```

Or via Azure CLI:
```bash
az group delete --name lamp-rg --yes
```

---

## 📊 Cost Estimate

**Resources:**
- Standard_B1ms VM: ~$15/month
- Standard Public IP: ~$3/month
- Network bandwidth: ~$1/month

**Total: ~$19/month** (if running 24/7)

**Tip:** Stop the VM when not using it:
```bash
az vm deallocate --resource-group lamp-rg --name lamp-vm
```

---

## 🔄 AWS to Azure Translation

| AWS | Azure | This Project |
|-----|-------|--------------|
| IAM Role | Service Principal | `github-oidc-lamp` |
| VPC | Virtual Network | `lamp-vnet` |
| Subnet | Subnet | `lamp-subnet` |
| Security Group | NSG | `lamp-nsg` |
| EC2 | Virtual Machine | `lamp-vm` |
| Elastic IP | Public IP | `lamp-pip` |
| CloudFormation | Terraform | `infra/terraform/` |
| Systems Manager | Ansible | `infra/ansible/` |
| GitHub OIDC | Federated Credential | OIDC trust |

---

## 🎓 What You Learned

1. **OIDC Authentication** - Keyless, secure cloud access
2. **Terraform** - Infrastructure as Code on Azure
3. **Ansible** - Configuration management
4. **GitHub Actions** - CI/CD automation
5. **Azure Networking** - VNets, NSGs, Public IPs
6. **Azure Compute** - Virtual Machines
7. **Troubleshooting** - Import resources, fix SKU limits

---

## 🔜 Next Steps to Enhance

1. **Add HTTPS** - Use Let's Encrypt for SSL
2. **Add Database** - Deploy Azure Database for MySQL
3. **Add Monitoring** - Azure Monitor + Log Analytics
4. **Add Backup** - Azure Backup for VM
5. **Add Scaling** - VM Scale Sets
6. **Add Load Balancer** - Azure Load Balancer
7. **Add DNS** - Azure DNS with custom domain
8. **Add CI/CD for App** - Deploy PHP app automatically

---

## 📚 Key Files Reference

```
AZLAMP/
├── .github/workflows/
│   └── deploy.yml              # CI/CD pipeline
├── infra/
│   ├── terraform/
│   │   ├── main.tf            # Azure resources
│   │   ├── variables.tf       # Input parameters
│   │   └── outputs.tf         # Export values
│   └── ansible/
│       └── playbook.yml       # LAMP installation
└── README.md                  # This file
```

---

## 🆘 Common Commands

**Check Azure resources:**
```bash
az resource list --resource-group lamp-rg --output table
```

**Get VM public IP:**
```bash
az vm show -d --resource-group lamp-rg --name lamp-vm --query publicIps -o tsv
```

**View NSG rules:**
```bash
az network nsg rule list --resource-group lamp-rg --nsg-name lamp-nsg --output table
```

**Terraform commands (local):**
```bash
cd infra/terraform
terraform init
terraform plan -var="ssh_pub_key=$(cat ~/.ssh/id_rsa.pub)"
terraform apply -var="ssh_pub_key=$(cat ~/.ssh/id_rsa.pub)"
```

---

## ✅ Success Criteria

Your deployment is successful when:
- ✅ GitHub Actions workflow completes without errors
- ✅ All Azure resources exist in `lamp-rg` resource group
- ✅ VM has public IP assigned
- ✅ You can SSH into the VM
- ✅ Apache responds on port 80
- ✅ MySQL service is running
- ✅ PHP is installed and working

---

**Built by:** Ghyovani Vielot  
**Purpose:** Azure skills development for cloud engineering role  
**Date:** December 2025
