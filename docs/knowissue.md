# Known Issue
>**Note:**
>
>This chapter only contain the Recovery componented related JIRA with the hotfix listed in the Release Notes ***Known issues Vertica (version)*** section. 

JIRA|Description|Version|Fixed in
:--|:--|:--|:--
VER-58168|A transaction that started before a node began recovery is referred to as a dirty transaction in the context of recovery. A recovering node must wait for or cancel such a transaction in order to recover tables modified by the transaction. In rare instances such transactions may be hung and can not be canceled. Tables locked by such a transaction can not be recovered and therefore a recovering node can not transition to 'UP'. Usually you can stop the hung transaction by restarting the node on which the transaction was initiated, assuming this is not a critical node. In extreme cases or instances where the initiator node is a critical node the cluster must be restarted.|9.1.x|Not Fixed yet
VER-57752|If all partitions are moved from a table while a node is down and then a new projection is created on that table, the epoch for the new projection will be more recent than the epoch for the MOVE PARTITION event. In that rare case, the down node fails to recover. This problem also occurs if, when the node is down, Vertica swaps out all partitions, but no partitions have been swapped in. **Workaround**: Find the newly created projection in the PROJECTION_CHECKPOINT_EPOCHS system table. Drop that projection and restart the down node.|9.0.x|9.0SP1
