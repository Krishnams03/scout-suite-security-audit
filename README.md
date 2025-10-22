# 🔍 Scout Suite on Azure — Step-by-Step (PowerShell compatible, Southeast Asia)

**Purpose:** Deploy a small Azure lab (VM + resources), install and run **Scout Suite** (multi-cloud security auditor), capture the HTML report, and demonstrate findings for a security audit.

**Region used in this guide:** `southeastasia` (works well for many student/sandbox subscriptions)

---

## Table of Contents
1. Prerequisites
2. Quick overview (what this will create)
3. Step-by-step instructions
   - Login to Azure
   - Create resource group
   - Create VM (PowerShell multiline)
   - Open SSH port
   - Connect to VM
   - Install Scout Suite on VM
   - Authenticate Azure inside VM (for Scout Suite)
   - Run Scout Suite
   - Retrieve and view report
4. Expected findings (sample)
5. Troubleshooting & common errors
6. Cleanup
7. Repository layout & notes

---

## 1. Prerequisites

- Azure account (Free/Student/subscription)
- Azure CLI installed (`az`) — install from https://learn.microsoft.com/cli/azure/install-azure-cli
- PowerShell (Windows / macOS / Linux) — this guide uses PowerShell-compatible commands
- Local SSH client (for `ssh` and `scp`)

---

## 2. Quick overview (what we will create)

All resources will be created inside one resource group `scout-rg` in `southeastasia`:
- Ubuntu VM `scout-vm` (SSH access)
- Network Security Group (default) and Public IP
- Optional: Storage account or Key Vault if you add them manually later

Working in a single resource group makes cleanup trivial.

---

## 3. Step-by-step instructions

> All PowerShell multiline commands below use the backtick `` ` `` as the line continuation character.  
> Replace `<your-vm-ip>` when instructed with the public IP shown after VM creation.

### A. Login to Azure

az login

### If your account has no subscriptions and you only need tenant level access:

az login --allow-no-subscriptions

### B. Create resource group (Southeast Asia)
$region = "southeastasia"
az group create --name scout-rg --location $region

### C. Create an Ubuntu VM (PowerShell-compatible multiline)
az vm create `
  --resource-group scout-rg `
  --name scout-vm `
  --image Ubuntu2204 `
  --admin-username azureuser `
  --generate-ssh-keys `
  --location $region


## Note: after this command completes, Azure prints the VM public IP — copy that IP for SSH.

### D. Open SSH port (22) on the VM
az vm open-port `
  --port 22 `
  --resource-group scout-rg `
  --name scout-vm

### E. Connect to the VM using SSH

Replace <your-vm-ip> with the public IP from the VM creation output:

ssh azureuser@<your-vm-ip>

### F. On the VM — update & install required packages

(Execute these commands inside the VM after SSH)

# Update and install Python + pip + git
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip git

### G. Install Scout Suite inside the VM
# Install scoutsuite using pip
pip3 install scoutsuite

# Quick verification
scout --help


## If scout is not on PATH, run:

python3 -m scoutsuite --help

### H. Authenticate Azure for Scout Suite from the VM

## If your VM has a browser (most won't), you can run:

az login


## If no browser access, use device code flow (recommended inside VM):

az login --use-device-code
# Follow the printed URL and device code on your local machine's browser to authenticate

I. Run Scout Suite for Azure

Run the Azure scan (this may take several minutes depending on your subscription/resources):

# Basic Azure run
scout --provider azure

# Or if the above fails, explicitly:
scout azure --cli


Scout Suite will create a report directory (e.g., scoutsuite-report) containing an HTML report and supporting files.

### J. Retrieve and view the report locally

Exit the VM (if still connected):

exit


Copy the report from the VM to your local machine using scp. Replace <your-vm-ip> and the path as needed:

# Example PowerShell scp usage (run locally, not inside VM)
scp azureuser@<your-vm-ip>:/home/azureuser/scoutsuite-report/*.html . 


Open the copied HTML file in your browser (double-click or open).

If you prefer to view directly on the VM, you can install a lightweight HTTP server and download the page via browser using the VM public IP (not recommended for production):

# inside VM - serve the report directory (temporary)
cd scoutsuite-report
python3 -m http.server 8000
# Then from your local browser: http://<your-vm-ip>:8000/


Note: Exposing ports to the public internet may increase risk. Use only for short testing periods and remove the server after download.

### 4. Expected findings (sample)

When you run Scout Suite in this lab environment, possible findings include:

Network Security Group: Open SSH (22) from Any — flagged as risky (recommend restrict to trusted IPs).

Public IP: VM with public IP — informational, consider jumpbox or bastion.

Storage: If you create a storage account with public access, Scout Suite will flag public blob access.

Key Vault: If public network access enabled, flagged as medium risk.

Document any high/medium issues found in your report README and include screenshots of the Scout Suite HTML.

### 5. Troubleshooting & common errors

Error: RequestDisallowedByAzure

Meaning: Your subscription is restricted to certain regions by policy.

Fix: Use a permitted region (e.g., southeastasia, eastus, northeurope) or check Portal’s region dropdown.
Example: az group create --name scout-rg --location southeastasia

Error: az not recognized

Install Azure CLI and restart your terminal.

Error: Scout Suite missing or not found

Use pip3 install scoutsuite and ensure ~/.local/bin or Python scripts folder is in PATH, or run with python3 -m scoutsuite.

SSH cannot connect

Verify VM public IP and NSG inbound rule allowing port 22, and ensure local firewall/ISP not blocking outbound SSH.

### 6. Cleanup (delete everything created by this guide)

Remove the resource group (deletes all resources inside scout-rg only):

az group delete --name scout-rg --yes --no-wait


If you want a safer workflow (preview resources to be deleted first), list resources:

az resource list --resource-group scout-rg -o table


Then delete the group after confirming.
