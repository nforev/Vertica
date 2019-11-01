# Recovery Concept
## 1. Failure Recovery
Vertica can restore the database to a fully functional state after one or more nodes in the system experiences a software- or hardware-related failure. Vertica recovers nodes by querying replicas of the data stored on other nodes. For example, a hardware failure can cause a node to lose database objects or to miss changes made to the database (INSERTs, UPDATEs, and so on) while offline. When the node comes back online, queries other nodes in the cluster to recover lost objects and catch up with database changes.

K‑safety sets fault tolerance for the database cluster, where K can be set to 0, 1, or 2. The value of K specifies how many copies Vertica creates of segmented projection data. If K‑safety for a database is set to 1 or 2, Vertica creates K+1 instances, or buddies, of each projection segment. Vertica distributes these buddies across the database cluster, such that projection data is protected in the event of node failure. If any node fails, the database can continue to process queries so long as buddies of data on the failed node remain available elsewhere on the cluster.

>**Notes:**
>
>You can monitor the cluster state through the **View Database Cluster** state menu option.

### Recovery Scenarios
Vertica begins the database recovery process when you restart failed nodes or the database. The mode of recovery for a K-safe database depends on the type of failure:
+ One or more nodes in the database failed, but the database continued to operate.
+ The database shut down cleanly.
+ The database shut down uncleanly.

In the first two cases, node recovery is automatic; in the third case (unclean shutdown), recovery requires manual intervention by the database administrator. The following sections discuss these cases in greater detail.

**Recovery of failed nodes** 

One or more nodes failed but the remaining nodes in the database filled in for them, so database operations continued without interruption. Use Administration Tools to restart failed nodes through the Restart Vertica on Host option. While restarted nodes recover their data from other nodes, their status is set to RECOVERING. Except for a short period at the end, the recovery phase has no effect on database transaction processing. After recovery is complete, the restarted nodes status changes to UP.

**Recovery after clean shutdown**

The database was shut down cleanly through Administration Tools. To restart the database, use the Start Database option. On restart, all nodes whose status was UP before the shutdown resume a status of UP. If the database contained one or more failed nodes on shutdown and they are now available, they begin the recovery process as described above.

**Recovery after unclean shutdown**

Reasons for unclean shutdown include:
+ A critical node failed, leaving part of the database's data unavailable.
+ A site-wide event such as a power failure causes all nodes to reboot.
+ Vertica processes on the nodes exited due to a software or hardware failure.

Unclean shutdown can put the database in an inconsistent state—for example, Vertica might have been in the middle of writing data from WOS to disk at the time of failure, and this process was left incomplete. When you restart the database through the Administration Tools, Vertica determines that normal startup is not possible and uses the Last Good Epoch to determine when data was last consistent on all nodes. When you restart the database, Vertica prompts you to accept recovery with the suggested epoch. If accepted, the database recovers and all data changes after the Last Good Epoch are lost. If not accepted, startup is aborted.

Instead of accepting the recommended epoch, you can recover from a backup. You can also choose an epoch that precedes the Last Good Epoch, through the Administration Tools Advanced Menu option Roll Back Database to Last Good Epoch. This is useful in special situations—for example the failure occurs during a batch of loads, where it is easier to restart the entire batch, even though some of the work must be repeated. In most cases, you should accept the recommended epoch.

### Epochs and Node Recovery

The checkpoint epoch (CPE) for both the source and target projections are updated as ROS containers are moved. The start and end epochs of all storage containers, such as ROS containers, are modified to the commit epoch. When this occurs, the epochs of all columns without an actual data file rewrite advance the CPE to the commit epoch of move_partitions_to_table. If any nodes are down during the TM moveout and/or during the move partition, they will detect that there is storage to recover, and will recover from other nodes with the correct epoch upon rejoining the cluster.

