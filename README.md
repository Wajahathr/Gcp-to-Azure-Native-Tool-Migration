# GCP to Azure Migration Guide

This document outlines the step-by-step process for migrating a Virtual Machine from Google Cloud Platform (GCP) to Microsoft Azure. The process involves setting up a discovery appliance, performing a manual disk export, and provisioning the migrated VM in Azure.

## Table of Contents
1. [GCP: Set up Dummy VM](#step-1-set-up-dummy-vm-on-gcp)
2. [Azure: Create Resource Group & VNet](#step-2-create-resource-group-and-vnet-on-azure)
3. [Azure: Azure Migrate Project Setup](#step-3-azure-migrate-project-setup)
4. [GCP: Create Discovery Appliance](#step-4-create-discovery-appliance-on-gcp)
5. [GCP: Setup Discovery Appliance](#step-5-set-up-discovery-appliance-on-gcp)
6. [GCP: Manual Disk Export](#step-6-manual-disk-export-from-gcp)
7. [Azure: Create Storage Account & Deploy VM](#step-7-create-storage-account--deploy-vm-on-azure)

---
### Step 1 to Step 5 for Enterprise level migration (later have add Replication appliance also)

### Step 1: Set-up Dummy VM on GCP
1. Create and set up the dummy VM in GCP.

2. SSH into the machine and set a password:
   ```bash
   sudo passwd <your_username>
   # Save this password securely
   ```
3. Enable password authentication:
   ```bash
   sudo nano /etc/ssh/sshd_config  
   # Remove the '#' to uncomment the following line:
   PasswordAuthentication yes
   
   sudo systemctl restart ssh
   ```
4. Create a firewall rule to allow SSH discovery:
   * **Name:** `allow-ssh-discovery`
   * **Network:** `default` (VPC)
   * **Targets:** All instances in the network
   * **Source IPv4 ranges:** `0.0.0.0/0`
   * **Protocols and ports:** `TCP: 22`

---

### Step 2: Create Resource Group and VNet on Azure
1. Search for **Resource Groups** in the Azure search bar.

2. Click **Create** and select your Subscription, Name, and Region.

3. **Create Virtual Network (VNet):**
   * Search for **Virtual networks**.
   * Select the newly created Resource Group.
   * Provide a name for the Virtual Network.
   * Leave everything else as default and click **Create**.

---

### Step 3: Azure Migrate Project Setup

1. Search for **Azure Migrate** in the search bar.
2. Click **Create Project**.
3. Select your Subscription and the previously created Resource Group.
4. Name the migrate project, select the target region, and click **Create**.

---

### Step 4: Create Discovery Appliance on GCP

Create a new VM in GCP with the following specifications to act as the Discovery Appliance:
* **OS:** Windows Server 2022
* **Compute:** Minimum 8 vCPUs, 16 GB RAM
* **Storage:** 80 GB Hard Disk
* **Firewall:** Allow HTTP & HTTPS traffic

---

### Step 5: Set-up Discovery Appliance on GCP

1. **Sync Time:** Open PowerShell as Administrator on the appliance and run:
   ```powershell
   Set-TimeZone -Id "Pakistan Standard Time"
   net stop w32time
   w32tm /config /syncfromflags:manual /manualpeerlist:"time.windows.com"
   net start w32time
   w32tm /resync
   Set-Service -Name w32time -StartupType Automatic
   Start-Service -Name w32time
   w32tm /resync
   ```

2. **Setup Access:** Set up a Windows password (save this) and note the appliance's External IP via GCP console. Use MobaXterm (or RDP) to log in.

3. **Login to Azure:** Open Microsoft Edge on the appliance, navigate to the Azure Portal, and log in.

4. **Generate Key:** * Navigate to Azure Migrate -> Overview -> Discovery -> Discover using appliance.
   * Select your appliance, click **Generate key** (save this key).
   * Download the `.zip` file, extract the folder, and copy the file path.

5. **Install Appliance:** Open PowerShell as Administrator, navigate to the extracted folder, and run:
   ```powershell
   cd <copied-file-path>
   Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process
   .\AzureMigrateInstaller.ps1
   ```
6. **Configure Appliance Manager:**

   * Open the Appliance Manager on the VM.
   * Verify using the generated discovery key and log in.
   * Add credentials for the Linux VM: Select `Linux with password`, provide a Friendly server name, the VM's Username, and Password.

7. **Add Discovery Source:** Specify OS type, Internal IP of the target GCP VM, and select the Friendly name.

8. Click **Start discovery**.

9. Return to the Azure Migrate portal and check **Explore Inventory** to verify the machine is showing up.

---

### Step 6 to Step 7: For Manual (Lift and Shift migration).

### Step 6: Manual Disk Export from GCP

1. **Create Image:** Stop the target VM. Go to Images -> Create Image. Name it, select `Disk` as source, choose your VM's disk, and create.

2. **Create Bucket:** Go to Cloud Storage -> Buckets -> Create, and name your bucket (e.g., `wajahat-migration-bucket-2026`).

3. **Export Image:** Open Google Cloud Shell and run:
   ```bash
   gcloud compute images export \
     --destination-uri=gs://<YOUR_BUCKET_NAME>/ubuntu-dummy.vhd \
     --image=<IMAGE_NAME> \
     --export-format=vhd
   ```
4. **Verify:** Check Cloud Storage -> Buckets to ensure the `.vhd` file is present.

5. **Make Public (Temporary for Transfer):** Run the following in Cloud Shell:
   ```bash
   gsutil acl ch -u AllUsers:R gs://<YOUR_BUCKET_NAME>/ubuntu-dummy.vhd
   ```

---

### Step 7: Create Storage Account & Deploy VM on Azure

1. **Storage Account:** Search for Storage Accounts -> Create. Select your Resource Group, provide a name (e.g., `wajahatmigrationstore26`), select the region, and create.
2. **Container:** Go to Data Storage -> Containers. Create a new container (e.g., `vhd-files`).
3. **Copy Blob to Azure:** Open Azure Cloud Shell (Bash) and run the following commands:
   ```bash
   # Copy VHD from GCP Bucket to Azure Storage
   az storage blob copy start \
     --account-name wajahatmigrationstore26 \
     --destination-container vhd-files \
     --destination-blob ubuntu-dummy.vhd \
     --source-uri "[https://storage.googleapis.com/wajahat-migration-bucket-2026/ubuntu-dummy.vhd](https://storage.googleapis.com/wajahat-migration-bucket-2026/ubuntu-dummy.vhd)" \
     --auth-mode login

   # Check the copy status
   az storage blob show \
     --account-name wajahatmigrationstore26 \
     --container-name vhd-files \
     --name ubuntu-dummy.vhd \
     --query "properties.copy.status" \
     --auth-mode login \
     --output tsv
   ```
4. **Create Managed Disk:**
   ```bash
   az disk create \
     --resource-group <YOUR_RESOURCE_GROUP> \
     --name migrated-ubuntu-disk \
     --source [https://wajahatmigrationstore26.blob.core.windows.net/vhd-files/ubuntu-dummy.vhd](https://wajahatmigrationstore26.blob.core.windows.net/vhd-files/ubuntu-dummy.vhd) \
     --location eastus
   ```
5. **Create Target VM:**
   ```bash
   az vm create \
     --resource-group <YOUR_RESOURCE_GROUP> \
     --name Migrated-Ubuntu-VM \
     --attach-os-disk migrated-ubuntu-disk \
     --os-type linux \
     --location eastus \
     --size Standard_B2s
   ```
```

***

