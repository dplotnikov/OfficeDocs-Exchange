---
localization_priority: Normal
description: 'Summary: Learn about the MetaCacheDatabase (MCDB) feature in Exchange Server 2019, and how to configure it in your organization.'
ms.topic: overview
author: msdmaguire
ms.author: dmaguire
monikerRange: exchserver-2019
title: MetaCacheDatabase (MCDB) setup
ms.collection: exchange-server
ms.reviewer: toklima
ms.audience: ITPro
ms.prod: exchange-server-it-pro
manager: serdars

---

# MetaCacheDatabase (MCDB) setup

The MetaCacheDatabase (MCDB) feature is included in Exchange Server 2019. It allows a database availability group (DAG) to be accelerated by utilizing solid state disks (SSDs). `Manage-MetaCacheDatabase.ps1` is an automation script created for Exchange Server administrators to set up and manage MCDB instances in their Exchange 2019 DAGs.

After installing Exchange Server 2019, you can find `Manage-MetaCacheDatabase.ps1` here: ***drive*:\\Program Files\\Microsoft\\Exchange Server\\V15\\Scripts**

You use this script to configure MCDB prerequisites on a properly configured DAG, to enable or disable MCDB, and to configure and repair MCDB on your servers.

## SSD guidance

All SSDs used for MCDB need to be of the same capacity and type. A symmetrical configuration between servers is required, which means there needs to be an identical number of SSDs in each server, and the SSDs all need to be the same size.

> [!NOTE]
> The `Manage-MCDB` cmdlet will only work with devices exposed as **MediaType SSD** by Windows.

It is recommended to target a 1:3 ratio between SSD and HDD devices per server. Therefore, deploy one SSD for every three HDDs. In order to avoid having to reduce the number of HDDs in the server, consider using M.2 form factor SSDs.

Providing 5% to 6% of SSD capacity relative to total HDD capacity is sufficient for on-premises deployments. For example, if your server contains 100 TB of HDD capacity for mailbox databases, an allocation of 5 TB to 6 TB for SSD capacity is enough.

The SSDs you use should qualify for “mixed use” and support one drive write per day (DWPD) or greater in terms of write endurance.

## Prerequisites

The following prerequisites are required for successful configuration and use of MCDB:

1.  The DAG is configured for auto-reseed.

2.  RAW SSD drives are installed, and SSD count and size is equal for each server in the DAG.

3.  Exchange Server 2019.

## MCDB setup:

The process of setting up MCDB can be broken down into four basic steps:

1.  Update Active Directory (AD) settings and wait for propagation (by running `ConfigureMCDBPrerequisite`).

2.  Allow MCDB acceleration for each server of the DAG (by running `ServerAllowMCDB`).

3.  Create the necessary infrastructure (Volumes, Mount Points) for MCDB on each server (by running `ConfigureMCDBOnServer`).

4.  Let databases fail over to pick up the new settings.

After successful execution of all four steps, MCDB acceleration will begin for every database instance with a corresponding MCDB instance.

The following sections describe how to utilize the `Manage-MetaCacheDatabase.ps1` script to achieve the above four steps.

### 1. Run `Manage-MCDB -ConfigureMCDBPrerequisite`

This parameter sets the Active Directory state for the DAG object. Full replication of the Active Directory state is required before MCDB can function properly on all servers.

**ParameterSetIdentifier**:

- `ConfigureMCDBPrerequisite`

**Parameters**:

| Parameter          | Required   | Description                                                          |
|--------------------|------------|----------------------------------------------------------------------|
| DagName            | True       | Name of the Database availability group.                             |
| SSDSizeInBytes     | True       | The capacity in bytes of each SSD in the server to be used for MCDB. |
| SSDCountPerServer  | True       | The count of SSD devices to be utilize for MCDB in each server.      |

**Scope**: 

- **DAG**: `ConfigureMCDBPrerequisite` operates on a DAG object.

> [!NOTE]
> MCDB will utilize up to 95% of an SSD’s physical capacity. The remaining 5% is kept free to account for file system and partition overhead, as well as for a small amount of additional buffer and over-provisioning.

**Example**:

