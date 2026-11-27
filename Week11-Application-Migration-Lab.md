# Week 11 – Application Migration Lab: Tailwind Traders 3-Tier Application

**Course:** CST8913 – Cloud Migration
**Scenario:** Tailwind Traders is migrating a 3-tier on-premises application to Azure. The application includes WEB01 and WEB02 (frontend, VMware), APP01 (API/application server, VMware), SQL01 (SQL Server 2017, physical server), and LB01 (internal load balancer). It also has nightly backups, firewall rules, and a strict 1-hour downtime window.

---

## Task 1 – Set Up Tooling for Discovery

### Agent-based, agentless, or a mix?
**A mix, mostly agentless.**

- **VMware servers (WEB01, WEB02, APP01):** use **agentless** discovery through the Azure Migrate appliance connected to vCenter Server. This covers configuration, performance data, software inventory, SQL discovery, and agentless dependency analysis, and nothing has to be installed on the production VMs.
- **Physical server (SQL01):** SQL01 isn't on vCenter, so it is discovered by a second Azure Migrate appliance for **physical servers**, which connects to the server over WinRM. If agentless dependency analysis isn't available for physical servers in our environment, we install the **dependency agent** on SQL01 only (agent-based).

Agentless keeps the impact on production small. Agents are used only where the agentless option can't reach.

### How many appliances, and why?
**Two appliances:**

| Appliance | Scope | Reason |
|---|---|---|
| Appliance 1 – VMware | WEB01, WEB02, APP01 (via vCenter) | A VMware appliance discovers through the vCenter APIs |
| Appliance 2 – Physical | SQL01 | Physical servers need the physical-server appliance type, and one appliance can't be registered for both VMware and physical discovery |

Both appliances register to the same Azure Migrate project, so all servers show up in one inventory and can be included in the same assessment and dependency views.

### Credentials to prepare

| Purpose | Credentials needed |
|---|---|
| **vCenter discovery** | Read-only vCenter account, scoped to the Tailwind VMs where possible |
| **Software inventory** | Guest OS credentials: a domain or local Windows account with administrator rights (or a Linux account with the required permissions) |
| **SQL discovery** | Windows authentication (a domain account with access to SQL01) or a SQL login with the permissions Azure Migrate needs. Use the least-privileged account script from the Azure Migrate documentation rather than sysadmin |
| **Dependency mapping** | Windows guest credentials with administrator rights on the VMware VMs (agentless). For SQL01: WinRM access (ports 5985/5986) plus the account used to install the dependency agent if the agent-based option is used |

### Three best practices for discovery
1. **Use dedicated, least-privileged accounts.** Create service accounts just for Azure Migrate, give them only the required rights, store them securely, and disable them after the migration.
2. **Collect enough performance history.** Let discovery run for **at least 30 days** so the data covers normal business cycles, peaks, and the nightly backup jobs. Sizing on a few hours of data under-sizes servers.
3. **Prepare the network and keep the appliances healthy.** Open the required ports (443 outbound to Azure, 5985/5986 WinRM, 1433 for SQL, vCenter 443), allow the required Azure URLs through the firewall/proxy, turn on auto-update, and check appliance health in the portal regularly.

---

## Task 2 – Perform Assessment Planning

| Setting | Choice | Justification |
|---|---|---|
| **Assessment type** | **Production** (Azure VM assessment, plus an Azure SQL assessment for SQL01) | This is a business application in live use, so it needs production pricing, SLAs, and the production-grade VM and disk recommendations. Dev/Test pricing doesn't apply. |
| **Target region** | **Canada Central** (Toronto), with **Canada East** as the paired DR region | Lowest latency for Canadian users and offices, keeps data in Canada, and full service availability (Availability Zones, SQL MI). |
| **Performance history** | **1 month**, **95th percentile** utilization | Covers a full business cycle including month-end. The 95th percentile ignores rare spikes without sizing for the average. |
| **Sizing criteria** | **Performance-based** | See below. |
| **Comfort factor** | **1.3×** | Adds 30% headroom for growth, seasonal peaks, and uncertainty in the collected data. |
| **Pricing model** | **Pay-as-you-go** during migration and stabilization, then **3-year Reserved Instances** (or an Azure savings plan) once the sizes are confirmed | The servers run 24/7 with steady load, so reservations save a lot. Waiting until after stabilization avoids reserving the wrong size. |
| **Licensing** | **Azure Hybrid Benefit** for Windows Server and SQL Server (if Tailwind has Software Assurance) | Reuses existing licenses and greatly reduces compute and SQL cost. |