### Manual Recovery Notes
+ You can manually recover a database where up to K nodes are offline—for example, they were physically removed for repair or not reachable at the time of recovery. When the missing nodes are restored, they recover and rejoin the cluster as described earlier in Recovery Scenarios.
+ You can manually recover a database if the nodes to be restarted can supply all partition segments, even if more than K nodes remain down at startup. In this case, all data is available from the remaining cluster nodes, so the database can successfully start.
+ The default setting for the HistoryRetentionTime configuration parameter is 0, so Vertica only keeps historical data when nodes are down. This setting prevents use of the Administration Tools Roll Back Database to Last Good Epoch option because the AHM remains close to the current epoch and a rollback is not permitted to an epoch that precedes the AHM. If you rely on the Roll Back option to remove recently loaded data, consider setting a day-wide window to remove loaded data—for example:
```
=> ALTER DATABASE mydb SET HistoryRetentionTime = 86400;
```
See Epoch Management Parameters in the [Administrator's Guide](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/AdministratorsGuide/_Intro/AdministratorsGuide.htm).

+ When a node is down and manual recovery is required, it can take a full minute or longer for Vertica processes to time out while the system tries to form a cluster. Wait approximately one minute until the system returns the manual recovery prompt. Do not press CTRL-C during database startup.

## 2.Recovering the Cluster From a Backup

### Backing Up and Restoring the Database
Creating regular database backups is an important part of basic maintenance tasks. Vertica supplies a comprehensive utility, vbr, for this purpose. vbr lets you perform the following operations. Unless otherwise noted, operations are supported in both Enterprise Mode and Eon Mode:

+ Back up a database.
+ Back up specific objects (schemas or tables) in a database (Enterprise Mode only).
+ Restore a database or individual objects from backup.
+ Copy a database to another cluster (for example, to promote a test cluster to production) (Enterprise Mode only).
+ Replicate individual objects (schemas or tables) to another cluster (Enterprise Mode only).
+ List available backups.

When you run vbr you specify a configuration (.ini) file. In this file you specify all of the configuration parameters for the operation -- what to back up, where to back it up, how many backups to keep, whether to encrypt transmissions, and much more. Vertica provides several [Sample VBR .ini Files](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/AdministratorsGuide/BackupRestore/SampleConfigFiles/SampleIniFiles.htm) that you can use as templates.

You can use vbr to restore a backup created by vbr. Typically you use the same configuration file for both operations.

When doing a backup you can save your data to a local directory on each node, a remote file system, a different Vertica cluster (effectively cloning your database), or S3. If you are backing up an Eon Mode database you must use S3.

You cannot back up an Enterprise Mode database and restore it in Eon Mode or vice versa.

Common Use Cases introduces the most common vbr operations.

For more details about Backup and Restore, you could check [Backing Up and Restoring the Database](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/AdministratorsGuide/BackupRestore/BackingUpAndRestoringTheDatabase.htm).

## 3.Disaster Recovery
To protect your database from site failures caused by catastrophic disasters, maintain an off-site replica of your database to provide a standby. In case of disaster, you can switch database users over to the standby database. The amount of data loss between a disaster and fail over to the offsite replica depends on how frequently you save a full database backup.

The solution to employ for disaster recover depends upon two factors that you must determine for your application:

+ **Recovery point objective (RPO)**: How much data loss can your organization tolerate upon a disaster recovery?
+ **Recovery time objective (RTO)**: How quickly do you need to recover the database following a disaster?

Depending on your RPO and RTO, Vertica recommends choosing from the following solutions:

1. **Dual-load**: During each load process for the database, simultaneously load a second database. You can achieve this easily with off-the-shelf ETL software.
2. **Periodic Incremental Backups**: Use the procedure described in Copying the Database to Another Cluster to periodically copy the data to the target database. Remember that the script copies only files that have changed.
3. **Replication solutions provided by Storage Vendors**: Although some users have had success with SAN storage, the number of vendors and possible configurations prevent Vertica from providing support for SANs.



## 4.Recovery By Table
Vertica supports node recovery on a per-table basis. Unlike node-based recovery, recovering by table makes tables available as they recover, before the node itself is completely restored. You can [prioritize your most important tables](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/AdministratorsGuide/Recovery/PrioritizingTableRecovery.htm) so they become available as soon as possible. Recovered tables support all DDL and DML operations.

To enhance recovery speed, Vertica recovers multiple tables in parallel. The maximum number of tables recoverable at one time is set by the MAXCONCURRENCY parameter in the [RECOVERY resource pool](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/SQLReferenceManual/Statements/Built-inPoolConfiguration.htm#RECOVERY).

After a node has fully recovered, it enables full Vertica functionality.

### Prioritizing Table Recovery
You can specify the order in which Vertica recovers tables. This feature ensures that your most critical tables become available as soon as possible. To specify the recovery order of your tables, assign an integer priority value. Tables with higher priority values recover first. For example, a table with a priority of 1000 is recovered before a table with a value of 500. Table priorities have the maximum value of a 64-bit integer.

If you do not assign a priority, or if multiple tables have the same priority, Vertica restores tables by OID order. Assign a priority with a query such as this:

```
=> SELECT set_table_recover_priority('avro_basic', '1000');
      set_table_recover_priority
---------------------------------------
 Table recovery priority has been set.
(1 row)
```

View assigned priorities with a query using this form:

```
SELECT table_name,recover_priority FROM v_catalog.tables;
```

The next example shows prioritized tables from the VMart sample database. In this case, the table with the highest recovery priorities are listed first (DESC). The shipping_dimension table has the highest priority and will be recovered first. (Example has hard Returns for display purposes.)

```
=> SELECT table_name AS Name, recover_priority from v_catalog.tables WHERE recover_priority > 1 
   ORDER BY recover_priority DESC;
        Name         | recover_priority
---------------------+------------------
 shipping_dimension  |            60000
 warehouse_dimension |            50000
 employee_dimension  |            40000
 vendor_dimension    |            30000
 date_dimension      |            20000
 promotion_dimension |            10000
 iris2               |             9999
 product_dimension   |               10
 customer_dimension  |               10
(9 rows)
```

### Viewing Table Recovery Status

View general information about a recovery querying the V_MONITOR.[TABLE_RECOVERY_STATUS](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/SQLReferenceManual/SystemTables/MONITOR/TABLE_RECOVERY_STATUS.htm) table. You can also view detailed information about the status of the recovery the table being restored by querying the V_MONITOR.[TABLE_RECOVERIES](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/SQLReferenceManual/SystemTables/MONITOR/TABLE_RECOVERIES.htm) table.