`Manage-MCDB -DagName TestDag1 -ConfigureMCDBPrerequisite -SSDSizeInBytes 5242880000 -SSDCountPerServer 2`

![MCDB configure prerequisites](../../media/mcdb1.png)

### 2. Run `Manage-MCDB -ServerAllowMCDB`

This parameter sets the local state on each DAG member to allow/disallow MCDB population and read acceleration.

**ParameterSetIdentifier**:

- `ServerAllowMCDB`

**Parameters**:

| Parameter          | Required   | Description                                                          |
|--------------------|------------|----------------------------------------------------------------------|
| DagName            | True       | Name of the Database availability group.                             |
| ServerName         | True       | Specifies the server to enable MetaCacheDatabase on.                 |
| ForceFailover      | Optional   | This Boolean switch can be utilized to cause all databases on a server to fail over. This is required to make all configuration changes take effect and to begin utilizing MCDB after mount points and database instances have been successfully created in [3. Run Manage-MCDB -ConfigureMCDBOnServer](#3-run-configuremcdbonserver). It is also needed to disable SSD acceleration.      |

**Scope**: 

- **Server**: `ServerAllowMCDB` has to be executed on each server in the DAG.

**Examples**:

`Manage-MCDB -DagName TestDag1 -ServerAllowMCDB:$true -ServerName "exhs-5046"`

`Manage-MCDB -DagName TestDag1 -ServerAllowMCDB:$false -ServerName "exhs-5046" -ForceFailover:$true`

![MCDB run ServerAllowMCDB](../../media/mcdb2.png)

### 3. Run `Manage-MCDB -ConfigureMCDBOnServer`

This parameter identifies unformatted SSD devices and formats them, and also creates the necessary mount points on a server for hosting MCDB instances. This parameter set can also be used to re-create mount points on a raw SSD that was added to replace a failed SSD.

**ParameterSetIdentifier**:

- `ConfigureMCDBOnServer`

**Parameters**:

| Parameter          | Required   | Description                                                                         |
|--------------------|------------|-------------------------------------------------------------------------------------|
| DagName            | True       | Name of the Database availability group.                                            |
| ServerName         | True       | Specifies the server to identify unformatted SSD devices and create mount points on.|
| SSDSizeInBytes     | True       | This is the capacity, in bytes, of each SSD in the server to be used for MCDB.      |

**Scope**: 

- **Server**: `ConfigureMCDBOnServer` has to be executed on each server in the DAG.

Example:

`Manage-MCDB -DagName TestDag1 -ConfigureMCDBOnServer -ServerName "exhs-4056" -SSDSizeInBytes 5242880000`

![Run MCDBOnServer1](../../media/mcdb3.png)

![Run MCDBOnServer2](../../media/mcdb4.png)

After performing the previous three steps (running `ConfigureMCDBPrerequisite`, `ServerAllowMCDB`, and `ConfigureMCDBOnServer`), the MCDB state will display as **Storage Offline**. This means that the environment is prepared and ready for MCDB instances to be created and populated. The next fail over of the database instance will cause the creation of the MCDB instance and enable acceleration. The instances will transition through the health states shown in [MCDB health states](#mcdb-health-states).

You can use the `ServerAllowMCDB` parameter set to cause fail overs of all DB instances present on a given server. Alternatively, the `Move-ActiveMailboxDatabase` cmdlet can be used to cause individual databases to fail over.

`Manage-MCDB.ps1 -DagName TestDag1 -ServerAllowMCDB:$true -ServerName “exhs-5046” -ForceFailover:$true`

## MCDB health states

Use `Get-MailboxDatabaseCopyStatus` to query the state of the MCDB instances. There are five states that an MCDB instance can be in, as shown in the following table:

| State          | Description                                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Disabled       | MCDB is turned off.                                                                                                                   |
| StorageOffline | Basic infrastructure is missing or inaccessible, such as mount points or file paths. This is the state MCDB is in following an SSD failure. |
| Offline        | Errors at the logical level, for example missing MCDB instances.                                                                     |
| Initializing   | Transient state, the system is determining what other state it should be in.                                                         |
| Healthy        | Ready to serve requests.                                                                                                             |

