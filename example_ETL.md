# ETLs - Azure VM Development Migration
> [!CAUTION]
> **NOTE:** This is an example template only.

## General Information

| Field | Value |
| --- | --- |
| **Application Name** | ETLs |
| **Application ID** |  |
| **Application Stacks** |  |
| **Environments** | Development |
| **Business or Technical Function** | Python, VBScript and DOS command shell. The processes are scheduled across a number of servers using Windows Task Scheduler or Oracle Job Scheduler. |
| **Primary Contacts** | Technical / Application Owner: <br> Business Owner: <br> Business Unit: Information Services |
| **Service Characteristics** | Describe primary function for this application and any other secondary use case relevant to this design |
| **Data Classification** | OFFICIAL:SENSITIVE |
| **Migration Strategy** | Replatform |
| **Migration Pattern** | Replatform-App |
| **Region(s)** | australiaeast |
| **Number of Availability Zones** | 1 |
| **Azure Subscription** | Account ID / Account Name |
| **URL Change** | Yes / No (list current and proposed URLs) |

---

## Current Application Documentation

List documentation for the current implementation (on-premises, colocation, other).

| **Version** | **Document Type** | **Title** | **Links** |
| --- | --- | --- | --- |
| 1.1 | Confluence Pages | Document Title |  |
|  |  | ETL Testing - Azure VM vs Onprem |  |

---

## Technical Data & Dependencies

### PROD Current Architecture

| **App Server** | **Role** | **OS** | **IP Address** | **URLs / Notes** | **DNS Records** |
| --- | --- | --- | --- | --- | --- |
| `<Server ID>` | Windows Server (scheduled tasks) |  |  |  | A Record points to IP address: <br>`cmd > nslookup -type=a -debug` <br>`cmd > nslookup -type=cname -debug` <br>Note TTL as this may need to be lowered |

---

### ETLs

|  | **Folder** | **Task** | **Schedule** | **Purpose** | **ETL Sources** | **ETL Targets** | **Migration Pattern** | **Notes/Issues** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 |  <br>Key Contacts: | Update `<components>` | Weekdays at 12:30AM |  |  |  | Retain |  |
|  |  |  |  |  |  |  |  |  |

---

### Architecture Integrations and Interactions

| **Integrations** | **Notes** |
| --- | --- |
| File Transfers (SFTP / FTP)? |  |
| File Share Access? |  |
| Other application? |  |
| Email | Email SMTP server settings should be updated for Azure. <br>**SMTP Settings to use in Azure:** <br>SMTP Server: `<Server Name>` <br>Port: `<Port Number>` |
| BI / Reporting |  |

---

### Current Backup Environment

> *(To be documented)*

---

### Licensing

| **License** | **Notes** |
| --- | --- |
|  | Review to determine if tied to unique attribute on source server such as MAC address / Hardware ID |

---

### External Organization Access

| **Organization** | **Notes** |
| --- | --- |
|  | List if any other organizations access this application. |

---

## Non-Technical Dependencies

*(To be documented)*

---

## Testing Strategy

- What test strategy will be implemented for this application?
- When will testing need to start — e.g. prior to migration wave or during migration wave?

---

## Migration Strategy & Pattern

### Strategy

Replatform strategy will be used for the ETL server with a new server to be set up on Azure VM.

### Patterns & Tools

ETLs will be migrated individually based on their dependencies. Each ETL will be assessed, and a pattern assigned either:

- **Replatform** — where it is possible to move to Azure VM
- **Retain** — where dependencies prevent the timely execution of the ETL on Azure VM

### Cut-over Considerations

Document here any factors that need to be taken into consideration when planning for migration.

- Cut-over dates to avoid
- Skills required

---

## Azure Architecture Design

### Architectural Component Details

| **Logical Component Name** | **Description** | **Details** |
| --- | --- | --- |
| **Azure VM** | Azure VM App Server | Server ID |
| **Virtual Machine** | Onprem App Server | Server ID |

---

### Azure VM Instances

| **Server Name** | **VM Size** | **Storage** | **Azure Subscription Name** | **Azure Subscription ID** | **VNet Name** | **VNet ID** | **Subnet Name** | **Subnet ID** | **IP Address** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Server ID | Standard_D4s_v3 | C: 80GB <br>E: 50GB <br>All disks Premium SSD LRS |  |  |  |  |  |  |  |

---

### Network Security Groups (NSG)

The following existing NSGs are to be added to the DEV instance:

| **Security Group** | **Security Group ID** | **Protocol** | **Port** | **Source** |
| --- | --- | --- | --- | --- |
| NSG-Private-DEV-VNet-App-Management | nsg-dev-xxxxd0 | ssh, RDP | 22, 3389 | Prefix List - Security-Application-Management-Support <br>Service Tag - NSG-Private-DEV-VNet-App-Management |
| NSG-Appliance-Scanning-DEV | nsg-dev-uhdsjkjs27 | all | all | Prefix List - Security-Security-Scanning-PrefixList |

**New Azure NSG to be created:** `NSG-DEV-SGNAME` (`nsg-id`)

The following rules are to be added to the new Security Group:

| **Source Host** | **Source** | **Protocol** | **Port** | **Comment** |
| --- | --- | --- | --- | --- |
| Managed Prefix List | DTUP-Domain-Controllers | udp | UDP 123 | Allow NTP from Domain Controllers |
| Managed Prefix List | DTUP-Domain-Controllers | smb | TCP / UDP 445 | Allow SMB from Domain Controllers |
| Managed Prefix List | DTUP-Domain-Controllers | rpc | TCP / UDP 135 | Allow RPC from Domain Controllers |
| Managed Prefix List | DIT-Statenet-Prefixes | smb | TCP / UDP 445 | Allow SMB from DIT Statenet for File Shares |
|  |  |  |  |  |

---

### Azure Backup Environment

Refer to Operational Handover & Readiness Confluence page:

> `<INSERT LINK TO BACKUPS SECTION FROM OPERATIONAL & HANDOVER>`

---

## Annual Run Rate Estimate

| **Applications** | **Names** | **OS** | **Instance Type** | **Tenancy** | **# Instances** | **Usage (hrs/week)** | **Usage Type** | **Purchasing Option** | **Storage Type** | **Storage per Instance (GB)** | **Monthly (USD)** | **Annual (USD)** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |  |  |  |  |  |  |  |

This estimate is based on the following assumptions:

- 1 Yr Reserved Instances (Dev/Test pricing) are used for Development Instances.
- Azure Disk Snapshots not calculated.
- Utilization for all Development instances (VM/Managed Disk) is per dev schedule.
- Utilization for all Development instances is 5 days/week at 10hrs/day — 50hrs/week.
- Does not include Oracle Licensing.
- Includes Development environment only.
- Includes Windows OS licensing for all Windows instances.
- Includes RHEL Licensing for all Linux instances.
- All internal networking and security is provided by Network Security Groups (NSG).
- Pricing is for Azure `australiaeast` region.

---

## Testing Requirements

Testing is performed both prior to and during the migration window. Test cases have been documented here:

> `<INSERT LINK TO TEST PLANS>`

---

## Migration Runbook

Link to migration runbook:

> `<INSERT LINK TO RUNBOOKS>`

---

## Open Questions / Parking Lot

List any part of the design that has not been resolved. This could be requirements, risks, or parts of other systems that can impact the design. The goal is to acknowledge these gaps and confirm they will be addressed later, or during discussion.

> **During discussion:** Is it okay to proceed with the design given that we don't know how some things will work?

| **Item** | **Description** |
| --- | --- |
|  |  |
|  |  |
