---
title: "Automatic seeding for secondary replicas (SQL Server) | Microsoft Docs"
description: "Use automatic seeding to initialize secondary replicas."
services: data-lake-analytics
ms.custom: ""
ms.date: "06/22/2017"
ms.prod: "sql-server-2016"
ms.reviewer: ""
ms.suite: ""
ms.technology: 
- "dbe-high-availability"
ms.tgt_pltfrm: ""
ms.topic: "article"
helpviewer_keywords: 
- "Automatic seeding [SQL Server], secondary replica"
ms.assetid: 
caps.latest.revision: 
author: "MikeRayMSFT"
ms.author: "mikeray"
manager: "jhubbard"
---
# Automatic seeding for secondary replicas

[!INCLUDE [tsql-appliesto-ss2016-xxxx-xxxx-xxx_md](../../../includes/tsql-appliesto-ss2016-xxxx-xxxx-xxx-md.md)]

In SQL Server 2012 and 2014, the only way to initialize a secondary replica in an availability group is to use backup, copy, and restore. SQL Server 2016 introduces a new feature to initialize a secondary replica – *automatic seeding*. Automatic seeding uses the log stream transport to stream the backup using VDI to the secondary replica for each database of the availability group using the configured endpoints. This new feature can be used either during the initial creation of an availability group or when a database is added to one. Automatic seeding is in all editions of SQL Server that support Always On availability groups, and can be used with both traditional availability groups and [distributed availability groups](distributed-availability-groups.md).

## Considerations

Considerations for using automatic seeding include:

