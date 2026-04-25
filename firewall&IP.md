# 🔥 SQL Server Firewall Rules — Complete Guide

> A comprehensive reference for understanding and configuring firewall rules during SQL Server / Azure SQL Server creation, with practical examples.

---

## 📚 Table of Contents

- [What is a Firewall Rule?](#what-is-a-firewall-rule)
- [Why Firewall Rules Matter](#why-firewall-rules-matter)
- [Types of Firewall Rules](#types-of-firewall-rules)
- [Firewall Rule Levels](#firewall-rule-levels)
- [Setting Up During Server Creation](#setting-up-during-server-creation)
- [Practical Examples](#practical-examples)
- [How to Manage Firewall Rules](#how-to-manage-firewall-rules)
- [Common Connection Errors](#common-connection-errors)
- [Best Practices](#best-practices)
- [FAQ](#faq)

---

## What is a Firewall Rule?

A **Firewall Rule** is a security setting that controls **who can connect** to your SQL Server by allowing or blocking IP addresses.

Think of it as a **bouncer at the door** — only IPs on the guest list are allowed in.

```
Your App / Client
        │
        ▼
  ┌─────────────┐
  │  FIREWALL   │  ← Checks your IP against the rules
  └─────────────┘
        │
   ✅ IP Allowed?        ❌ IP Blocked?
        │                      │
        ▼                      ▼
  SQL Server            Connection Refused
  (Access Granted)      Error 40615
```

---

## Why Firewall Rules Matter

| Without Firewall Rules | With Firewall Rules |
|------------------------|---------------------|
| Anyone on the internet can attempt to connect | Only whitelisted IPs can connect |
| Brute force attacks possible | Attackers can't even reach the server |
| Data exposed to the public | Data protected at network level |
| ❌ Not safe for production | ✅ Required for production |

> 🔐 **Azure SQL by default blocks ALL connections** — you must explicitly allow IPs to connect.

---

## Types of Firewall Rules

### 1. 🌐 Allow Azure Services Rule

Allows all services **inside Azure** to connect (Azure Functions, App Service, etc.)

```
Setting: "Allow Azure services and resources to access this server" → ON
IP Range: 0.0.0.0 → 0.0.0.0 (special Azure-internal rule)
```

> ⚠️ This allows **any** Azure customer's service too — use with caution in production.

---

### 2. 💻 Client IP Rule

Allows a **specific single IP address** to connect — typically your local machine.

```
Rule Name : MyLaptop
Start IP  : 203.0.113.45
End IP    : 203.0.113.45   ← same IP = single machine only
```

---

### 3. 📋 IP Range Rule

Allows a **range of IP addresses** — useful for office networks or teams.

```
Rule Name : OfficeNetwork
Start IP  : 203.0.113.0
End IP    : 203.0.113.255   ← entire /24 subnet allowed
```

---

### 4. 🔓 Open Rule *(DANGEROUS — Never use in production)*

Allows connections from **anywhere on the internet**.

```
Rule Name : DANGER_OpenAccess
Start IP  : 0.0.0.0
End IP    : 255.255.255.255   ← ⛔ Never do this in production!
```

---

## Firewall Rule Levels

Firewall rules exist at **2 levels** in Azure SQL:

```
┌──────────────────────────────────────────────┐
│             SERVER-LEVEL RULES               │
│                                              │
│  Applies to ALL databases on the server      │
│  Managed by: Server Admin / Azure Portal     │
│  Set in: master database                     │
│                                              │
│   ┌──────────────────────────────────────┐   │
│   │        DATABASE-LEVEL RULES          │   │
│   │                                      │   │
│   │  Applies to ONE specific database    │   │
│   │  Managed by: db_owner role           │   │
│   │  Set in: user database               │   │
│   └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

| Feature | Server-Level | Database-Level |
|---------|-------------|----------------|
| Scope | All databases | Single database |
| Who manages | Server admin | DB owner |
| Stored in | `master` db | User database |
| Use case | General access | Specific DB access |
| Azure Portal | ✅ Yes | ❌ T-SQL only |

---

## Setting Up During Server Creation

When creating an **Azure SQL Server** in the portal, you'll see the **Networking** tab:

### Step-by-Step

```
Azure Portal → Create SQL Database / Server
       │
       ▼
  Basics Tab          → Set server name, region, admin login
       │
       ▼
  Networking Tab      → ⬅️ FIREWALL RULES ARE SET HERE
       │
       ├── Connectivity Method
       │     ├── Public endpoint    (accessible over internet)
       │     └── Private endpoint   (accessible via VNet only)
       │
       ├── Firewall Rules
       │     ├── [ ] Allow Azure services to access server
       │     └── [ ] Add current client IP address  ⬅️ Tick this for local dev
       │
       ▼
  Review + Create     → Deploy
```

### Recommended Settings for Different Environments

#### 🧪 Development / Local Machine
```
✅ Allow Azure services     → ON  (for cloud tools)
✅ Add current client IP    → ON  (for your laptop)
Connectivity               → Public endpoint
```

#### 🏢 Staging / QA
```
✅ Allow Azure services     → ON
✅ Add specific office IPs  → Add your office IP range
Connectivity               → Public endpoint with restricted IPs
```

#### 🚀 Production
```
❌ Allow Azure services     → OFF (unless required)
❌ Add current client IP    → OFF
✅ Private endpoint only    → ON  (most secure)
✅ VNet Integration         → Recommended
```

---

## Practical Examples

### Example 1: Allow Your Local Machine

```sql
-- Via T-SQL (run on master database)
EXECUTE sp_set_firewall_rule
    @name = N'MyLaptop',
    @start_ip_address = '203.0.113.45',
    @end_ip_address = '203.0.113.45'
```

### Example 2: Allow an Office Network

```sql
EXECUTE sp_set_firewall_rule
    @name = N'MumbaiOffice',
    @start_ip_address = '203.0.113.0',
    @end_ip_address = '203.0.113.255'
```

### Example 3: Allow Multiple Locations

```sql
-- Head Office
EXECUTE sp_set_firewall_rule
    @name = N'HeadOffice_Mumbai',
    @start_ip_address = '203.0.113.0',
    @end_ip_address = '203.0.113.255'

-- Branch Office
EXECUTE sp_set_firewall_rule
    @name = N'BranchOffice_Delhi',
    @start_ip_address = '198.51.100.0',
    @end_ip_address = '198.51.100.255'

-- Remote Developer
EXECUTE sp_set_firewall_rule
    @name = N'RemoteDev_John',
    @start_ip_address = '192.0.2.10',
    @end_ip_address = '192.0.2.10'
```

### Example 4: Database-Level Rule (T-SQL only)

```sql
-- Connect to the specific database, then run:
EXECUTE sp_set_database_firewall_rule
    @name = N'ReportingTeam',
    @start_ip_address = '203.0.113.50',
    @end_ip_address = '203.0.113.60'
```

### Example 5: Remove a Firewall Rule

```sql
-- Remove server-level rule
EXECUTE sp_delete_firewall_rule
    @name = N'MyLaptop'

-- Remove database-level rule
EXECUTE sp_delete_database_firewall_rule
    @name = N'ReportingTeam'
```

### Example 6: Azure CLI

```bash
# Add a firewall rule via Azure CLI
az sql server firewall-rule create \
    --resource-group MyResourceGroup \
    --server my-sql-server \
    --name MyLaptop \
    --start-ip-address 203.0.113.45 \
    --end-ip-address 203.0.113.45

# List all firewall rules
az sql server firewall-rule list \
    --resource-group MyResourceGroup \
    --server my-sql-server

# Delete a rule
az sql server firewall-rule delete \
    --resource-group MyResourceGroup \
    --server my-sql-server \
    --name MyLaptop
```

### Example 7: PowerShell / ARM

```powershell
# Add firewall rule via PowerShell
New-AzSqlServerFirewallRule `
    -ResourceGroupName "MyResourceGroup" `
    -ServerName "my-sql-server" `
    -FirewallRuleName "MyLaptop" `
    -StartIpAddress "203.0.113.45" `
    -EndIpAddress "203.0.113.45"

# Get all rules
Get-AzSqlServerFirewallRule `
    -ResourceGroupName "MyResourceGroup" `
    -ServerName "my-sql-server"

# Remove a rule
Remove-AzSqlServerFirewallRule `
    -ResourceGroupName "MyResourceGroup" `
    -ServerName "my-sql-server" `
    -FirewallRuleName "MyLaptop"
```

---

## How to Manage Firewall Rules

### View Existing Rules

```sql
-- View server-level rules
SELECT *
FROM sys.firewall_rules
ORDER BY start_ip_address

-- View database-level rules
SELECT *
FROM sys.database_firewall_rules
ORDER BY start_ip_address
```

**Output columns:**

| Column | Description |
|--------|-------------|
| `id` | Unique rule ID |
| `name` | Rule name you gave it |
| `start_ip_address` | Start of allowed IP range |
| `end_ip_address` | End of allowed IP range |
| `create_date` | When rule was created |
| `modify_date` | When rule was last modified |

### Find Your Current IP

```sql
-- Find the IP SQL Server sees you connecting from
SELECT client_net_address
FROM sys.dm_exec_connections
WHERE session_id = @@SPID
```

---

## Common Connection Errors

### ❌ Error 40615 — IP Not Allowed
```
Cannot open server 'myserver' requested by the login.
Client with IP address '203.0.113.45' is not allowed to access the server.
```
**Fix:** Add your IP to the firewall rules in Azure Portal or via T-SQL.

---

### ❌ Error 40197 — Service Unavailable
```
The service has encountered an error processing your request.
```
**Fix:** Check if "Allow Azure services" is enabled if connecting from an Azure resource.

---

### ❌ Error 10060 — Connection Timeout
```
A network-related or instance-specific error occurred.
The server was not found or was not accessible.
```
**Fix:** Check firewall rules AND verify server name/port (default: 1433).

---

### ❌ Error 18456 — Login Failed
```
Login failed for user 'username'.
```
**Fix:** This is NOT a firewall error — IP is allowed but credentials are wrong.

---

## Best Practices

- ✅ **Always use specific IPs** instead of wide ranges in production
- ✅ **Name rules clearly** — e.g., `Mumbai_Office_2ndFloor`, `Dev_John_HomeIP`
- ✅ **Use Private Endpoints** for production databases (most secure)
- ✅ **Use VNet Service Endpoints** for Azure-to-Azure communication instead of "Allow Azure services"
- ✅ **Audit firewall rules regularly** — remove stale/unused rules
- ✅ **Document all rules** — note who owns each rule and why it exists
- ✅ **Use database-level rules** for multi-tenant scenarios (different clients, different DBs)
- ❌ **Never use 0.0.0.0 – 255.255.255.255** — this opens the server to the entire internet
- ❌ **Don't leave "Add current client IP" on in production** — your IP may change
- ❌ **Avoid "Allow Azure services" in production** unless absolutely needed — it's too broad
- ❌ **Don't share firewall admin credentials** — use Azure RBAC roles instead

---

## Security Layers (Defence in Depth)

Firewall rules are just ONE layer. Use all layers together:

```
Layer 1 → 🔥 Firewall Rules          (IP whitelist)
Layer 2 → 🔐 Strong Authentication   (complex passwords / Azure AD)
Layer 3 → 🔑 Least Privilege         (only grant needed permissions)
Layer 4 → 🔒 Encryption              (TLS in transit, TDE at rest)
Layer 5 → 👁️  Auditing & Monitoring  (Azure Defender, audit logs)
```

---

## FAQ

**Q: Can I set firewall rules before the server is created?**
> Yes — the Azure Portal lets you configure firewall rules on the **Networking tab** during server creation.

**Q: I ticked "Add current client IP" but still can't connect. Why?**
> Your IP might be dynamic (changes on router restart). Check your current IP at [whatismyip.com](https://whatismyip.com) and update the rule.

**Q: What port does Azure SQL use?**
> Default is **TCP port 1433**. Make sure your local firewall/router also allows outbound traffic on port 1433.

**Q: What's the difference between "Allow Azure services" and a Private Endpoint?**
> "Allow Azure services" allows all Azure IPs (including other customers). A **Private Endpoint** restricts access to only your specific VNet — much more secure.

**Q: Can I have both server-level and database-level rules?**
> Yes. If EITHER level allows your IP, access is granted. Database-level rules are evaluated after server-level rules pass.

**Q: How many firewall rules can I have?**
> Azure SQL supports up to **128 server-level firewall rules** per server.

**Q: Does the firewall rule apply immediately?**
> Yes — firewall rule changes take effect **within seconds** with no server restart needed.

---

## 📖 Further Reading

- [Microsoft Docs — Azure SQL Firewall Rules](https://docs.microsoft.com/en-us/azure/azure-sql/database/firewall-configure)
- [Private Link for Azure SQL](https://docs.microsoft.com/en-us/azure/azure-sql/database/private-endpoint-overview)
- [VNet Service Endpoints](https://docs.microsoft.com/en-us/azure/azure-sql/database/vnet-service-endpoint-rule-overview)
- [Azure SQL Security Overview](https://docs.microsoft.com/en-us/azure/azure-sql/database/security-overview)
- [sp_set_firewall_rule (T-SQL)](https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-set-firewall-rule-azure-sql-database)

---

*Generated with ❤️ — Feel free to contribute or raise issues!*
