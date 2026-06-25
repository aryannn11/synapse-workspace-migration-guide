# Azure Synapse Workspace Redeployment SOP

A complete, step-by-step Standard Operating Procedure for replicating an Azure Synapse workspace from a **source subscription** to a **destination subscription** using Azure DevOps Git integration and Classic Release Pipelines.

---

## What This Covers

- Migrating Synapse artifacts: pipelines, datasets, data flows, notebooks, SQL scripts, triggers, and integration runtimes
- Dedicated SQL Pool cross-subscription restore via PowerShell
- Apache Spark Pool recreation
- Azure DevOps Classic Release Pipeline setup
- Self-Hosted and Microsoft-Hosted agent configuration
- Post-deployment validation checklist

---

## Table of Contents

1. [Objective](#objective)
2. [Prerequisites](#prerequisites)
3. [Migration Procedure](#migration-procedure)
   - [Step 1 — Link Source Synapse to Azure DevOps](#step-1--link-source-synapse-workspace-to-azure-devops)
   - [Step 2 — Publish Artifacts from Source Synapse](#step-2--publish-artifacts-from-source-synapse)
   - [Step 3 — Migrate Dedicated SQL Pool](#step-3--migrate-dedicated-sql-pool-data-if-required)
   - [Step 4 — Migrate Supporting Azure Resources](#step-4--migrate-supporting-azure-resources)
   - [Step 5 — Fix Source Synapse Connectivity](#step-5--fix-source-synapse-connectivity)
   - [Step 6 — Create Destination Synapse Workspace](#step-6--create-destination-synapse-workspace)
   - [Step 7 — Recreate Apache Spark Pool](#step-7--recreate-apache-spark-pool)
   - [Step 8 — Update Configurations Manually](#step-8--update-configurations-manually)
   - [Step 9 — Azure DevOps Configuration](#step-9--azure-devops-configuration)
4. [Template Hygiene (Optional)](#template-hygiene-optional)
5. [Post-Deployment Validation](#post-deployment-validation)

---

## Objective

This SOP provides a step-by-step procedure for replicating an existing Azure Synapse workspace from a source subscription to a destination subscription using Azure DevOps Git integration and Release Pipelines. The objective is to migrate Synapse artifacts while maintaining configuration consistency.

---

## Prerequisites

- Access to both source and destination Azure subscriptions
- Azure DevOps organisation with appropriate permissions
- PowerShell with `Az` module installed
- Owner or Contributor role on both subscriptions

---

## Migration Procedure

### Step 1 — Link Source Synapse Workspace to Azure DevOps

The source Azure Synapse workspace must be connected to an Azure DevOps Git repository.

1. Open the **Source Synapse Workspace** → navigate to **Manage → Git configuration**
2. Select **Azure DevOps Git** as the repository type
3. Provide the following details:
   - Azure DevOps organisation
   - Project
   - Repository
   - Collaboration branch
4. Save the configuration

---

### Step 2 — Publish Artifacts from Source Synapse

Publishing the Synapse workspace generates ARM deployment templates.

1. Open **Synapse Studio** → click **Publish** on any pipeline or dataset
2. This generates deployment templates in the repository under the `workspace_publish` branch
3. The branch will include the following key artefacts:
   - `TemplateForWorkspace.json`
   - `TemplateParametersForWorkspace.json`

---

### Step 3 — Migrate Dedicated SQL Pool Data (If Required)

A Dedicated SQL Pool cannot be migrated directly and must be restored in the destination subscription.

#### Restore via Azure Portal

1. Navigate to **Source Subscription → SQL Server → Dedicated SQL Pool Database**
2. Click **Backups → Restore**
3. Configure:
   - Target Subscription
   - Target Resource Group
   - Target SQL Server (create if required)
   - Restore Point (Latest recommended)
4. Click **Restore**

#### Phase 1 — Create Restore Point (Run on Source)

```powershell
Write-Host "Phase 1 Starts" -ForegroundColor Green

# Set variables
$TenantId                = "<Tenant-ID>"
$SourceSubscriptionId    = "<Source-Subscription-ID>"
$DestinationSubscriptionId = "<Destination-Subscription-ID>"

Set-AzContext -SubscriptionId $SourceSubscriptionId

$SourceResourceGroupName = "<Source-ResourceGroup-Name>"
$Location                = "<Location>"
$SourceWorkspaceName     = "<Source-SynapseWorkspaceName>"
$SourceDedicatedPoolName = "<Source-DedicatedPoolName>"
$TargetResourceGroupName = "<Destination-ResourceGroup-Name>"
$TargetServerName        = "<Destination-SQLServer-Name>"

# Create a restore point
New-AzSynapseSqlPoolRestorePoint `
    -WorkspaceName $SourceWorkspaceName `
    -Name $SourceDedicatedPoolName `
    -RestorePointLabel "UserDefined-RP"

$Database     = Get-AzSynapseSqlPool `
    -ResourceGroupName $SourceResourceGroupName `
    -WorkspaceName $SourceWorkspaceName `
    -Name $SourceDedicatedPoolName

$RestorePoint = $Database | Get-AzSynapseSqlPoolRestorePoint | Select -Last 1

Write-Host "Restore point created in Source Synapse Workspace" -ForegroundColor Green
Write-Host "Phase 1 Ends" -ForegroundColor Green
```

#### Phase 2 — Restore to Destination Subscription

```powershell
Write-Host "Phase 2 Starts" -ForegroundColor Green

# Set variables
$TenantId                  = "<Tenant-ID>"
$SourceSubscriptionId      = "<Source-Subscription-ID>"
$DestinationSubscriptionId = "<Destination-Subscription-ID>"

Set-AzContext -SubscriptionId $SourceSubscriptionId

$SourceResourceGroupName = "<Source-ResourceGroup-Name>"
$Location                = "<Location>"
$SourceWorkspaceName     = "<Source-SynapseWorkspaceName>"
$SourceDedicatedPoolName = "<Source-DedicatedPoolName>"
$TargetResourceGroupName = "<Destination-ResourceGroup-Name>"
$TargetServerName        = "<Destination-SQLServer-Name>"
$PerformanceLevel        = "<Performance-Level>"   # e.g. DW100c

Set-AzContext -SubscriptionId $DestinationSubscriptionId

$RestorePoint = $Database | Get-AzSynapseSqlPoolRestorePoint | Select -Last 1

Write-Host "This is the Latest Restore Point" -ForegroundColor Green

# Restore database to destination subscription
$RestoredDatabase = Restore-AzSqlDatabase `
    -FromPointInTimeBackup `
    -PointInTime $PointInTime `
    -ResourceGroupName $TargetResourceGroupName `
    -ServerName $TargetServerName `
    -TargetDatabaseName $SourceDedicatedPoolName `
    -ResourceId $Database.ID `
    -Edition "DataWarehouse" `
    -ServiceObjectiveName $PerformanceLevel

Write-Host "SQL Database restored in Destination SQL Server" -ForegroundColor Green

# Create restore point in destination SQL Server
New-AzSqlDatabaseRestorePoint `
    -ResourceGroupName $RestoredDatabase.ResourceGroupName `
    -ServerName $RestoredDatabase.ServerName `
    -DatabaseName $RestoredDatabase.DatabaseName `
    -RestorePointLabel "UserDefined-RP"

Write-Host "Restore point created in Destination SQL Server" -ForegroundColor Green

# Restore as Dedicated SQL Pool in destination Synapse workspace
$TargetWorkspaceName = "<Destination-SynapseWorkspace-Name>"

$RestorePoint = Get-AzSqlDatabaseRestorePoint `
    -ResourceGroupName $RestoredDatabase.ResourceGroupName `
    -ServerName $RestoredDatabase.ServerName `
    -DatabaseName $RestoredDatabase.DatabaseName | Select -Last 1

Restore-AzSynapseSqlPool `
    -FromRestorePoint `
    -RestorePoint $RestorePoint.RestorePointCreationDate `
    -ResourceGroupName $TargetResourceGroupName `
    -WorkspaceName $TargetWorkspaceName `
    -TargetSqlPoolName $SourceDedicatedPoolName `
    -ResourceId $RestoredDatabase.ResourceID `
    -PerformanceLevel $PerformanceLevel

Write-Host "Dedicated Pool created in Destination Synapse Workspace" -ForegroundColor Green
Write-Host "Phase 2 Ends" -ForegroundColor Green
```

---

### Step 4 — Migrate Supporting Azure Resources

Before deploying the Synapse workspace in the target subscription, migrate all dependent Azure resources:

- Storage Accounts
- Azure SQL Databases
- Azure Key Vault
- Data Lake Storage (ADLS Gen2)
- Networking components

> Ensure all resources are accessible from the destination Synapse workspace before proceeding.

---

### Step 5 — Fix Source Synapse Connectivity

If the source workspace loses connectivity after resource migration:

1. Reconfigure **Managed Private Endpoints (MPE)**
2. Update **Linked Services** to reference the migrated resources and validate connectivity

This step ensures the source workspace remains operational during parallel deployment.

---

### Step 6 — Create Destination Synapse Workspace

Create a new Synapse workspace in the destination subscription with identical configuration.

#### Attach Dedicated SQL Pool

1. Open **Destination Synapse Workspace → SQL Pools → + New**
2. Select **Use Existing Database (Restore)**
3. Choose the restored database
4. Configure:
   - Performance Level (DWU)
   - Auto-pause settings
5. Click **Create**

---

### Step 7 — Recreate Apache Spark Pool

1. Navigate to **Manage → Apache Spark Pools → + New**
2. Configure using previously documented settings:
   - Spark Version
   - Node Size
   - Auto-scale
   - Auto-pause
3. Click **Create**

---

### Step 8 — Update Configurations Manually

Recreate the following key Synapse components in the destination workspace:

- **Managed Private Endpoint**
- **SHIR (Self-Hosted Integration Runtime)** — to be done by the client

---

### Step 9 — Azure DevOps Configuration

Deploy Synapse artifacts using a **Classic Release Pipeline** on the Azure DevOps portal.

#### 9.1 Required Azure DevOps Roles

| Level | Role Required |
|-------|--------------|
| Organisation | Project Collection Administrator (PCA) |
| Project | Project Administrator |

#### 9.2 Install Synapse Workspace Deployment Extension

1. Open **Azure DevOps Marketplace** and search for `Synapse Workspace Deployment`
2. Click **Get it Free**
3. Select your Azure DevOps organisation and confirm the installation

#### 9.3 Enable Classic Release Pipelines

1. Navigate to **Organisation Settings → Pipelines → Settings**
2. Toggle **off**:
   - Disabled creation of classic build pipelines
   - Disable creation of classic release pipelines

#### 9.4 Configure Service Connection (ARM)

1. Navigate to **Project Settings → Service Connections**
2. Click **New Service Connection → Azure Resource Manager**
3. Choose Identity Type → select the target subscription, resource group, and service connection name
4. Grant access permission to all pipelines
5. Assign **Synapse Administrator** RBAC role on the target workspace under **Access Control**

#### 9.5 Ensure Agent Availability

**Option A — Microsoft Hosted Agent** ($40/month per job)

1. Navigate to **Organisation Settings → Billing → Parallel Jobs**
2. Ensure at least 1 Microsoft-hosted parallel job is available

**Option B — Self-Hosted Agent** (1 free job)

**Step 1 — Create a VM on Azure**
- OS: Windows Server 2022
- Recommended size: `B2ms`
- Connect via RDP

**Step 2 — Create Agent Pool**
1. Open **Azure DevOps → Organisation Settings → Agent Pools**
2. Click **Add Pool**
3. Configure:
   - Pool Name: `Self Hosted`
   - Pool Type: `Self-Hosted`

**Step 3 — Create Personal Access Token (PAT)**
1. Open **User Profile → Personal Access Tokens → New Token**
2. Configure:
   - Name: `SelfHostedAgentPAT`
   - Expiration: 90–180 days
   - Scopes: Agent Pools → Read & Manage
3. Copy the generated token

**Step 4 — Prepare Agent Directory**

```powershell
mkdir C:\ado-agent
cd C:\ado-agent
```

**Step 5 — Download Agent Package**
1. Navigate to **Azure DevOps → Agent Pools → New Agent**
2. Select **Windows x64**
3. Download and extract the package to `C:\ado-agent`

**Step 6 — Configure Agent**

Run `config.cmd` and provide:

| Prompt | Value |
|--------|-------|
| Server URL | `https://dev.azure.com/<org-name>` |
| Authentication | PAT |
| Agent Pool | `Self-Hosted` |
| Agent Name | Default or custom |
| Run as Service | Yes |
| Start Service Now | Yes |

**Step 7 — Verify Agent**

Navigate to **Organisation Settings → Agent Pools → Self Hosted** — the agent should show **Online**.

> If the agent shows **Offline**, restart it with:
> ```powershell
> Restart-Service vstsagent*
> ```

#### 9.6 Create Classic Release Pipeline

**Step 1 — Create Pipeline**
1. Navigate to **Pipelines → Releases → New Pipeline**

**Step 2 — Create Stage**
1. Click **Stage → Empty Job**
2. Configure the Agent Job:
   - Agent Pool: Microsoft-hosted or Self-Hosted
   - Agent Specification: `Windows-2022` or `Windows-latest` (for Microsoft-hosted)

**Step 3 — Add Deployment Task**

Add task: **Synapse Workspace Deployment** and configure:

| Field | Value |
|-------|-------|
| Operation | Deploy |
| Template File | `TemplateForWorkspace.json` |
| Parameters File | `TemplateParametersForWorkspace.json` |
| Azure Connection | ARM Service Connection |
| Resource Group | Target Resource Group |
| Workspace Name | Target Synapse Workspace |
| Delete artifacts not in template | Uncheck |
| Deploy Managed Private Endpoints | Uncheck |

**Step 4 — Add Artifact**
1. Select **Azure Repos Git**
2. Choose the Synapse repository
3. Select branch: `workspace_publish`

**Step 5 — Deploy Release**
1. Save the pipeline
2. Click **Create Release**
3. Click on Stage (1 job, 0 task) → add Agent Job → complete Step 3 configuration
4. After completing Synapse Workspace Deployment configuration, complete Step 4
5. Select the stage → click **Deploy**

---

## Template Hygiene (Optional)

Before deployment, review templates to avoid incorrect configurations.

**Recommended checks:**

- [ ] Remove source environment secrets from the template
- [ ] Parameterise linked services in the template
- [ ] Remove Managed Private Endpoints from the template (they require manual approval)

---

## Post-Deployment Validation

After deployment, verify the following:

- [ ] Linked services updated by the client
- [ ] Linked services connect successfully
- [ ] Managed private endpoints are created and approved
- [ ] Access control and RBAC roles are configured
- [ ] Networking configuration matches the source workspace
- [ ] Managed identity permissions are assigned
- [ ] Triggers are enabled

---

## Contributing

Found an issue or have an improvement? Feel free to open a PR or raise an issue — always happy to improve this guide for the community.
Best Regards,
Aryan Kaushik

---

*Created as part of a personal learning journey in Azure cloud engineering.*