# MatterMost Azure Deployment - Setup Guide

This guide provides step-by-step instructions for deploying the MatterMost application on Microsoft Azure with a secure, two-tier architecture.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Phase 1: Azure Infrastructure Setup](#phase-1-azure-infrastructure-setup)
- [Phase 2: Database Server Configuration](#phase-2-database-server-configuration)
- [Phase 3: MatterMost Server Configuration](#phase-3-mattermost-server-configuration)
- [Phase 4: Testing and Verification](#phase-4-testing-and-verification)
- [Phase 5: Cleanup](#phase-5-cleanup)

---

## Prerequisites

### Required Tools
- Azure subscription with appropriate permissions
- Azure CLI installed and configured
- SSH client (OpenSSH, PuTTY, or similar)
- Basic Linux command line knowledge

### Required Information
- Azure subscription ID
- SSH key pair for VM authentication
- Desired Azure region (e.g., `centralus`, `eastus`)

### Install Azure CLI (if needed)

**macOS:**
```bash
brew update && brew install azure-cli
```

**Linux:**
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**Windows:**
```powershell
winget install -e --id Microsoft.AzureCLI
```

### Login to Azure
```bash
az login
az account set --subscription "YOUR_SUBSCRIPTION_ID"
```

---

## Phase 1: Azure Infrastructure Setup

### Step 1.1: Create Resource Group

```bash
az group create \
  --name GreaterLearning \
  --location centralus
```

**Verify:**
```bash
az group show --name GreaterLearning
```

### Step 1.2: Create Virtual Network

```bash
az network vnet create \
  --resource-group GreaterLearning \
  --name Project3 \
  --address-prefix 10.0.0.0/16 \
  --subnet-name PublicSubnet \
  --subnet-prefix 10.0.0.0/28
```

### Step 1.3: Create Private Subnet

```bash
az network vnet subnet create \
  --resource-group GreaterLearning \
  --vnet-name Project3 \
  --name PrivateSubnet \
  --address-prefix 10.0.0.16/28
```

**Verify Subnets:**
```bash
az network vnet subnet list \
  --resource-group GreaterLearning \
  --vnet-name Project3 \
  --output table
```

### Step 1.4: Create Network Security Group for Public VM

```bash
az network nsg create \
  --resource-group GreaterLearning \
  --name PubMattMostSvr-nsg
```

**Add Inbound Rules:**
```bash
# SSH Access
az network nsg rule create \
  --resource-group GreaterLearning \
  --nsg-name PubMattMostSvr-nsg \
  --name AllowSSH \
  --priority 100 \
  --source-address-prefixes Internet \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

# HTTP Access
az network nsg rule create \
  --resource-group GreaterLearning \
  --nsg-name PubMattMostSvr-nsg \
  --name AllowHTTP \
  --priority 110 \
  --source-address-prefixes Internet \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow

# MatterMost Access
az network nsg rule create \
  --resource-group GreaterLearning \
  --nsg-name PubMattMostSvr-nsg \
  --name AllowMatterMost \
  --priority 120 \
  --source-address-prefixes Internet \
  --destination-port-ranges 8065 \
  --protocol Tcp \
  --access Allow
```

### Step 1.5: Create Network Security Group for Private VM

```bash
az network nsg create \
  --resource-group GreaterLearning \
  --name PrivMySqlSvr-nsg
```

**Add Inbound Rules:**
```bash
# Allow MySQL from MatterMost server only
az network nsg rule create \
  --resource-group GreaterLearning \
  --nsg-name PrivMySqlSvr-nsg \
  --name AllowMySQLFromMatterMost \
  --priority 100 \
  --source-address-prefixes 10.0.0.4 \
  --destination-port-ranges 3306 \
  --protocol Tcp \
  --access Allow

# Deny all other inbound traffic from Internet
az network nsg rule create \
  --resource-group GreaterLearning \
  --nsg-name PrivMySqlSvr-nsg \
  --name DenyInternetInbound \
  --priority 110 \
  --source-address-prefixes Internet \
  --destination-address-prefixes '*' \
  --destination-port-ranges '*' \
  --protocol '*' \
  --access Deny
```

### Step 1.6: Create Public IP for MatterMost Server

```bash
az network public-ip create \
  --resource-group GreaterLearning \
  --name PubMattMostSvr-ip \
  --allocation-method Static \
  --sku Standard
```

**Get Public IP Address:**
```bash
az network public-ip show \
  --resource-group GreaterLearning \
  --name PubMattMostSvr-ip \
  --query ipAddress \
  --output tsv
```

### Step 1.7: Create Network Interface for Public VM

```bash
az network nic create \
  --resource-group GreaterLearning \
  --name PubMattMostSvr-nic \
  --vnet-name Project3 \
  --subnet PublicSubnet \
  --private-ip-address 10.0.0.4 \
  --public-ip-address PubMattMostSvr-ip \
  --network-security-group PubMattMostSvr-nsg
```

### Step 1.8: Create Network Interface for Private VM

```bash
az network nic create \
  --resource-group GreaterLearning \
  --name PrivMySqlSvr-nic \
  --vnet-name Project3 \
  --subnet PrivateSubnet \
  --private-ip-address 10.0.0.20 \
  --network-security-group PrivMySqlSvr-nsg
```

### Step 1.9: Create MySQL Database VM (Private)

```bash
az vm create \
  --resource-group GreaterLearning \
  --name PrivMySqlSvr \
  --nics PrivMySqlSvr-nic \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --no-wait
```

### Step 1.10: Create MatterMost Web VM (Public)

```bash
az vm create \
  --resource-group GreaterLearning \
  --name PubMattMostSvr \
  --nics PubMattMostSvr-nic \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --no-wait
```

**Wait for VMs to complete:**
```bash
az vm wait \
  --resource-group GreaterLearning \
  --name PrivMySqlSvr \
  --created

az vm wait \
  --resource-group GreaterLearning \
  --name PubMattMostSvr \
  --created
```

---

## Phase 2: Database Server Configuration

### Step 2.1: SSH into Private VM (via Public VM as Jump Host)

First, SSH to the public VM:
```bash
PUBLIC_IP=$(az network public-ip show \
  --resource-group GreaterLearning \
  --name PubMattMostSvr-ip \
  --query ipAddress \
  --output tsv)

ssh azureuser@$PUBLIC_IP
```

From the public VM, SSH to the private VM:
```bash
ssh azureuser@10.0.0.20
```

### Step 2.2: Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2.3: Install MySQL Server

```bash
sudo apt install mysql-server -y
```

### Step 2.4: Secure MySQL Installation

```bash
sudo mysql_secure_installation
```

Configuration recommendations:
- Set root password: **Yes** (choose strong password)
- Remove anonymous users: **Yes**
- Disallow root login remotely: **Yes**
- Remove test database: **Yes**
- Reload privilege tables: **Yes**

### Step 2.5: Configure MySQL for Remote Access

Edit MySQL configuration:
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Find and modify the `bind-address` line:
```ini
# Change from:
bind-address = 127.0.0.1

# To:
bind-address = 10.0.0.20
```

Save and exit (`Ctrl+X`, then `Y`, then `Enter`).

### Step 2.6: Create MatterMost Database and User

```bash
sudo mysql -u root -p
```

Execute the following SQL commands:
```sql
-- Create database
CREATE DATABASE mattermost CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user (replace 'secure_password' with a strong password)
CREATE USER 'mmuser'@'10.0.0.4' IDENTIFIED BY 'secure_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON mattermost.* TO 'mmuser'@'10.0.0.4';

-- Flush privileges
FLUSH PRIVILEGES;

-- Verify
SHOW DATABASES;
SELECT User, Host FROM mysql.user;

-- Exit
EXIT;
```

### Step 2.7: Restart MySQL Service

```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

### Step 2.8: Verify MySQL is Listening

```bash
sudo ss -tulpn | grep 3306
```

Expected output:
```
tcp   LISTEN 0  151  10.0.0.20:3306  0.0.0.0:*  users:(("mysqld",pid=XXXX,fd=XX))
```

Exit from the private VM:
```bash
exit
```

---

## Phase 3: MatterMost Server Configuration

### Step 3.1: Install Dependencies

On the public VM (PubMattMostSvr):
```bash
sudo apt update
sudo apt install -y wget curl mysql-client
```

### Step 3.2: Download MatterMost

```bash
cd /tmp
wget https://releases.mattermost.com/9.5.1/mattermost-9.5.1-linux-amd64.tar.gz
```

> **Note**: Check [MatterMost releases](https://github.com/mattermost/mattermost/releases) for the latest version.

### Step 3.3: Extract and Install MatterMost

```bash
tar -xvzf mattermost-*.tar.gz
sudo mv mattermost /opt
sudo mkdir /opt/mattermost/data
```

### Step 3.4: Create MatterMost User

```bash
sudo useradd --system --user-group mattermost
sudo chown -R mattermost:mattermost /opt/mattermost
sudo chmod -R g+w /opt/mattermost
```

### Step 3.5: Configure MatterMost Database Connection

```bash
sudo nano /opt/mattermost/config/config.json
```

Find the `SqlSettings` section and update:
```json
"SqlSettings": {
    "DriverName": "mysql",
    "DataSource": "mmuser:secure_password@tcp(10.0.0.20:3306)/mattermost?charset=utf8mb4,utf8&readTimeout=30s&writeTimeout=30s",
    ...
}
```

Also update the `ServiceSettings` section:
```json
"ServiceSettings": {
    "SiteURL": "http://YOUR_PUBLIC_IP:8065",
    "ListenAddress": ":8065",
    ...
}
```

Save and exit.

### Step 3.6: Test Database Connection

```bash
mysql -h 10.0.0.20 -u mmuser -p mattermost
```

Enter the password. If successful, you'll see the MySQL prompt. Exit:
```sql
EXIT;
```

### Step 3.7: Create Systemd Service

```bash
sudo nano /etc/systemd/system/mattermost.service
```

Add the following content:
```ini
[Unit]
Description=MatterMost
After=network.target
After=mysql.service
Requires=mysql.service

[Service]
Type=notify
ExecStart=/opt/mattermost/bin/mattermost
TimeoutStartSec=3600
KillMode=mixed
Restart=always
RestartSec=10
WorkingDirectory=/opt/mattermost
User=mattermost
Group=mattermost
LimitNOFILE=49152

[Install]
WantedBy=multi-user.target
```

Save and exit.

### Step 3.8: Start MatterMost Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable mattermost
sudo systemctl start mattermost
sudo systemctl status mattermost
```

### Step 3.9: Check MatterMost Logs

```bash
sudo journalctl -u mattermost -f
```

Look for:
```
Server is listening on :8065
```

Press `Ctrl+C` to exit.

---

## Phase 4: Testing and Verification

### Step 4.1: Test from Command Line

From your local machine:
```bash
PUBLIC_IP=$(az network public-ip show \
  --resource-group GreaterLearning \
  --name PubMattMostSvr-ip \
  --query ipAddress \
  --output tsv)

curl -I http://$PUBLIC_IP:8065
```

Expected response:
```
HTTP/1.1 200 OK
...
```

### Step 4.2: Access MatterMost Web Interface

Open your browser and navigate to:
```
http://YOUR_PUBLIC_IP:8065
```

You should see the MatterMost setup wizard.

### Step 4.3: Complete Initial Setup

1. **Create Admin Account**
   - Email: your-email@example.com
   - Username: admin
   - Password: (strong password)

2. **Create Team**
   - Team Name: Your Team Name
   - Team URL: your-team

3. **Invite Team Members** (optional)

### Step 4.4: Verify Database Connectivity

SSH to the public VM:
```bash
ssh azureuser@$PUBLIC_IP
```

Check MatterMost database:
```bash
mysql -h 10.0.0.20 -u mmuser -p mattermost -e "SHOW TABLES;"
```

You should see MatterMost tables created.

### Step 4.5: Verify Network Security

Test that private VM is not accessible from Internet:
```bash
# This should timeout/fail
mysql -h YOUR_PRIVATE_VM_IP -u mmuser -p
```

Test NSG rules:
```bash
# From public VM - should work
mysql -h 10.0.0.20 -u mmuser -p mattermost

# Verify firewall
sudo ufw status
```

---

## Phase 5: Cleanup

### Step 5.1: Stop Services (Optional)

```bash
ssh azureuser@$PUBLIC_IP
sudo systemctl stop mattermost
exit
```

### Step 5.2: Delete Resource Group

**⚠️ Warning**: This will delete ALL resources in the GreaterLearning resource group.

```bash
az group delete \
  --name GreaterLearning \
  --yes \
  --no-wait
```

### Step 5.3: Verify Deletion

```bash
az group show --name GreaterLearning
```

Expected: ResourceNotFound error.

### Step 5.4: Check for Orphaned Resources

```bash
# Check for any remaining public IPs
az network public-ip list --output table

# Check for any remaining NSGs
az network nsg list --output table

# Check for any remaining VNets
az network vnet list --output table
```

---

## Troubleshooting

### Issue: Cannot SSH to Private VM

**Solution**: Use public VM as jump host:
```bash
ssh -J azureuser@$PUBLIC_IP azureuser@10.0.0.20
```

### Issue: MatterMost Cannot Connect to MySQL

**Diagnosis:**
```bash
# Check MySQL is listening
sudo ss -tulpn | grep 3306

# Test connection from MatterMost server
mysql -h 10.0.0.20 -u mmuser -p mattermost

# Check NSG rules
az network nsg rule list \
  --resource-group GreaterLearning \
  --nsg-name PrivMySqlSvr-nsg \
  --output table
```

**Solution**: Verify bind-address in MySQL config and NSG rules.

### Issue: Cannot Access MatterMost from Browser

**Diagnosis:**
```bash
# Check service status
sudo systemctl status mattermost

# Check if port is listening
sudo ss -tulpn | grep 8065

# Check NSG rules
az network nsg rule list \
  --resource-group GreaterLearning \
  --nsg-name PubMattMostSvr-nsg \
  --output table
```

**Solution**: Ensure MatterMost service is running and NSG allows port 8065.

### Issue: MySQL Access Denied

**Solution**: Re-grant privileges:
```sql
GRANT ALL PRIVILEGES ON mattermost.* TO 'mmuser'@'10.0.0.4';
FLUSH PRIVILEGES;
```

---

## Additional Configuration

### Enable HTTPS (Production)

1. Install Nginx:
```bash
sudo apt install nginx certbot python3-certbot-nginx -y
```

2. Configure Nginx as reverse proxy:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8065;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
    }
}
```

3. Obtain SSL certificate:
```bash
sudo certbot --nginx -d your-domain.com
```

### Configure Automated Backups

```bash
# MySQL backup script
sudo nano /opt/scripts/backup-mysql.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/opt/backups"
DATE=$(date +%Y%m%d_%H%M%S)
mysqldump -h 10.0.0.20 -u mmuser -p'secure_password' mattermost > $BACKUP_DIR/mattermost_$DATE.sql
find $BACKUP_DIR -name "mattermost_*.sql" -mtime +7 -delete
```

Add to crontab:
```bash
0 2 * * * /opt/scripts/backup-mysql.sh
```

---

## Resources

- [MatterMost Documentation](https://docs.mattermost.com/)
- [Azure CLI Reference](https://docs.microsoft.com/cli/azure/)
- [MySQL Documentation](https://dev.mysql.com/doc/)

---

**Author**: Patrick Jones  
**Last Updated**: September 8, 2025
