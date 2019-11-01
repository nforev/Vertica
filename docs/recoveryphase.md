# Recovery Phase

## Node Recovery Process

Vertica performs node recovery in two phases:
+ **Pre-recovery phase**: In this phase, the recovering node performs the following tasks:
    + Restarts spread daemon
    + Reads its catalog on disk
    + Validates data storage
    + Joins the cluster 

	
+ **Recovery phase**: In this phase, the recovering node:
    + Receives new data loaded into the cluster
    + Recovers data that it missed when the node was in DOWN state before moving to the UP state

If a database node is in the RECOVERING state, Tuple Mover can perform moveout and mergeout operations as needed during the recovery. Tuple Mover performs mergeout during recovery, merging ROS containers and preventing ROS pushback. In addition, Tuple Mover performs moveout, moving data from the WOS to the ROS. Doing so allows tasks like trickle-loading to take place during recovery, creating a sustainable load process that generates fewer ROS containers. Both mergeout and moveout can improve the performance of your node recovery.

The following image gives a detailed workflow that shows all the stages of the node recovery process:
	
![How recovery works](https://www.vertica.com/kb/Node-Recovery-in-Vertica/Content/Resources/Images/Node-Recovery-in-Vertica/noderecoveryprocess.png)

### Pre-Recovery Phase
The pre-recovery phase starts immediately after the node restarts and runs until the node rejoins the cluster. To track the node progress during this phase, use the startup.log file located in the catalog directory. The startup.log file is a subset of the vertica.log file.

**The pre-recovery phase has following stages:**

**a. Reads the catalog**:
The recovering node reads the catalog checkpoints, applies the transaction logs, and indexes the catalog objects.

**b. Starts and connects to the spread deamon**:
Vertica starts the spread daemon on the recovering node and connects to spread daemon.

However, if your database runs in [large cluster mode](https://vertica.com/docs/latest/HTML/index.htm#Authoring/AdministratorsGuide/ClusterManagement/LargeCluster/InstallingALargeCluster.htm), a subset of nodes known as control nodes run the spread daemon instead.

**c. Reads the DataCollector files**:
The recovering node creates an inventory of the DataCollector files.

**d. Checks data storage**:
+ The recovering node checks the data files specified in the catalog. It verifies the files for existence, permission, and correctness.
+ The recovering node removes the data files that are not referenced in the catalog.

**e. Loads UDx libraries**:
The recovering node loads the User Defined Extension (UDx) libraries.

**f. Prepares for Cluster Invite**:
The recovering node joins the spread group and broadcasts messages to the other nodes. The recovering node waits for an invitation from the other nodes to rejoin the cluster.

**g. Joins the cluster**:
Upon invitation, the node joins the cluster and the node state changes to the INITIALIZING state.

**h. Receives and install the new global catalog**:
The recovering node shares the catalog version with other nodes in the cluster.

If the catalog version of the recovering nodes is lower than the nodes in the UP state, a node in the UP state sends the global catalog to the recovering node in the chunks of 1 GB.

The recovering node installs the new catalog received from one of the UP nodes.


### Recovery Phase
In the recovery phase, the recovering node receives new data loaded in the database. The recovering node also recovers data that it missed while in the DOWN state until this point from the recovering node’s buddy nodes.

You can track the progress of this phase using the following system tables:
+ TABLE_RECOVERY_STATUS
+ TABLE_RECOVERIES
+ PROJECTION_RECOVERIES

**The recovery phase has following stages:**
**a. Checks and replays missing catalog events**:
The recovering node checks for missing events and replays the missed events in the order in which they were committed.

The Vertica catalog has a log of the following types of catalog events, the tables involved, and the epochs in which the events were committed.

+ Alter partition
+ Restore table
+ Replace node

**b. Marks dirty transactions**:
Dirty transactions are uncommitted transactions that start before the beginning of the RECOVERY phase.

During recovery, Vertica marks the dirty transactions, and the state of the recovering node state changes from INITIALIZING to RECOVERING. The node in RECOVERING state participates in all data load plans.

**c. Loads UDx libraries**:
The recovering node loads the UDx libraries received from the nodes in the UP state.
**d. Makes list of tables that need recovery**:
The recovering node makes a list of tables that need recovery.
**e. Table Recovery**:
For step-by-step details about table recovery, see Steps for Table Recovery.

As of Vertica 7.2.2, Vertica recovers more than one table concurrently. The number of tables that Vertica can recovery concurrently depends on:

The MAXCONCURRENCY of the resource pool RECOVERY
The number of projections per table
As of Vertica 7.2.x, you can specify the order of table recovery. For more information, see Prioritizing Table Recovery in Vertica documentation.
**f. Check table list and move to READY state**:
If the list of tables that need recovery is now empty, Vertica changes the node state from RECOVERING to READY.

If the list is not empty, Vertica retries the steps for recovery up to 20 times. If these repeated attempts fail, node recovery fails and the node state changes to DOWN.
**g. Move to UP state**:
In the final step of the recovery phase, Vertica changes the state of the node from READY to UP. From this point, the recovering node accepts new connections and participates in all database plans.

## Phases of a Recovery
The phases of a Vertica recovery are the same regardless of whether you are recovering by table or node. In the case of a [recovery by table](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/AdministratorsGuide/Recovery/RecoveringByTable.htm), tables become individually available as they complete the final phase. In the case of a recovery by node, the database objects only become available after the entire node completes recovery.

When you perform a recovery in Vertica, each recovered table goes through the following phases:

Order|Phase|Description|Lock Type
:--|:--|:--|:--
1|Historical|Vertica copies any historical data it may have missed while in a state of DOWN or INITIALIZING.|none
2|Historical Dirty|Vertica recovers any DML transactions that committed after the node or table began recovery.|none
3|Current Replay Delete|Vertica replays any delete transactions that took place during the recovery.|T-lock
4|Aggregate Projections|Vertica recovers any aggregate projections.|T-lock

### Steps for Table Recovery
1. **Replay catalog events**: The recovering node replays catalog events that the recovering table missed while the node was in DOWN state. The recovering node checks the following catalog events for the recovering table:
    + Truncate table
    + Add column
    + Rebalance table
    + Alter partition
    + Drop partition
    + Restore table
    + Move partition
    + Swap partition
    + Database rollback
    + Replace node
    + Merge projection

2. **Historical recovery phase**: For each projection, Vertica retains the Checkpoint Epoch for each node. The Checkpoint Epoch represents a point in time up to which all the data was stored on disk.
The projections anchored on the recovering table recover the historical data that the table missed. During this stage, Vertica recovers data from projection Checkpoint Epoch to the Current Epoch. There are three methods for data recovery. 
If the Checkpoint Epoch of the projection is 0, the node recovers the projection data using the recovery-by-container method. In the recovery-by-container method, the recovering node copies the storage containers and delete vectors from the buddy nodes. If the Checkpoint Epoch of the projection is not 0, Vertica uses the incremental method to recover the projection data. With the incremental method, Vertica first scans the storage containers of the buddy projection to fetch the needed projection data. Then, the projection data are sorted on demand and written to disk on the recovering node. The incremental method only recovers storage containers. Hence, if there are any delete vectors committed when the recovering node is down, an incremental-replay-delete method is used to recover the delete vectors by following the incremental method. Vertica does not acquire any locks in the historical recovery stage.

3. **Recover dirty transactions phase**: Use the RecoveryDirtyTxnWait configuration parameter to control the amount of time the recovering node waits for the dirty transactions to be committed.
    + Sometimes the recovering node finds uncommitted dirty transactions. In such cases, by default the node waits five minutes for those transactions to be committed. After five minutes, the Vertica recovery process terminates all the uncommitted dirty transactions.
    + After the dirty transactions commit, the recovering node will recover the data loaded through dirty transactions. Vertica can use the incremental and incremental-replay-delete methods for data recovery in this phase. No lock is acquired at the dirty transaction phase.
	
4. **Replay delete phase**: Deletes could occur during node recovery, and steps 2 and 3 do not take locks. However, delete operations started after the node enters the RECOVERING state do not create delete vectors on the recovering node. Vertica recovers those missed delete vectors in the replay delete phase. At this stage, Vertica takes a T-lock on the recovering tables and uses the incremental-replay-delete method to recover the missed delete vectors. At this stage, the recovering node also determines if the tables have any discrepancies in the catalog events from the last replay of catalog events. If the recovering node finds a missed event, the recovery for this table fails and table is marked as "Failed to recover". Then, the recovering node will retry the entire table recovery process from step 1 on the failing tables. Vertica allows a maximum of 20 recovery attempts before failing the node recovery.

5. **Finish Table Recovery**: Upon successful completion of the above steps, Vertica marks the table as recovered and moves the table off the list of tables that need to be recovered. From this point, the recovered table participates in all DDL and DML plans.


## Recovery in Vertica.log
