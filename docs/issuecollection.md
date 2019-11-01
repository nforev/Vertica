# Recovery Issue Collection

## 1. Startup Failed with error "ASR Required"
If the startup failed and got 'Startup Failed, ASR Required' or 'Manual recovery possible: Last good epoch=0x3117a1' message from vertica.log or startup.log, we need manual intervention to choose the right LGE
1. Then use Admintools, Rollback to Last Good Epoch
2. admintools -t restart_db -d [database name] –p [password] -e last

Below is an example, we need choose the smallest epoch inside 'Filling more nodes to satisfy node dependencies' which is 117835:
   
   
**Example 1:**
```
   [Init] <INFO> Startup [Startup Failed, ASR Required] Node Dependencies:

   Nodes certainly in the cluster:
   	Node 0(v_hljdb_node0001), epoch 117863
   	Node 1(v_hljdb_node0002), epoch 117863
   	Node 2(v_hljdb_node0003), epoch 117863
   	Node 3(v_hljdb_node0004), epoch 117863
   	Node 14(v_hljdb_node0015), epoch 117863
   	Node 15(v_hljdb_node0016), epoch 117863
   	Node 16(v_hljdb_node0017), epoch 117863
   	Node 17(v_hljdb_node0018), epoch 117863
   	Node 18(v_hljdb_node0019), epoch 117863
   	Node 19(v_hljdb_node0020), epoch 117863
   	Node 20(v_hljdb_node0021), epoch 117863
   	Node 21(v_hljdb_node0022), epoch 117863
   	Node 22(v_hljdb_node0023), epoch 117863
   	Node 23(v_hljdb_node0024), epoch 117863
   	Node 24(v_hljdb_node0025), epoch 117863
   	Node 25(v_hljdb_node0026), epoch 117863
   	Node 26(v_hljdb_node0027), epoch 117863
   	Node 27(v_hljdb_node0028), epoch 117863
   	Node 28(v_hljdb_node0029), epoch 117863
   Filling more nodes to satisfy node dependencies:
   	Node 29(v_hljdb_node0030), epoch 117863
   	Node 30(v_hljdb_node0031), epoch 117863
   	Node 31(v_hljdb_node0032), epoch 117863
   	Node 32(v_hljdb_node0033), epoch 117863
   	Node 33(v_hljdb_node0034), epoch 117863
   	Node 34(v_hljdb_node0035), epoch 117863
   	Node 35(v_hljdb_node0036), epoch 117863
   	Node 10(v_hljdb_node0011), epoch 117841
   	Node 12(v_hljdb_node0013), epoch 117841
   	Node 7(v_hljdb_node0008), epoch 117839
   	Node 13(v_hljdb_node0014), epoch 117839
   	Node 6(v_hljdb_node0007), epoch 117837
   	Node 9(v_hljdb_node0010), epoch 117837
   	Node 4(v_hljdb_node0005), epoch 117835
   Data dependencies fulfilled, remaining nodes LGEs don't matter:
   	Node 5(v_hljdb_node0006), epoch 117835
   	Node 8(v_hljdb_node0009), epoch 117831
   	Node 11(v_hljdb_node0012), epoch 117831
   --
```

**Example 2:**   
check the vertica.log with key words 'My local node LGE =' to find the local LGE from each node, and choose the second smallest global LGE to recover.

[Case Example](/caseexample)

## 2. Recovery failed due to corrupted data
In some cases cluster encounter unexpect shutdown, the Vertica catalog may reference a data file that is either missing or corrupted.  In the above situations, the nodes computed recovery epoch may be behind ancient history mark (AHM). If two of the buddy nodes (nodes that share a segment) have the above problem, the database can’t be started in the standard way.  

To recover the database in this situation follow the steps below:

>**Best Option: Restore from backup**
>
>Ask customer if they have recent database backup, restoring from a backup is safest and best option.   If they do not have a backup to restore from, when the database comes back up, take a couple of minutes to suggest a backup method, such as using AWS s3.
 
**Step 1: Start database in unsafe mode**

Start database in unsafe mode to identify impacted projections. 

```
$ admintools -t start_db -d <db1> -U
```

This mode should only be used under supervision of technical support engineer to help recover database that is facing one of the above issues. 

**Step 2:  Identify AHM and nodes required for data safety**

Run following API’s to identify AHM and the nodes required to meet data safety and what their node last good epoch (LGE) is.

```
SELECT get_ahm_epoch();
SELECT get_expected_recovery_epoch();
```

Following is the output of get_ahm_epoch and get_expected_node_recovery from my 5 node cluster.  AHM epoch is 150 and expected recovery epoch is 149.  You can see node0001, node0002 and node0003 has node LGE of 170 and we have to fill in node004 to satisfy node dependency.  Node0004 has node LGE of 149 and node0005 has node LGE of 100.  We need to identify projections that are pulling node LGE of node0004 and node0005 behind AHM epoch.   