### Performance-based or as-on-premises sizing?
**Performance-based sizing.** On-premises servers are usually over-provisioned because hardware is bought for peak load and future growth. As-on-premises sizing copies that waste into Azure, where every unused vCPU and GB of RAM is billed monthly. Performance-based sizing uses actual CPU, memory, disk IOPS/throughput, and network usage, then adds the comfort factor for safety. The result is right-sized VMs and disks at a lower cost. Once the application runs in Azure, the sizes can be scaled up if needed.

---

## Task 3 – Dependency Analysis

### Application components

| Component | Role | Current platform |
|---|---|---|
| LB01 | Internal load balancer in front of the web tier | On-premises appliance |
| WEB01, WEB02 | Frontend web servers | VMware |
| APP01 | API/application server | VMware |
| SQL01 | SQL Server 2017 database | Physical server |
| Supporting services | AD domain controllers, DNS, backup server, firewall, monitoring | On-premises infrastructure |

### Dependencies (8+)

| # | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|
| 1 | Users/clients | LB01 | TCP 443 (HTTPS) | Application access |
| 2 | LB01 | WEB01, WEB02 | TCP 443 / 80 | Load-balanced web traffic and health probes |
| 3 | WEB01, WEB02 | APP01 | TCP 443 (HTTPS/REST API) | Frontend calls the API tier |
| 4 | APP01 | SQL01 | TCP 1433 (TDS) | Application database queries |
| 5 | All servers | Domain controllers | TCP/UDP 88 (Kerberos), 389/636 (LDAP/LDAPS), 445 (SMB) | Authentication, Group Policy, service accounts |
| 6 | All servers | DNS servers | TCP/UDP 53 | Name resolution (including connection strings that use hostnames) |
| 7 | SQL01 | Backup server / storage | TCP 445 (SMB) or backup agent port | Nightly database backups |
| 8 | APP01 | SMTP relay | TCP 25 / 587 | Application email notifications |
| 9 | APP01 | External third-party API (if used) | TCP 443 outbound | External integrations, through the firewall |
| 10 | All servers | WSUS / patch server | TCP 8530 / 8531 | Patching |

### Items to filter out as noise
These show up in dependency maps but **aren't application dependencies**, so they don't decide how servers are grouped:
- Antivirus/EDR update and telemetry traffic
- Monitoring and management agents (SCOM, SCCM, the Azure Migrate appliance itself)
- Windows Update / Microsoft telemetry endpoints
- Admin RDP sessions (TCP 3389) and SSH from administrator workstations
- ICMP (ping) and NTP (UDP 123) time synchronization
- Failed or one-off connections (port scans, decommissioned servers)

They still need to work in Azure, but they are covered by shared landing-zone services (Defender, Azure Monitor, Update Manager, Bastion) rather than by migration grouping.

### Business requirements (from the application owner)