* [Performance and transaction log impact on the primary replica](#performance-and-transaction-log-impact-on-the-primary-replica)
* [Disk layout](#disk-layout)
* [Security](#security)


### Performance and transaction log impact on the primary replica

Automatic seeding may or may not practical to initialize a secondary replica, depending on the size of the database, network speed, and distance between the primary and secondary replicas. For example, given:

* The database size is 5 TB
* The network speed is 1Gb/sec
* The distance between the two sites is 1000 miles

A 1Gb/sec network can provide sustained throughput of 125 MB/sec if the full bandwidth is available. In this example, automatic seeding would take just over 11 hours. In practice, the automatic seeding process is somewhat slower, as network signals degrade over longer distances and the link is usually shared with other resources on the network. During seeding, the transaction log on the database at the primary replica will continue to grow and cannot be truncated until automatic seeding of that database is complete.  The transaction log can then be truncated using a transaction log backup.

Automatic seeding is a single-threaded process that can handle up to five databases. This may affect performance, especially if the availability group has more than one database.

Compression can be used for automatic seeding, but it is disabled by default. Turning on compression reduces network bandwidth and possibly speeds up the process, but the tradeoff is additional processor overhead. To use compression during automatic seeding, enable trace flag 9567 - see [Tune compression for availability group](tune-compression-for-availability-group.md).

### Disk layout

The folder where the database will be created by automatic seeding must already exist and be the same as the path on the primary replica.

### Security

Security permissions vary depending on the type of replica being initialized:

* For a traditional availability group, permissions must be granted to the availability group on the secondary replica as it is joined to the availability group. In Transact-SQL, use the command ALTER AVAILABILITY GROUP [AGName] GRANT CREATE ANY DATABASE.
* For a distributed availability group where the replica’s databases that are being created are on the primary replica of the second availability group, no extra permissions are required as it is already a primary.
* For a secondary replica on the second availability group of a distributed availability group, you must use the command ALTER AVAILABILITY GROUP [2ndAGName] GRANT CREATE ANY DATABASE. This secondary replica will be seeded from the primary of the second availability group.

## Create an availability group with automatic seeding

You create an availability group using automatic seeding with either Transact-SQL or SQL Server Management Studio (SSMS, version 17 or later). To use the Availability Group Wizard in SSMS, follow [these instructions](use-the-availability-group-wizard-sql-server-management-studio.md) - when you get to Step 9, you will see automatic seeding as the first, and default, option.

![Select initial data synchronization][1]

The following example creates an availability group using Transact-SQL. See also the topic [Create an Availability Group (Transact-SQL)](create-an-availability-group-transact-sql.md). Seeding is enabled  on a secondary replica by setting the SEEDING_MODE option to AUTOMATIC. The default behavior is MANUAL, which is the pre-SQL Server 2016 behavior requiring a backup of the database to be made on the primary replica, a copy of the backup file to the secondary replica, and a restore of the backup WITH NORECOVERY.

```
CREATE AVAILABILITY GROUP [AGName]
  FOR DATABASE db1
  REPLICA ON N'Primary_Replica'
WITH (
  ENDPOINT_URL = N'TCP://Primary_Replica.Contoso.com:5022', 
  FAILOVER_MODE = AUTOMATIC, 
  AVAILABILITY_MODE = SYNCHRONOUS_COMMIT, 
),
  N'Secondary_Replica' WITH (
    ENDPOINT_URL = N'TCP://Secondary_Replica.Contoso.com :5022', 
    FAILOVER_MODE = AUTOMATIC, 
    SEEDING_MODE = AUTOMATIC);
 GO
```

Setting SEEDING_MODE on a primary replica during a CREATE AVAILABILITY GROUP statement has no effect since the primary replica already contains the main read/write copy of the database. SEEDING_MODE would only apply when another replica was made the primary and a database was added. The seeding mode can be changed later - see [Change the seeding mode of a replica](#change-the-seeding-mode-of-a-replica).

On an instance that becomes a secondary replica, once the instance is joined the following message is added to the SQL Server Log:

Local availability replica for availability group 'AGName' has not been granted permission to create databases, but has a SEEDING_MODE of AUTOMATIC. Use the ALTER AVAILABILITY GROUP … GRANT CREATE ANY DATABASE command to allow the creation of databases seeded by the primary availability replica.

After joining, issue the following statement:

```
ALTER AVAILABILITY GROUP [AGName] GRANT CREATE ANY DATABASE
 GO
````

> [!NOTE] 
> There is currently a known issue as of SQL Server 2016 SP1 CU2 where a secondary replica must wait three minutes to allow the AG to seed the database before executing an ALTER AVAILABILITY GROUP... statement. Before that time elapses, the statement will not return an error, rather it will indicate success. This is also a known issue. These issues will be fixed in a future update to SQL Server. As a workaround, insert a WAITFOR statement:

```
WAITFOR DELAY '00:03:15';
ALTER AVAILABILITY GROUP [AGName] GRANT CREATE ANY DATABASE;
GO
```

If successful, the database(s) are automatically created on the secondary replica with a state of either:

* SYNCHRONIZED if the secondary replica is configured to be synchronous and the data is completely synchronized.
* SYNCHRONIZING if the secondary replica is configured with asynchronous data movement, or when configured with synchronous but not yet synchronized with the primary replica.

<a name="sql-server-log"></a>
In addition to the [Dynamic Management Views](#dynamic-management-views) described below, the start and completion of automatic seeding can be seen in the SQL Server Log:

![SQL server log][2]

## Combine backup and restore with automatic seeding

It is possible to combine the traditional backup, copy, and restore with automatic seeding. In this case, first restore the database on a secondary replica including all available transaction logs. Next, enable automatic seeding when creating the availability group to "catch up" the secondary replica’s database, as if a tail-log backup were restored (see [Tail-Log Backups (SQL Server)](../../../relational-databases/backup-restore/tail-log-backups-sql-server.md)).

## Add a database to an availability group with automatic seeding

You can add a database to an availability group using automatic seeding using Transact-SQL or SQL Server Management Studio (SSMS, version 17 or later).
If the secondary replica used automatic seeding when it was added to the availability group, nothing additional needs to be done. If backup, copy, and restore was used, first change the seeding mode (see next section) and then when adding the database use the GRANT statement - see [Availability Group - Add a Database](availability-group-add-a-database.md).

## Change the seeding mode of a replica

A replica’s seeding mode can be altered after the availability group is created, so automatic seeding can be enabled or disabled. Enabling automatic seeding after creation allows a database to be added to the availability group using automatic seeding if it was created with backup, copy, and restore. For example:

```
ALTER AVAILABILITY GROUP [AGName]
  MODIFY REPLICA ON 'Replica_Name'
  WITH (SEEDING_MODE = AUTOMATIC)
```

To disable automatic seeding, use a value of MANUAL.

## Prevent automatic seeding after an availability group is created

If you do not want to disable automatic seeding completely for a secondary replica, but want to temporarily prevent the secondary replica from being able to automatically create databases, deny the availability group CREATE permission. This is the case when a new database is added to the availability group, but the availability group should not be allowed to create the database on a secondary replica.

```
ALTER AVAILABILITY GROUP [AGName] DENY CREATE ANY DATABASE
GO
```

## Monitor automatic seeding

There are four ways to monitor and troubleshoot automatic seeding:

* [SQL Server Log](#sql-server-log) as already described
* [Dynamic management views](#dynamic-management-views)
* [Backup history tables](#backup-history-tables)
* [Extended events](#extended-events)

### Dynamic Management Views

There are two dynamic management views (DMVs) for monitoring seeding: sys.dm_hadr_automatic_seeding and sys.dm_hadr_physical_seeding_stats.

* sys.dm_hadr_automatic_seeding contains the general status of automatic seeding, and retains the history for each time it is executed (whether successful or not). The column current_state will have either a value of COMPLETED or FAILED. If the value is FAILED, use the value in failure_state_desc to help in diagnosing the problem. You may need to combine that with what it in the [SQL Server Log](#sql-server-log) to see what went wrong. This DMV is populated on the primary replica and all secondary replicas.

* sys.dm_hadr_physical_seeding_stats shows the status of the automatic seeding operation as it is executing. As with sys.dm_hadr_automatic_seeding, this will show values on both the primary and secondary replicas, but this history is not stored. The values are for the current execution only, and will not be retained. Columns of interest include start_time_utc, end_time_utc, estimate_time_complete_utc, total_disk_io_wait_time_ms, total_network_wait_time_ms, and if the seeding operation fails, failure_message.

### Backup history tables

Automatic seeding also puts entries into the `msdb` tables which store the history for backups and restores. On the secondary replica which is receiving automatic seeding, the `backupmediafamily` table's physical_device_name column has a GUID for its value, and the corresponding entry in `backupset` has the name of the primary replica for server_name and machine_name.

### Extended events

Automatic seeding adds new extended events for tracking state change, failures, and performance statistics during initialization.
For example, the following script creates an extended events session that captures events related to automatic seeding.

```
CREATE EVENT SESSION [AG_autoseed] ON SERVER 
    ADD EVENT sqlserver.hadr_automatic_seeding_state_transition,
    ADD EVENT sqlserver.hadr_automatic_seeding_timeout,
    ADD EVENT sqlserver.hadr_db_manager_seeding_request_msg,
    ADD EVENT sqlserver.hadr_physical_seeding_backup_state_change,
    ADD EVENT sqlserver.hadr_physical_seeding_failure,
    ADD EVENT sqlserver.hadr_physical_seeding_forwarder_state_change,
    ADD EVENT sqlserver.hadr_physical_seeding_forwarder_target_state_change,
    ADD EVENT sqlserver.hadr_physical_seeding_progress,
    ADD EVENT sqlserver.hadr_physical_seeding_restore_state_change,
    ADD EVENT sqlserver.hadr_physical_seeding_submit_callback
    ADD TARGET package0.event_file(SET filename=N’autoseed.xel’,max_file_size=(5),max_rollover_files=(4))
  WITH (MAX_MEMORY=4096 KB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=30 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=OFF,STARTUP_STATE=ON)
GO

ALTER EVENT SESSION AlwaysOn_autoseed ON SERVER STATE=START
GO
```

The following table lists extended events related to automatic seeding.

|Name|Description|
|----|-----------|
|hadr_db_manager_seeding_request_msg|Seeding request message.|
|hadr_physical_seeding_backup_state_change|Physical seeding backup side state change.|
|hadr_physical_seeding_restore_state_change|Physical seeding restore side state change.|
|hadr_physical_seeding_forwarder_state_change|Physical seeding forwarder side state change.|
|hadr_physical_seeding_forwarder_target_state_change|Physical seeding forwarder target side state change.|
|hadr_physical_seeding_submit_callback|Physical seeding submit callback event.|
|hadr_physical_seeding_failure|Physical seeding failure event.|
|hadr_physical_seeding_progress|Physical seeding progress event.|
|hadr_physical_seeding_schedule_long_task_failure|Physical seeding schedule long task failure event.|
|hadr_automatic_seeding_start|Occurs when an automatic seeding operation is submitted.|
|hadr_automatic_seeding_state_transition|Occurs when an automatic seeding operation changes state.|
|hadr_automatic_seeding_success|Occurs when an automatic seeding operation succeeds.|
|hadr_automatic_seeding_failure|Occurs when an automatic seeding operation fails.|
|hadr_automatic_seeding_timeout|Occurs when an automatic seeding operation times out.|

## See also

[ALTER AVAILABILITY GROUP (Transact-SQL)](/sql/t-sql/statements/alter-availability-group-transact-sql)

[CREATE AVAILABILITY GROUP (Transact-SQL)](https://msdn.microsoft.com/library/ff878399.aspx)

[Always On Availability Groups Troubleshooting and Monitoring Guide](http://technet.microsoft.com/library/dn135328.aspx)

> This content written by [Allan Hirt](http://mvp.microsoft.com/en-us/PublicProfile/4025254?fullName=Allan%20Hirt), Microsoft Most Valued Professional.

<!--Image references-->
[1]: ./media/auto-seed-new-availability-group.png
[2]: ./media/auto-seed-sql-server-log.png
