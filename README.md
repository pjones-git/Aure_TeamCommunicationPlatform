# Azure MatterMost Deployment Project

[![Azure](https://img.shields.io/badge/Azure-Cloud-0078D4?logo=microsoft-azure)](https://azure.microsoft.com/)
[![MatterMost](https://img.shields.io/badge/MatterMost-Team_Chat-0058CC?logo=mattermost)](https://mattermost.com/)
[![MySQL](https://img.shields.io/badge/MySQL-Database-4479A1?logo=mysql)](https://www.mysql.com/)
[![Linux](https://img.shields.io/badge/Linux-Ubuntu_24.04-E95420?logo=ubuntu)](https://ubuntu.com/)

## 📋 Project Overview

This project demonstrates the deployment of **MatterMost**, an open-source team collaboration platform, on **Microsoft Azure** using a secure, multi-tier architecture. The implementation showcases cloud infrastructure design, network security best practices, and Linux system administration.

### 🎯 Project Goals

- Deploy a production-ready team communication platform on Azure
- Implement secure network architecture with public and private subnets
- Configure Network Security Groups (NSGs) for granular traffic control
- Establish secure communication between web and database tiers
- Document lessons learned and troubleshooting processes

---

## 🏗️ Architecture

### Network Topology

```
Internet → Azure Load Balancer → Public Subnet (MatterMost) → Private Subnet (MySQL)
```

### Architecture Diagram

<img width="522" height="410" alt="Screenshot 2025-10-29 at 8 40 38 PM" src="https://github.com/user-attachments/assets/49d68f24-e19e-4932-9462-4cff27330e0e" />


<img width="706" height="576" alt="Screenshot 2025-10-29 at 6 52 55 PM" src="https://github.com/user-attachments/assets/7ea6069b-1b96-4129-b96a-f3f2e538036e" />


### Infrastructure Components

| Component | Details |
|-----------|---------|
| **Resource Group** | GreaterLearning |
| **Virtual Network** | Project3 (10.0.0.0/16) |
| **Public Subnet** | 10.0.0.0/28 |
| **Private Subnet** | 10.0.0.16/28 |
| **Public VM** | PubMattMostSvr (Ubuntu 24.04, 10.0.0.4) |
| **Private VM** | PrivMySqlSvr (Ubuntu 24.04, 10.0.0.20) |
| **Public IP** | 130.131.225.36:8065 |

---

## 🛠️ Technical Skills Demonstrated

### Cloud Infrastructure
- ✅ Azure Resource Group management
- ✅ Virtual Network (VNet) design and implementation
- ✅ Subnet segmentation (public/private architecture)
- ✅ Virtual Machine provisioning and configuration
- ✅ Public IP allocation and management

### Network Security
- ✅ Network Security Group (NSG) configuration
- ✅ Inbound/outbound rule management
- ✅ Port-based access control (22, 80, 3306, 8065)
- ✅ Source IP filtering and network isolation
- ✅ Zero-trust network principles

### Linux System Administration
- ✅ Ubuntu Server 24.04 configuration
- ✅ MySQL database installation and hardening
- ✅ MatterMost application deployment
- ✅ Service management (systemd)
- ✅ Network troubleshooting (ping, netstat, ss)

### Database Management
- ✅ MySQL server installation
- ✅ Database creation and user management
- ✅ Network binding configuration
- ✅ Security hardening (localhost-only access)
- ✅ Remote database connectivity setup

### DevOps Practices
- ✅ Infrastructure documentation
- ✅ Security best practices implementation
- ✅ Troubleshooting and problem resolution
- ✅ Architecture diagram creation
- ✅ Technical documentation writing

---

## 🚀 Key Features

### Security Features
- **Network Isolation**: Database server completely isolated in private subnet with no internet access
- **Granular NSG Rules**: Separate Network Security Groups for each VM with specific port rules
- **Minimal Attack Surface**: MySQL only accessible from MatterMost server (10.0.0.4)
- **SSH Access Control**: Restricted SSH access through NSG rules

### High Availability Design
- Separate compute tiers for web and database
- Independent VM scaling capability
- Isolated failure domains

### Production-Ready Configuration
- MatterMost accessible on standard port 8065
- HTTP/HTTPS support (ports 80/443)
- Database connection pooling
- Persistent data storage

---

## 📊 Network Security Group Rules

### PubMattMostSvr-nsg (Public Subnet)

| Priority | Direction | Port | Source | Destination | Action |
|----------|-----------|------|--------|-------------|--------|
| 100 | Inbound | 22 | Internet | 10.0.0.4 | Allow |
| 110 | Inbound | 80 | Internet | 10.0.0.4 | Allow |
| 120 | Inbound | 8065 | Internet | 10.0.0.4 | Allow |
| 100 | Outbound | 3306 | 10.0.0.4 | 10.0.0.16/28 | Allow |

### PrivMySqlSvr-nsg (Private Subnet)

| Priority | Direction | Port | Source | Destination | Action |
|----------|-----------|------|--------|-------------|--------|
| 100 | Inbound | 3306 | 10.0.0.4 | 10.0.0.20 | Allow |
| 110 | Inbound | * | Internet | * | Deny |
| 100 | Outbound | * | * | Internet | Deny |

---

## 🔧 Setup and Configuration

### Prerequisites
- Active Azure subscription
- Azure CLI or Azure Portal access
- Basic understanding of Linux administration
- SSH client (OpenSSH, PuTTY, etc.)

### Deployment Steps

1. **Create Resource Group**
   ```bash
   az group create --name GreaterLearning --location centralus
   ```

2. **Create Virtual Network and Subnets**
   ```bash
   az network vnet create \
     --resource-group GreaterLearning \
     --name Project3 \
     --address-prefix 10.0.0.0/16 \
     --subnet-name PublicSubnet \
     --subnet-prefix 10.0.0.0/28
   
   az network vnet subnet create \
     --resource-group GreaterLearning \
     --vnet-name Project3 \
     --name PrivateSubnet \
     --address-prefix 10.0.0.16/28
   ```

3. **Deploy Virtual Machines**
   - See [SETUP.md](docs/SETUP.md) for detailed VM configuration steps

4. **Configure MySQL Database**
   ```bash
   sudo apt update
   sudo apt install mysql-server -y
   sudo mysql_secure_installation
   ```

5. **Install MatterMost**
   ```bash
   wget https://releases.mattermost.com/X.X.X/mattermost-X.X.X-linux-amd64.tar.gz
   tar -xvzf mattermost*.gz
   sudo mv mattermost /opt
   ```

For complete setup instructions, see [SETUP.md](docs/SETUP.md).

---

## 🐛 Troubleshooting & Lessons Learned

### Key Challenges Resolved

#### 1. NSG Rule Subnet Mismatch
**Problem**: Initial NSG rules didn't match actual subnet ranges, causing connectivity issues.

**Solution**: Updated NSG rules to precisely match the subnet CIDR blocks (10.0.0.0/28 and 10.0.0.16/28).

#### 2. MySQL Connectivity Issues
**Problem**: MySQL default configuration binds to localhost only, preventing connections from MatterMost server.

**Solution**: Modified `/etc/mysql/mysql.conf.d/mysqld.cnf` to bind to 0.0.0.0 or specific private IP (10.0.0.20), then restarted MySQL service.

```bash
sudo sed -i 's/bind-address.*/bind-address = 10.0.0.20/' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl restart mysql
```

#### 3. MatterMost Service Not Running
**Problem**: MatterMost web interface not accessible after installation.

**Solution**: Started MatterMost service and configured it to run as a systemd service for automatic startup.

```bash
sudo systemctl start mattermost
sudo systemctl enable mattermost
```

#### 4. Network Connectivity Testing
**Problem**: Needed to verify network connectivity between tiers.

**Solution**: Used ping, telnet, and netcat to test connectivity:
```bash
# Test network reachability
ping 10.0.0.20

# Test MySQL port
nc -zv 10.0.0.20 3306
```

For more detailed troubleshooting information, see [LESSONS_LEARNED.md](docs/LESSONS_LEARNED.md).

---

## 📸 Screenshots

### MatterMost Interface
The deployed MatterMost instance showing team channels and messaging functionality.

### Azure Resource Group
All resources organized under the GreaterLearning resource group for easy management.

---

## 🎓 Skills & Technologies

### Cloud Platforms
- Microsoft Azure (IaaS)

### Networking
- Virtual Networks (VNet)
- Subnets & CIDR notation
- Network Security Groups (NSG)
- Public/Private IP addressing
- Network isolation & segmentation

### Operating Systems
- Ubuntu Linux 24.04 LTS
- SSH remote administration
- systemd service management

### Databases
- MySQL 8.0+
- Database security & hardening
- User & permission management

### Applications
- MatterMost (Open-source collaboration platform)
- Web server configuration
- Application deployment

### Security
- Network security best practices
- Defense in depth
- Principle of least privilege
- Firewall rule configuration

---

## 📈 Project Outcomes

- ✅ Successfully deployed secure, multi-tier application on Azure
- ✅ Implemented production-ready network security architecture
- ✅ Gained hands-on experience with Azure networking and compute services
- ✅ Developed troubleshooting skills for cloud infrastructure
- ✅ Created comprehensive technical documentation

---

## 🗑️ Project Cleanup

After project completion, all resources were properly decommissioned to avoid unnecessary costs:

```bash
az group delete --name GreaterLearning --yes --no-wait
```

This removed:
- Both Virtual Machines (PubMattMostSvr, PrivMySqlSvr)
- Virtual Network and subnets
- Network Security Groups
- Public IP addresses
- Network interfaces
- Storage accounts

---

## 📚 Additional Resources

- [MatterMost Official Documentation](https://docs.mattermost.com/)
- [Azure Virtual Networks Documentation](https://docs.microsoft.com/azure/virtual-network/)
- [Azure Network Security Groups](https://docs.microsoft.com/azure/virtual-network/network-security-groups-overview)
- [MySQL Documentation](https://dev.mysql.com/doc/)

---

## 👤 Author

**Patrick Jones**

- GitHub: [@pjones-git](https://github.com/pjones-git)
- Project Date: September 8, 2025

---

## 📄 License

This project is for educational and portfolio purposes.

---

## 🙏 Acknowledgments

- Greater Learning Cloud Computing Program
- MatterMost open-source community
- Microsoft Azure documentation team

---

**Note to Recruiters**: This project demonstrates practical cloud architecture skills, security-first design thinking, and the ability to troubleshoot complex multi-tier applications. The documentation reflects professional technical writing capabilities suitable for DevOps and Cloud Engineering roles.