| Requirement | Details |
|---|---|
| **Business criticality** | **High / tier-1.** Customer- or revenue-facing application, so an outage directly affects the business. |
| **Uptime & downtime** | 99.9%+ availability target. **Maximum 1-hour downtime window** for cutover, scheduled outside business hours (e.g., Saturday 01:00–02:00). |
| **Data classification** | **Confidential.** The database holds customer and business data, so it needs encryption at rest and in transit, private endpoints, and data residency in Canada. |
| **Licensing dependencies** | SQL Server 2017 licenses (Software Assurance needed for Azure Hybrid Benefit), Windows Server licenses, and any third-party software tied to hardware IDs or MAC addresses that must be re-issued. |
| **Patching requirements** | Monthly OS patching. SQL Server 2017 needs current CUs and leaves extended support in October 2027, which is a reason to consider a PaaS target. |
| **Firewall & IP considerations** | Any hard-coded IPs in configs must be found and replaced with DNS names. Firewall rules move to NSGs and Azure Firewall. Third-party partners that allow-list Tailwind's public IP must be told about the new Azure public IP. LB01's VIP is replaced by the Azure load balancer IP. |

---

## Task 4 – Validate Assessment Results with the Application Owner

### Recommended VM sizes

| Tier | Server(s) | Recommended size | Reason |
|---|---|---|---|
| WEB | WEB01, WEB02 | **Standard_D2s_v5** (2 vCPU, 8 GB RAM) ×2, Premium SSD | General purpose, light stateless workload. Two instances across Availability Zones for HA. |
| APP | APP01 | **Standard_D4s_v5** (4 vCPU, 16 GB RAM), Premium SSD | Handles API/business logic with more CPU and memory headroom. |
| SQL | SQL01 (if rehosted to IaaS) | **Standard_E8ds_v5** (8 vCPU, 64 GB RAM) with Premium SSD v2 / P30 disks for data and logs | Memory-optimized for the SQL buffer pool. Separate data and log disks for IOPS. |

*(Final sizes come from the performance data plus the 1.3 comfort factor and are confirmed with the application owner.)*

### Components to replace or optimize
- **LB01:** replace with **Azure Application Gateway (with WAF)** for HTTPS traffic to the web tier, or an **internal Azure Standard Load Balancer** if only layer-4 balancing is needed. The hardware appliance doesn't migrate.
- **WEB tier:** deploy across **Availability Zones**. Later, consider a VM Scale Set or Azure App Service.
- **APP01:** a single point of failure today. Consider adding a second instance behind an internal load balancer.
- **SQL01:** move to **Azure SQL Managed Instance** (recommended, see below) instead of a physical-to-VM rehost. This removes the single physical server and adds built-in HA.
- **Backups:** replace the nightly backup job with **Azure Backup** (VMs) and **automated SQL MI backups** (point-in-time restore).
- **Firewall:** replace on-premises rules with **NSGs** and **Azure Firewall**.

### Dependency validation
Every dependency in Task 3 has been mapped to an Azure equivalent:
WEB→APP (443), APP→SQL (1433 → SQL MI private endpoint), AD/DNS (domain controllers in Azure or reached over VPN/ExpressRoute), backups (Azure Backup), SMTP (relay over VPN or a cloud email service), patching (Azure Update Manager). **No dependencies are left unmapped.** The servers have been reviewed with the application owner to confirm none were missed (e.g., scheduled tasks or file shares).

### SLA and downtime check
- **SLA:** zone-redundant web VMs (99.99% VM SLA across zones) plus SQL MI (99.99% Business Critical tier) **meets or exceeds** the 99.9% target.
- **Downtime:** web and app VMs are replicated ahead of time with Azure Migrate (replication runs while the source stays live), so their cutover takes minutes. The database uses an **online** migration (continuous sync), so the final cutover is about 15–30 minutes. **The 1-hour window can be met.**

### SQL migration options

| Option | Description | Pros | Cons | Fit |
|---|---|---|---|---|
| **Rehost (SQL Server on Azure VM)** | Lift-and-shift SQL Server 2017 onto an Azure VM | Full compatibility, OS-level control, fastest move | Still need to patch, back up, and manage HA ourselves. SQL 2017 support ends soon. | Good fallback |
| **Azure SQL Managed Instance** | PaaS with near-100% SQL Server compatibility | Supports SQL Agent jobs, cross-database queries, CLR, and linked servers. Automated patching, backups, and HA. Online migration via Azure DMS / Log Replay Service / MI link. | Higher cost than SQL DB, VNet deployment needed | **Recommended** |
| **Azure SQL Database** | Fully managed single database | Lowest management overhead, serverless/elastic options | No SQL Agent, limited cross-database queries, may need app changes | Not suitable without refactoring |

