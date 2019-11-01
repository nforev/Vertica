# Fixed Bug

>**Note:**
>
>This chapter only contain the Recovery componented related JIRA with the hotfix listed in the Release Notes ***Vertica (Version): Resolved Issue*** section. 

## Veritca 8.1.x
JIRA | Description | Fix Version
:--|:--|:--
VER-50315|When recovering to a specific epoch, Vertica no longer sets the current checkpoint epoch to zero for projections created after the specified epoch.| 8.1.0
VER-53793|If Vertica attempted to recover a table with out-of-date projections after a partition swapping operation, Vertica incorrectly dropped ROSes in the swapped partition, causing projection row count mismatch.|8.1.0-2
VER-54522|During recovery, if acquiring a global catalog lock failed on some nodes, the nodes sometimes panicked and aborted recovery.|8.1.0-4
VER-54960|The initial phase of a recovery by table is no longer slow in instances with thousands of tables. This issue had also affected other concurrent cluster operation for a brief period of time.|8.1.0-5
VER-53425|If Vertica attempted to recover a table with out-of-date projections after a partition swapping operation, Vertica incorrectly dropped ROSes in the swapped partition, causing projection row count mismatch.|
VER-51674|The initial phase of a recovery by table is no longer slow in instances with thousands of tables. This issue had also affected other concurrent cluster operation for a brief period of time.|8.1.1
VER-58590|Recovering a node from Scratch was restricted to a single threaded storage transfer, this has been fixed to use the recovery resource pool maxconcurrency.|8.1.1-9
VER-57777|Previously, all live aggregate projections were marked out-of-date when the cluster underwent an administrator-specified recovery. This issue has been resolved so that only live aggregate projections whose anchor tables' data was modified since the recovery epoch would be marked out-of-date during recovery.|8.1.1-9
VER-59755|If you turn off RecoverByContainer, LAP may be skipped to recover data on a recovered node. This issue has been fixed by preventing the node from recovering if it detects that LAP cannot perform a recover by container until you turn on RecoverByContainer.|8.1.1-11


## Vertica 9.0.x
JIRA|Description|Fix Version
:--|:--|:--
VER-55139|Previously, all live aggregate projections were marked out-of-date when the cluster underwent an administrator-specified recovery. This issue has been resolved so that only live aggregate projections whose anchor tables' data was modified since the recovery epoch would be marked out-of-date during recovery.|9.0.0
VER-56802|After recovering a database where recovery by table was disabled, a placeholder Tuple Mover marker inside system table V_MONITOR.TUPLE_MOVER_OPERATIONS was not always cleaned up. This prevented other Tuple Mover operations from executing. This problem has been resolved.|9.0.0
VER-58591|Recovering a node from Scratch was restricted to a single threaded storage transfer, this has been fixed to use the recovery resource pool maxconcurrency.|9.0.0-2
VER-59754|If you turn off RecoverByContainer, LAP may be skipped to recover data on a recovered node. This issue has been fixed by preventing the node from recovering if it detects that LAP cannot perform a recover by container until you turn on RecoverByContainer.|9.0.1-2

## Vertica 9.1.x
JIRA|Description|Fix Version
:--|:--|:--



## Vertica 9.2.x
JIRA|Description|Fix Version
:--|:--|:--
