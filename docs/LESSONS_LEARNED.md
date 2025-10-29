# Lessons Learned - MatterMost Azure Deployment

This document captures key lessons, challenges, and solutions encountered during the MatterMost Azure deployment project.

---

## Table of Contents
- [Executive Summary](#executive-summary)
- [Technical Challenges](#technical-challenges)
- [Key Lessons](#key-lessons)
- [Best Practices](#best-practices)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Recommendations](#recommendations)

---

## Executive Summary

The MatterMost deployment project successfully demonstrated secure multi-tier application architecture on Azure. While the initial infrastructure setup was straightforward, several networking and connectivity challenges provided valuable learning opportunities in cloud security, database configuration, and troubleshooting methodologies.

**Project Success Metrics:**
- ✅ Secure two-tier architecture implemented
- ✅ Zero database exposure to internet
- ✅ Working MatterMost application accessible via public IP
- ✅ All issues resolved through systematic troubleshooting
- ✅ Complete documentation created

---

## Technical Challenges

### Challenge 1: Network Security Group Subnet Mismatch

#### Problem Description
Initial NSG rules did not precisely match the actual subnet CIDR ranges, resulting in connectivity issues between the MatterMost server and MySQL database.

#### Symptoms
- Connection timeouts when MatterMost tried to connect to MySQL
- Inability to ping between VMs in different subnets
- NSG rules appearing correct but traffic still blocked

#### Root Cause Analysis
The NSG rules were created with generic or incorrect subnet ranges that didn't align with the actual deployed subnet configuration:
- **Expected**: 10.0.0.16/28
- **Configured**: 10.0.0.16/24 (incorrect subnet mask)

This caused Azure to reject legitimate traffic between the tiers.

#### Solution Implemented
1. Verified actual subnet ranges using Azure CLI:
```bash
az network vnet subnet show \
  --resource-group GreaterLearning \
  --vnet-name Project3 \
  --name PrivateSubnet \
  --query addressPrefix
```

2. Updated NSG rules to match exact subnet ranges:
```bash
az network nsg rule update \
  --resource-group GreaterLearning \
  --nsg-name PubMattMostSvr-nsg \
  --name AllowMySQLSubnet \
  --destination-address-prefixes 10.0.0.16/28
```

3. Verified rule changes took effect:
```bash
az network nsg rule list \
  --resource-group GreaterLearning \
  --nsg-name PubMattMostSvr-nsg \
  --output table
```

#### Lesson Learned
**Always verify infrastructure configuration before creating security rules.** Document actual deployed values (subnet ranges, IP addresses) before configuring NSGs and firewall rules.

**Tools for Verification:**
- `az network vnet subnet list` - List all subnets and their ranges
- Azure Portal network topology view
- Network Watcher connection troubleshoot feature

---

### Challenge 2: MySQL Default Security Configuration

#### Problem Description
MySQL server's default configuration binds only to `127.0.0.1` (localhost), preventing remote connections from the MatterMost application server.

#### Symptoms
- MatterMost application failed to start with database connection errors
- `mysql` client on MatterMost server could not connect
- Error message: "Can't connect to MySQL server on '10.0.0.20'"
- MySQL server was running and accessible locally on the database VM

#### Root Cause Analysis
MySQL's security-first default configuration restricts access to local connections only. The `bind-address` directive in `/etc/mysql/mysql.conf.d/mysqld.cnf` was set to `127.0.0.1`.

```ini
# Default (restrictive) configuration
bind-address = 127.0.0.1
```

This is a security feature but incompatible with multi-tier architectures where applications connect remotely.

#### Solution Implemented

**Option 1: Bind to All Interfaces (Development)**
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

# Change to:
bind-address = 0.0.0.0
```

**Option 2: Bind to Specific Private IP (Production - Recommended)**
```bash
# More secure - only listen on private network interface
bind-address = 10.0.0.20
```

**Restart MySQL:**
```bash
sudo systemctl restart mysql
```

**Verify Binding:**
```bash
sudo ss -tulpn | grep 3306
# Expected: tcp LISTEN 0 151 10.0.0.20:3306 0.0.0.0:*
```

**Test Connection:**
```bash
# From MatterMost server
mysql -h 10.0.0.20 -u mmuser -p mattermost
```

#### Additional Security Considerations
Even with MySQL listening on 0.0.0.0 or a private IP:
- Database VM has no public IP
- NSG rules restrict traffic to source IP 10.0.0.4 only
- MySQL user grants specify allowed source IP (`mmuser@10.0.0.4`)
- Private subnet has no internet gateway

This creates defense-in-depth security.

#### Lesson Learned
**Default security configurations often prevent legitimate inter-tier communication.** When deploying multi-tier applications:
1. Understand the default security posture of each service
2. Configure services for network architecture (not just localhost)
3. Maintain security through network isolation (NSGs, subnets)
4. Use specific IP bindings rather than 0.0.0.0 when possible

---

### Challenge 3: MatterMost Service Management

#### Problem Description
After successful installation and database configuration, the MatterMost web interface remained inaccessible.

#### Symptoms
- Port 8065 not responding to HTTP requests
- `curl http://130.131.225.36:8065` returned connection refused
- No obvious errors in system logs
- Database connectivity tests successful

#### Root Cause Analysis
The MatterMost binary was installed but not configured to run as a service. The application needed to be started manually and wasn't configured for auto-start on boot.

#### Solution Implemented

**Created Systemd Service Unit:**
```bash
sudo nano /etc/systemd/system/mattermost.service
```

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

**Enabled and Started Service:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable mattermost
sudo systemctl start mattermost
```

**Verified Service Status:**
```bash
sudo systemctl status mattermost
sudo journalctl -u mattermost -f
```

#### Verification Steps
```bash
# Check port is listening
sudo ss -tulpn | grep 8065

# Test locally
curl -I http://localhost:8065

# Test from local machine
curl -I http://130.131.225.36:8065
```

#### Lesson Learned
**Application installation ≠ Application running.** For production deployments:
- Always configure applications as managed services (systemd, supervisord)
- Enable auto-start on boot
- Implement restart policies for resilience
- Configure proper logging
- Monitor service health

---

### Challenge 4: Network Connectivity Diagnostics

#### Problem Description
Multiple connectivity issues required systematic troubleshooting approach to isolate and resolve.

#### Diagnostic Methodology

**Layer 1: VM Connectivity**
```bash
# Check VM is running
az vm get-instance-view \
  --resource-group GreaterLearning \
  --name PrivMySqlSvr \
  --query instanceView.statuses[1]

# Check network interface is attached
az vm nic list \
  --resource-group GreaterLearning \
  --vm-name PrivMySqlSvr
```

**Layer 2: Network Reachability**
```bash
# Test ICMP (if not blocked by NSG)
ping 10.0.0.20

# Test with specific count
ping -c 4 10.0.0.20
```

**Layer 3: Port Connectivity**
```bash
# Test TCP connectivity
nc -zv 10.0.0.20 3306

# Telnet test
telnet 10.0.0.20 3306

# With timeout
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/10.0.0.20/3306' && echo "Port is open"
```

**Layer 4: Service-Level Testing**
```bash
# MySQL client connection test
mysql -h 10.0.0.20 -u mmuser -p mattermost -e "SELECT 1"

# HTTP/HTTPS tests
curl -I http://10.0.0.4:8065
curl -v http://130.131.225.36:8065
```

**Azure Network Diagnostics**
```bash
# Check effective NSG rules
az network nic list-effective-nsg \
  --resource-group GreaterLearning \
  --name PrivMySqlSvr-nic

# Check effective routes
az network nic show-effective-route-table \
  --resource-group GreaterLearning \
  --name PrivMySqlSvr-nic
```

#### Lesson Learned
**Systematic troubleshooting is essential in cloud environments.** Build a troubleshooting toolkit:
1. Start with basic connectivity (ping, VM status)
2. Test network layer by layer (OSI model)
3. Use Azure-specific diagnostic tools
4. Document each test and result
5. Compare working vs. non-working configurations

---

## Key Lessons

### 1. Security by Design

**Principle:** Security should be built into architecture from the beginning, not added later.

**Implementation:**
- Private subnet for database tier (no internet access)
- NSG rules with explicit allow (default deny)
- Minimal port exposure
- Source IP restrictions on database access

**Takeaway:** Defense-in-depth provides multiple security layers. Even if one layer fails, others protect the system.

### 2. Network Segmentation is Critical

**Principle:** Different security zones should be in different subnets with controlled traffic flow.

**Implementation:**
- Public subnet (10.0.0.0/28) for internet-facing services
- Private subnet (10.0.0.16/28) for data tier
- NSGs controlling traffic between subnets
- No public IPs on database tier

**Takeaway:** Proper network segmentation limits attack surface and contains potential breaches.

### 3. Documentation is Essential

**Principle:** Document architecture decisions, configurations, and troubleshooting steps.

**Benefits:**
- Faster issue resolution
- Knowledge transfer to team members
- Audit trail for security compliance
- Easier replication of environment

**Takeaway:** Time spent documenting is time saved during troubleshooting and maintenance.

### 4. Testing at Each Layer

**Principle:** Verify functionality at each layer before proceeding to the next.

**Testing Progression:**
1. Infrastructure (VMs, VNet, Subnets)
2. Network connectivity (ping, traceroute)
3. Port accessibility (netcat, telnet)
4. Service functionality (MySQL client, HTTP requests)
5. Application integration (MatterMost to MySQL)
6. End-to-end user experience (web browser access)

**Takeaway:** Incremental testing identifies issues early and narrows problem scope.

### 5. Azure NSG Rule Priority Matters

**Principle:** NSG rules are processed in priority order (100-4096).

**Key Points:**
- Lower numbers = higher priority
- First matching rule is applied
- Default rules have priority 65000+
- Inbound and outbound rules are separate

**Best Practice:**
- Use priority ranges by function (100-199 for critical, 200-299 for apps)
- Leave gaps between priorities for future additions
- Document the purpose of each rule

### 6. MySQL User Grants are IP-Specific

**Principle:** MySQL user permissions include the source host/IP.

**Example:**
```sql
-- This user can ONLY connect from 10.0.0.4
CREATE USER 'mmuser'@'10.0.0.4' IDENTIFIED BY 'password';

-- This is a DIFFERENT user (connects from anywhere)
CREATE USER 'mmuser'@'%' IDENTIFIED BY 'password';
```

**Takeaway:** MySQL authentication combines username AND source address. Both must match for access.

### 7. Cloud Costs Add Up Quickly

**Principle:** Running resources incur continuous charges.

**Cost Factors:**
- VM compute time (hourly charges)
- Public IP addresses (static IPs cost more)
- Storage (OS disks, data disks)
- Data egress (outbound internet traffic)

**Cost Management:**
- Use B-series burstable VMs for development
- Deallocate VMs when not in use
- Delete resources promptly after testing
- Use Azure Cost Management for monitoring

**Cleanup Command:**
```bash
az group delete --name GreaterLearning --yes --no-wait
```

---

## Best Practices

### Azure Resource Organization

1. **Use Resource Groups Effectively**
   - One resource group per project/environment
   - Consistent naming conventions
   - Tagging for cost allocation

2. **Naming Conventions**
   ```
   Format: <service>-<environment>-<region>-<instance>
   Example: vm-prod-eastus-01
   ```

3. **Use Azure Bastion for Production**
   - Eliminates need for public IPs on VMs
   - Provides secure RDP/SSH access
   - Reduces attack surface

### Network Security

1. **Default Deny, Explicit Allow**
   - Start with restrictive rules
   - Add specific allow rules as needed
   - Document reason for each rule

2. **Use Service Tags**
   ```bash
   # Allow access from Azure Load Balancer
   --source-address-prefixes AzureLoadBalancer
   ```

3. **Regular Security Audits**
   ```bash
   # Review NSG rules quarterly
   az network nsg list --output table
   ```

### Database Security

1. **Never Expose Databases Directly to Internet**
   - Use private networking
   - Implement VPN or ExpressRoute for access
   - Consider Azure Private Link

2. **Use Strong Authentication**
   - Complex passwords (Azure Key Vault)
   - Certificate-based authentication
   - Azure AD authentication (when supported)

3. **Regular Backups**
   - Automated daily backups
   - Test restore procedures
   - Off-site backup storage

### Application Deployment

1. **Use Configuration Management**
   - Ansible, Terraform, or ARM templates
   - Version control for infrastructure code
   - Reproducible deployments

2. **Implement Monitoring**
   - Azure Monitor
   - Application Insights
   - Log Analytics
   - Custom alerts

3. **High Availability Considerations**
   - Availability Zones
   - Load balancers
   - Auto-scaling
   - Disaster recovery plan

---

## Troubleshooting Guide

### Quick Diagnostic Commands

```bash
# Check VM status
az vm get-instance-view --resource-group RG --name VM --query instanceView.statuses

# Check NSG rules
az network nsg rule list --resource-group RG --nsg-name NSG --output table

# Test connectivity
nc -zv <ip> <port>

# Check service status
sudo systemctl status <service>

# View recent logs
sudo journalctl -u <service> -n 50 --no-pager

# Check open ports
sudo ss -tulpn
```

### Common Issues and Solutions

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Cannot connect to VM | NSG blocking traffic | Check NSG rules for source/dest/port |
| Service not responding | Service not running | `sudo systemctl start <service>` |
| Database connection failed | MySQL bind-address | Update mysqld.cnf bind-address |
| Slow performance | Undersized VM | Resize VM to larger SKU |
| Cost overrun | Resources not deallocated | Delete or stop unused resources |

---

## Recommendations

### For Future Projects

1. **Use Infrastructure as Code (IaC)**
   - Terraform or ARM templates
   - Version control for all infrastructure
   - Automated testing of deployments

2. **Implement CI/CD**
   - Azure DevOps or GitHub Actions
   - Automated testing
   - Blue/green deployments

3. **Enhanced Monitoring**
   - Application Performance Monitoring (APM)
   - Custom dashboards
   - Proactive alerting

4. **Security Enhancements**
   - Azure Key Vault for secrets
   - Managed identities
   - Azure Security Center recommendations

5. **High Availability**
   - Multi-region deployment
   - Azure Traffic Manager
   - Database replication

### For Production Deployment

**Do:**
- Use managed services where possible (Azure Database for MySQL)
- Implement backup and disaster recovery
- Use HTTPS with valid certificates
- Enable Azure DDoS Protection
- Implement proper logging and monitoring
- Use Azure Policy for governance
- Regular security audits

**Don't:**
- Expose databases to public internet
- Use default passwords
- Skip backup testing
- Ignore security updates
- Deploy without monitoring
- Forget about cost management

---

## Conclusion

This MatterMost deployment project provided valuable hands-on experience with:
- Azure infrastructure design and implementation
- Network security best practices
- Multi-tier application architecture
- Troubleshooting methodologies
- Documentation and knowledge sharing

The challenges encountered were typical of real-world cloud deployments and provided excellent learning opportunities in problem-solving and system administration.

**Key Takeaway:** Success in cloud infrastructure requires a combination of technical knowledge, systematic troubleshooting, and proper documentation.

---

**Author**: Patrick Jones  
**Project Date**: September 8, 2025  
**Last Updated**: September 8, 2025