**Recommendation:** **Azure SQL Managed Instance (Business Critical)**. It has the compatibility of a rehost without patching or backup work, removes the physical server, solves the SQL 2017 end-of-support problem, and supports online migration to fit the 1-hour window.

---

## Task 5 – Create the Migration Plan (Runbook)

### Pre-migration tasks
1. Build the Azure landing zone: VNet with subnets (web, app, data/MI, gateway), NSGs, Azure Firewall, and site-to-site VPN or ExpressRoute to on-premises.
2. Make sure domain controllers and DNS are reachable from Azure (deploy a DC in Azure or extend over VPN).
3. Deploy Azure SQL Managed Instance and Application Gateway/Load Balancer ahead of time.
4. Start Azure Migrate replication for WEB01, WEB02, and APP01. Set up the database migration (DMS online / LRS / MI link) for SQL01.
5. Lower **DNS TTL** on application records to 5 minutes at least 48 hours before cutover.
6. Run a **test migration** in an isolated VNet and have the application owner complete user acceptance testing (UAT).
7. Take full backups of all servers and the database. Document the current configuration and connection strings.
8. Announce the change window to stakeholders, get CAB approval, and freeze application changes.

### Migration steps (per server group)

**Wave 1 – WEB01, WEB02**
1. Confirm replication is healthy and the last sync is recent.
2. Stop IIS/web services on-premises (or drain them from LB01).
3. Run **Migrate** in Azure Migrate. VMs are created in the web subnet across Availability Zones.
4. Add the VMs to the Application Gateway backend pool and confirm health probes pass.
5. Point the web tier to APP01 (on-premises over VPN until Wave 2).

**Wave 2 – APP01**
1. Stop application services on APP01 and run the final replication sync.
2. Migrate APP01 to the app subnet.
3. Update the WEB config so the API endpoint points to the new APP01 address (by DNS name).
4. APP01 temporarily connects to SQL01 on-premises over VPN until Wave 3.

**Wave 3 – SQL01**
1. Confirm the online sync to SQL MI is caught up (minimal lag).
2. Put the application in maintenance mode and stop writes to SQL01.
3. Complete the final log sync and **cut over** to SQL MI.
4. Check row counts, logins, SQL Agent jobs, and linked servers.
5. Update APP01 connection strings to the SQL MI endpoint and bring the application back online.

### DNS updates
- Update A/CNAME records for the application URL to the **Application Gateway / Azure LB IP**.
- Update internal records (`app01`, `sql01` or aliases) to the new Azure private IPs or the SQL MI FQDN.
- Use **Azure Private DNS zones** for private endpoints. Set up conditional forwarders between on-premises DNS and Azure DNS.
- Restore DNS TTL to normal after the migration is stable.

### Connection string changes
- **WEB → APP:** API base URL in `web.config` / `appsettings.json` → new APP01 hostname.
- **APP → SQL:** `Server=sql01;...` → `Server=<mi-name>.<dns-zone>.database.windows.net;...` with `Encrypt=True`.
- Move secrets into **Azure Key Vault** and use managed identities where possible.
- Search configs, scheduled tasks, and scripts for **hard-coded IPs** and replace them.

### Load balancer considerations
- LB01 is **replaced**, not migrated. Recreate its configuration in Application Gateway / Azure LB: listeners, backend pools, health probes, session persistence (if the app needs sticky sessions), and SSL certificates.
- Upload the TLS certificate to Application Gateway (or Key Vault).
- Deploy the load balancer and web VMs **zone-redundant**.
- Test failover by stopping one web VM and confirming traffic moves to the other.