```
dbadmin=> select get_ahm_epoch();
 get_ahm_epoch
---------------
           150
(1 row)

dbadmin=> select get_expected_recovery_epoch();
INFO 4544:  Recovery Epoch Computation:
Node Dependencies:
00011 - cnt: 10
00110 - cnt: 10
01100 - cnt: 10
11000 - cnt: 10
10001 - cnt: 10
11111 - cnt: 10

00001 - name: v_vmart_node0001
00010 - name: v_vmart_node0002
00100 - name: v_vmart_node0003
01000 - name: v_vmart_node0004
10000 - name: v_vmart_node0005

Nodes certainly in the cluster:
        Node 0(v_vmart_node0001), epoch 170
        Node 1(v_vmart_node0002), epoch 170
        Node 2(v_vmart_node0003), epoch 170
Filling more nodes to satisfy node dependencies:
        Node 3(v_vmart_node0004), epoch 149
Data dependencies fulfilled, remaining nodes LGEs don't matter:
        Node 4(v_vmart_node0005), epoch 100
--
 get_expected_recovery_epoch
-----------------------------
                         149
(1 row)
```

The nodes identified in the outlined section above are the nodes we need to be concerned with, the other node(s) can be ignored. In our example above these are; v_vmart_node0001, v_vmart_node0002, v_vmart_node0003, and v_vmart_node0004.

**Step 3: Identify impacted projections**

Run the following query to identify nodes with projections that have checkpoint epochs (CPE) lower than AHM epoch. Minimum value of projection CPE on a node is Node LGE of that node. 

```
SELECT e.node_name, t.table_schema, t.table_name, e.projection_schema, e.projection_name, checkpoint_epoch FROM projection_checkpoint_epochs e, projections p, tables t WHERE e.projection_id = p.projection_id and p.anchor_table_id = t.table_id and not (is_temp_table) and is_behind_ahm and node_name in (<list of nodes identified in step 3>); 
```

**Step 4:  Running abortrecovery**

You need to run the following command for each table identified in step 4. When you run abortrecovery, you are updating CPE of projection to value of “-1”, which means no recovery will take place when you restart database.  

```
SELECT do_tm_task('abortrecovery','<schema.tbl_name>');
```

**Step 5: Restarting database**

Stop and restart database from admintools. At this point database will be up, but tables that were marked with abortrecovery will have data inconsistencies between projections. Please follow next few steps to clean data inconsistencies once database is started before opening it to general usage.

**Step 6: Clean up**

When database is started, the tables that were marked with abortrecovery will need to be fixed.  The DBA will need to determine if the table from the list can be rebuilt (data fully truncated) or if some of the data needs to be salvaged due to critical business need. 

Tables that can be truncated should be truncated using TRUNCATE TABLE command. 


### Critical table data salvage

For each business critical table that was marked with abortrecovery, you should pick one of the following options to fix data integrity issues:

**Partitioned table:**

1. Identify partitions that have count mismatch between two buddy projections by running following query.

```
SELECT partition_key, diff FROM 
(SELECT a.partition_key,sum(a.ros_row_count - a.deleted_row_count) - sum(b.ros_row_count - b.deleted_row_count) as diff FROM partitions a JOIN partitions b ON a.partition_key = b.partition_key  WHERE a.projection_name in (<projection name and buddy projection name>)  group by 1 ) as sub WHERE diff <> 0;
```

2. For partitions with count mismatch between buddy projections, you can drop partition and reload or move impacted partition to new table using move_partitions_to_table and then insert select data from new table to original table. Impacted partition will have some data loss here but end result is no count mismatch between two buddy projections of table.

3. Once you have fixed all impacted partitions in a table, above query should return an empty result set.

4. Run following query to leave comment for future reference:

```
COMMENT ON TABLE <schema_name.table name> is 'abortrecovery was run by <TSE> on <date>';
```

**Non-partitioned table:**

1.	Create a new table and insert select data from impacted fact table into new table.  The new table may have missing data, but there is no count mismatch between two buddy projections of table.
2.	Drop impacted table and rename new table back to original name.


## 3. Recovery takes too long time

### How to check recovery status

### Scenario 1: Tuning RECOVERY resource pool

In some cases, large database that contains a single large table with two projections, and with default settings, recovery is taking too long. You want to give recovery more memory to improve speed.

Set PLANNEDCONCURRENCY and MAXCONCURRENCY in the RECOVERY pool to 1, so recovery can take as much memory as possible from the GENERAL pool and run only one thread at once.

>**Caution:** This setting can slow down other queries in your system.

### Scenario 2: Recovery blocked by lock

Recovery 

### Scenario 3: Replay delete

http://confluence.verticacorp.com/pages/viewpage.action?pageId=5309038

