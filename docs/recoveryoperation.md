# Recovery Operation
## 1. Monitoring Recovery
When you have a node in the DOWN state, to monitor the recovery process, follow these steps:
Identify the state of the recovering node by running the following command:
```
$ /opt/vertica/bin/admintools -t view_cluster
     DB    |     Host     |   State 
-----------+--------------+------------ 
mydatabase | 112.17.31.10 | UP
mydatabase | 112.17.31.11 | UP
mydatabase | 112.17.31.12 | INITIALIZING
mydatabase | 112.17.31.17 | UP 
```
If the node state is DOWN or INITIALIZING, monitor the progress of the recovering node using tail startup.log:
```
$ tail catalog-path/database-name/v_database-name_node_catalog/startup.log  
```
During the node recovery, Vertica updates the recovering node's startup.log. These updates are in the form of JSON information blocks that show the following:

```
{
"goal" : 477688606,
"node" : "v_newdb2_node0003",
"progress" : 146732903,
"stage" : "Read DataCollector",
"text" : "Inventory files (bytes)",
"timestamp" : "2016-03-16 17:28:32.016"
			}
```
Note that the recovery status about a node in the system tables TABLE_RECOVERY_STATUS, TABLE_RECOVERIES, and PROJECTION_RECOVERIES is not available when the state of that node is DOWN or INITIALIZING.
When the node state is RECOVERING, track the progress of the node using system tables:
To view the summarized status, run the following query:
```
=> SELECT * FROM TABLE_RECOVERY_STATUS;
```

To view tables and projections recovery details, run the following queries:
```
=> SELECT * FROM TABLE_RECOVERIES;
=> SELECT * FROM PROJECTION_RECOVERIES;
```

For example, if the recovery-by-container method is used, you are expected to see the following content in the recovery system tables during the historical recovery phase. The details about the recovery phase and recovery methods are shown in the system tables TABLE_RECOVERIES and PROJECTION_RECOVERIES, respectively.

## 2. Troubleshooting the Recovery
The following list describes the most common troubleshooting use cases and recommendations for resolving the issues:

1.**Spread daemon fails to start and connect to Vertica**.

**Description**: If the spread daemon failed to start, the node recovery fails and the recovering node continues to remain in the DOWN state.

**Action**: To identify the problem use the startup.log file or the vertica.log file of the recovering node. Some possible resolutions are as follows:

+ Verify that the spread.conf file in the catalog directory of the recovering node is identical to all spread.conf files + on the other nodes in the cluster.
+ Verify that the IP address of the recovering node is appropriate in the spread.conf file in the catalog directory.
Remove the file 4803 from the temporary (/tmp) directory and restart the node.

2.**The recovering node spends an unusually long time in the “check data storage” stage of the pre-recovery phase.**

**Description**: The recovering node can spend an unusually long time in the pre-recovery phase if:

+ The catalog is large in size and has millions of ROS files.

+ The recovering node has to remove large number of ROS files that are not referenced by the catalog.

**Action**: To monitor the progress of the recovery node, use the startup.log file.

3.**The recovering node fails with an error.**

**Description**: The error message reads “Data consistency problems found; startup aborted”.
**Action**: To resolve this problem, restart the node from admintools CLI with the --force flag:

```
$ /opt/vertica/bin/admintools -t restart_node -d <database_name> --hosts <ip address> --force
```

4.**The recovering node is waiting for an invitation to join the cluster.**

**Description**: The recovering node cannot communicate with other nodes in the cluster and is waiting for an invitation from other nodes to join the cluster in the following situations:

+ The firewall is enabled or the port is blocked.
+ The recovering node belongs to different subnet.

**Action**: To resolve these issues, perform the following tasks:

+ Verify the communication with other nodes in the cluster using the netcat [$ nc]
+ If the recovering node belongs to a different subnet than the other nodes in the cluster, configure spread in a point-to-point mode.

5.**The table recovery fails multiple times because of catalog events.**

**Description**: Performing catalog events such as TRUNCATE TABLE or DROP PARTITION on a currently recovering table may result in table recovery failure.
If the table fails to recover after 20 attempts, Vertica recovery fails. Multiple failed recovery attempts can delay node recovery.
**Action**: To resolve this issue, perform the following tasks:

+ To track the recovery attempts, use the following command:

```
$ grep “incrCatchUpFailureCount” vertica.log
```

For example, the following message suggests, Vertica made 17 attempts to recover the node.

```
[Recover] <INFO> incrCatchUpFailureCount: 17 failures, max 20
```

+ If you notice multiple retry attempts from the above step, stop the ETL processes that aggressively performs catalog events.

6.**The UPDATE and DELETE statements run for an unusually longer time.**

**Description**: If the recovering node cannot acquire T lock on the recovering table for more than five minutes, the table recovery fails. Five minutes is a default setting of the LOCKTIMEOUT configuration parameter.
If the table fails to recover after 20 attempts, Vertica recovery fails. Multiple failed recovery attempts can delay node recovery.
**Action**: To resolve the issue, perform the following tasks:

+ To track the recovery attempts, use the following command:

```
$ grep “incrCatchUpFailureCount” vertica.log
```

+ Cancel the long-running DML operations.