### SQL migration considerations
- Run the **Data Migration Assistant / Azure SQL assessment** to find compatibility issues before migration.
- Migrate **logins, SQL Agent jobs, linked servers, and credentials**. These aren't stored in the user database.
- Set the database compatibility level properly (keep 140 initially for SQL 2017 behaviour, raise it later).
- Use an **online** migration method so the cutover fits inside the 1-hour window.
- Turn on TDE (on by default), auditing, and a private endpoint. Confirm the backup retention policy.

### Post-migration validation checklist
- [ ] Application URL loads through Application Gateway (HTTPS, valid certificate)
- [ ] Both web servers healthy in the backend pool
- [ ] WEB → APP API calls succeed
- [ ] APP → SQL MI connections succeed with no errors in application logs
- [ ] Data integrity: row counts and checksums match the source
- [ ] User logins and AD authentication work
- [ ] SQL Agent jobs and scheduled tasks run
- [ ] Email/SMTP and external API integrations work
- [ ] Azure Backup and SQL MI backups configured and the first backup succeeded
- [ ] Azure Monitor alerts and Log Analytics set up
- [ ] Performance close to or better than the on-premises baseline
- [ ] Application owner signs off on UAT

### Back-out plan (if the migration fails)
- **Decision point:** if validation isn't passed within **40 minutes** of starting cutover, the migration lead calls a rollback so it finishes inside the 1-hour window.
- **Web/App:** source VMs on VMware are **powered off, not deleted**. Power them back on, revert DNS to the LB01 VIP, and return traffic to LB01.
- **SQL:** SQL01 stays intact in read-only/offline state. Bring it back online, point APP01's connection string back to SQL01, and reconcile any transactions written to SQL MI during the window. A short cutover with maintenance mode keeps this minimal.
- Keep the on-premises environment available for **at least 2 weeks** after a successful migration before decommissioning.
- Document the failure cause, fix it, and reschedule the wave.

---

## Task 6 – Plan the Migration Waves

### Justification

- **Dependencies:** the WEB tier depends on APP01, and APP01 depends on SQL01. Moving from the top of the stack down means each wave's dependency is still reachable on-premises over the VPN/ExpressRoute link. Latency between tiers during the hybrid period must be tested in the test migration.
- **Business criticality:** SQL01 holds all the data and has the most business risk, so it goes last, after the team has gained experience and confidence from the earlier waves.
- **Downtime restrictions:** each wave has its own 1-hour window. The web servers can be moved one at a time with almost no downtime because they sit behind a load balancer. The database needs the most careful cutover (online sync).
- **Testing requirements:** web servers are stateless and easy to test and roll back. APP01 needs API and integration testing. SQL01 needs data integrity checks, performance testing, and full UAT.

### Wave table

| Wave | Servers | Reason |
|---|---|---|
| **Wave 1** | WEB01, WEB02 (+ LB01 replaced by Application Gateway) | Stateless, low risk, easy rollback. Can be migrated one at a time behind the load balancer, which proves out the landing zone and network. |
| **Wave 2** | APP01 | Depends on the WEB tier being stable. Moderate risk. Needs API and integration testing. Still connects to SQL01 on-premises over the hybrid link. |
| **Wave 3** | SQL01 → Azure SQL Managed Instance | Highest risk and most critical (holds all the data). Needs online sync, data validation, and full UAT. Goes last so lessons from Waves 1–2 apply. |

---

## Summary
Tailwind Traders' 3-tier application will be discovered with two Azure Migrate appliances (VMware and physical), assessed with production, performance-based settings in Canada Central, and migrated in three waves from the web tier down to the database. LB01 is replaced with Azure Application Gateway, and SQL01 moves to Azure SQL Managed Instance with an online migration so each cutover stays within the 1-hour downtime window. A tested back-out plan keeps the on-premises servers available until the migration is confirmed stable.
