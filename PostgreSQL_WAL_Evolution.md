![wal](http://movead.gitee.io/picture/wal_train/walhistory/wal.jpg)

# PostgreSQL WAL Evolution

WAL is what an important part of PostgreSQL, it activity in all kinds of PostgreSQL feature and almosts all database behaive will record in to wal. Hense we can regard wal as a change roadmap of the history of PostgreSQL database, and also wal plays an important part in crash recovery, HA, replication and logical replication. Image below show GUC arguments related to wal(base on PG12). It makes sence to make clear every GUC argument, so to configure our database in high performance and reliably. We can find them on official document, but a little hard to really undersand what every GUC means by several lines describe, or maybe you find how some GUCs does't support in your PostgreSQL version. This blog I will research from a clear version of wal implement which is easy to handle and redo it until the complex wal implement.

<img src="http://movead.gitee.io/picture/wal_train/walhistory/wal12.jpg">





## WAL Genesis(7.1)

I will not to describe the POSTGRES project of  Berkeley again, for my topic today it is prehistoric civilization which is hardly to search. My foot is set on PostgreSQl 7.1 which complete and easy enough. Let us look at wal on PostgreSQL7.1 below, so clearly and we can hold every thing? 

![wal7.1](http://movead.gitee.io/picture/wal_train/walhistory/wal7.1.jpg)

Here is the wal's nomally job only, WAL's central concept is that changes to data files must be written only after those changes have been logged - that is, when log records have been flushed to permanent storage.  In this way, we don't need to refresh the data files in the shared cache to the disk at any time, because if the database crashes, we can get the data not written to the disk in the shared cache from the wal log. The purpose of this method is to replace random data writing with sequential writing wal to obtain higher execution efficiency.



**"*log records have been flushed to permanent storage*", there are similar words in the general introduction of the wal log, it seems easy but PostgreSQL do dozen things to implement the several words in purpose of performance, and look at [guc1] in the figure above some GUC related.**



#### WAL_BUFFERS

Every PostgreSQL backend may produce wal record, the records are written into PostgreSQL share buffer first and 'wal_buffers' GUC define the size of share buffer for wal.



#### FSYNC

PostgreSQL's data cache is written to the operating system cache before it is written to the persistent storage. PostgreSQL writes the data to the operating system cache by default and completes the whole writing process. However, if the operating system crashes before it finishes the writing of the operating system cache, there is a risk of data loss. Therefore, fsync parameter provides a way to write data into persistent storage finally before completing a writing process. However, this will lead to a decrease in the efficiency of database execution, there will be several GUC arguments appear to handle the performance. This parameter is valid not only for the flush of wal, but also for other data files. 



#### WAL_SYNC_METHOD

This parameter is a complement to the fsync parameter, which specifies the method to ensure the data is written to the persistent storage.



#### COMMIT_DELAY && COMMIT_SIBLINGS

The brush wal cache is triggered when the transaction is committed. At this time, PostgreSQL does a little more to improve performance: ‘commit_delay’ refers to the time allowed for wal cache to delay writing after the transaction is committed. The purpose of this delay is to wait for parallel brother transactions to be executed. After the brother transactions are committed, the wal logs will be written to disk together. If the brother transactions are not committed after the ‘commit_delay’ time, then current transaction completes the wall brush. The meaning of 'commit siblings' is the condition for the current transaction to wait. If the sibling transaction executed in parallel is smaller than commit siblings, the current transaction will not wait.



#### WAL_DEBUG

It is a develop argument, and we can ignore it.



**PostgreSQL usually uses the checkpoint process to clean up some obsolete files. For the wal file, some future wal files will be generated at checkpoint or the old wal files will be deleted. See the related configuration [guc2] . Note that the 7.1 version described here is no longer applicable to the latest PostgreSQL version.** 



#### WAL_FILES

In PostgreSQL version 7.1, if 'wal_files' parameter is greater than 0, some wal files will be created in advance during the checkpoint. When the 'wal_files' parameter is equal to 0, the wal segments will be created one by one drivered by PostgreSQL backends. This parameter was deleted in version 7.3 and another pre generation method for wal record instead.



#### CHECKPOINT_TIMEOUT

The time interval for PostgreSQL to execute a checkpoint. This parameter is not so related to wal, because the 'checkpoint_segment' below has similar functions, they are all the time to trigger the checopint, so it's also written here.



#### CHECKPOINT_SEGMENT

PostgreSQL executes the wal interval of checkpoint once, since the last checkpoint, PostgreSQL will trigger checkpoint again after writing a certain number of wal segments.



## Change the way of wal pregeneration(7.3)

This version has made minor changes based on version 7.1, mainly changing the method of pregenerating wal segment.

During a checkpoint, we do not pregenerate wal segments, but maintain wal segments in the wal generation directory with new policies: ① the number of wal segments is not more than 2 * checkpoint_segments + 1 in principle; ② a traffic peak may cause backend to generate wal logs quickly, and the number of wal segments may be increased to more than 2 * checkpoint ﹣ segments + 1. ③ When making checkpoints, if the number of wal segments exceeds the limit, the over limit wal segments will be deleted. If the number of wal segments does not exceed the limit, the old wal segment will be renamed as the future wal segment.

Therefore, the 'wal_file' parameter is no longer used and has been removed in this version.



## Convenience warning(7.4)

There is no functional change in this version, but a warning parameter 'checkpoint_warnings' been added.

#### CHECKPOINT_WARNING

If checkpoint caused by 'checkpoint_segments'  too frequently than 'checkkpoint_warning', it will come a warning in database log.



## PITR comming(8.0)

Finally, the layout of wal has been expanded to realize point in time recovery (Pitr). PITR is the physical backup mechanism of PostgreSQL. The main processes are: turn on archive; make base backup; create recovery.conf file in basebackup database and writing recovery parameters; starting base backup database. This blog is about history, not about the use of functions. 

![wal8.0](http://movead.gitee.io/picture/wal_train/walhistory/wal8.0.jpg)



#### ARCHIVE_COMMAND

This parameter provides a wal log archiving method for PostgreSQL



#### RESTORE_COMMAND

RECOVERY_TARGET_TIME,RECOVERY_TARGET_XID,RECOVERY_TARGET_INCLUSIVE,RECOVERY_TARGET_TIMELINE

These are parameters related to the recovery target.



## Full page write and warm standby(8.2)

When the operating system crashes, it may lead to partial write of PostgreSQL data pages, so data inconsistency may occur. In order to deal with this situation, PostgreSQL will back up a data page (full page write) in the wal log after a checkpoint, each time a data page is modified.

In addition, this version has the concept of PostgreSQL warm standby, which is based on the wal transfer at the wal segment level. The hot standby that appears later is based on the wal transfer at the wal record level. The warm standby machine periodically obtains the wal segment through the restore command, and performs the wal replay, so that the standby machine keeps catching up with the host data.

We can understand warm standby and Pitr here. In fact, if you want Pitr to stop after wal replays all wal logs and wait for the following wal to appear, this is warm standby. This version of "stop and wait for the next wal" is not implemented in the kernel, but requires you to integrate this step into the 'restore_command'.

![wal8.2](http://movead.gitee.io/picture/wal_train/walhistory/wal8.2.jpg)



**Here comes the [GUC5], this group is the arguments related to configure the contents of wal.**



#### FULL_PAGE_WRITE

The meaning of this parameter is to enable PostgreSQL to resist some write problems caused by operating system crash. On the other hand, because a lot of data page information is recorded in the wal log, the wal log will expand. In addition, the checkpoint executed during basic backup does not care about full page write and forces full page write.



#### ARCHIVE_TIMEOUT

Because wal archiving only takes effect on the completed wal segment, if a wal segment is not full for a long time, the wal segment will not be archived, and therefore cannot be used by the standby mechanism in the warm standby function.

'arcihve_timeout' defines a time period, if a wal period exceeds this time period and is not full, a wal switch archive is forced.



## Auto write wal buffer(8.3)

There are some small improvements and enhancements in version 8.3



#### SYNCHRONOUS_COMMIT

This parameter defines the conditions for the final completion of a transaction. If the parameter value is on, the transaction will not complete the commit until the wal log of transaction commit is synchronized to the wal segment. If the parameter value is off, the wal log of transaction commit can be written to the wal cache. In later versions, this parameter will be upgraded to correspond to some synchronous commits of hot standby.



#### WAL_WRITE_DELAY

In the description of the 'commit_delay' parameter, it is mentioned that in the case of no transaction commit, the wal cache will not write to the disk until the write is full. A time interval is defined here. If the wal cache is not written in this time range, the wal cache write will be triggered.



#### ARCHIVE_MODE

This parameter is added to separate the archive mode from the archive command parameter.



## Replication(9.0)

Replication is implemented here, and many corresponding GUC parameters are added for replication. Corresponding to warm standby, replication can also be called hot standby, which realizes to synchronization data by wal record between the primary and the standby.

![wal9.0](http://movead.gitee.io/picture/wal_train/walhistory/wal9.0.jpg)



Most GUCs appear as [GUC6] in the image, will not talk about all of them but pick some important.



#### WAL_LEVEL

The current version of wal supports three levels: minimal, aichive and hot standby.



#### WAL_SENDER_DELAY

The wal sending process sends the wal logs generated by the host to the standby machine every other period of time. This parameter configures the size of this time interval



#### WAL_KEEP_SEGMENTS

If for some reason, the synchronization delay between the master and the standby is large, it will cause the wal segments of the host to be removed before they are sent to the standby. In this way, the standby machine cannot synchronize the host data. Therefore, by configuring the wal 'keep_segments' parameter on the host, the wal logs generated by the host will not be cleared immediately, and the backup can complete data synchronization with sufficient time.



#### HOT_STANDBY

Configure whether to connect to this standby machine for query.



#### MAX_STANDBY_ARCHIVE_DELAY && MAX_STANDBY_STREAM_DELAY

When a wal redo operation conflicts with the currently executing query, you need to decide whether to wait for the query to complete redo or cancel the query to execute redo.This parameter sets a time to allow the query to continue executing. After this time, the query will be cancelled, and to finish th redo.

MAX_STANDBY_ARCHIVE_DELAY is used for file level wal transfer, that is, warm standby.

MAX_STANDBY_STREAM_DELAY is used for replication, or hot standby.



#### STANDBY_MODE

Define whether this is a PITR or a standby.





## Standby synchronous commit(9.1)



#### SYNCHRONOUS_COMMIT

This is not a new parameter, but an optional value of 'local' has been added to this parameter.

On: wal record writes to the disk in the local and standby before the transaction completes the commit state.

Local: wal record writes to the disk in the local,before the transaction completes the commit state.

Off: asynchronous submission



#### SYNCHRONOUS_STANDBY_NAMES

Point which is the synchronous standby.



#### HOT_STANDBY_FEEDBACK

The standby machine sends the query being executed by the standby machine to the host machine to inform the master not to clean up the related death tuples.



## Cascade replication(9.2)

The feature of version 9.2 is that there is a cascade replication. A standby machine can obtain wal logs from its upstream server, and can also deliver wal logs to the downstream server. There is no special GUC parameter for cascading replication, we just need to configure the upstream server as a standby machine.

![wal9.2](http://movead.gitee.io/picture/wal_train/walhistory/wal9.2.jpg)



Here are two new GUCs, which have no related to cascade replicate.



#### PAUSE_AT_RECOVERY_TARGET

If the recovery target is specified in the PITR, when the recovery target is reached, the startup process stops redo, and the database is still in the recovery state. You can connect to the database to see whether the current database state meets your expectations. If not, it provides an opportunity for you to reconfigure recovery.conf to continue recovery. Of course, this requires the 'pause_at_recovery_target parameter' to be configured as true, otherwise, redo will immediately enter a readable and writable state when it reaches the recovery target.



#### SYNCHRONOUS_COMMIT

This parameter comes again. This time, it provides a 'remote_write' option. 'remote_write' is more rigorous than on, when wal record is applied to the synchronous standby machine, the transaction will be considered completed.



## Small changes(9.3)

Deleted the 'replication_timeout' parameter, added the 'wal_sender_timeout' alternative parameter, and added the 'wal_receiver_timeout' parameter



## Slot and Logical(9.4)

Slot is implemented so that the sending server can just save the appropriate wal file.

The 'logical' level for wal appears, with the built-in wal log parsing plugin test_decoding and the tool pg_recvlogical.

![wal9.4](http://movead.gitee.io/picture/wal_train/walhistory/wal9.4.jpg)

#### WAL_LEVEL

Support 'logical' wal level.



#### WAL_LOG_HINTS

If some pages have unimportant page data changes, they also follow the full page write mechanism.



#### MAX_REPLICATION_SLOTS

Each replication standby machine can be configured to use a replication slot, which records the application of wal records of the corresponding standby machine, and the wal segments saved for the standby machine by the host are not cleaned.

This is the max value for number of slot.



#### PRIMARY_SLOT_NAME

Configure the slot to be used on the standby machine, and name the slot.



#### RECOVERY_MIN_APPLY_DELAY

After receiving the wal log, the standby machine will delay a period of time to complete the redo operation of the wal record. Synchronous replication is not affected by this configuration.

Consider that if you execute a delete operation without conditional filtering on the master, at this time, the delete operation has not been synchronized to the standby machine, emergency measures can be taken to remedy the data.



## Small changes(9.5)



#### WAL_COMPRESSION

Whether to compress full page write data in wal log.



#### MAX_WAL_SIZE && MIN_WAL_SIZE

Soft limit of wal segments size in wal directory. When the size of wal segment in wal directory is smaller than 'min_wal_segment', checkpoint will not touch them; when the size of wal segments in wal directory is larger than 'max_wal_segment', checkpoint will remove the old wal segment until the size of wal segment met 'max_wal_size'; otherwise, checkpoint will rename the unused wal segment to the name of future wal segment.



#### ARCHIVE_MODE

Add "always" option, whether a standby machine continues to archive the wal segments it receives.



#### RECOVERY_TARGET_ACTION

After the recovery, there are three options for database action: pause, promote, and shutdown. This parameter replaces the 'pause_at_recovery_target' parameter.



Pause means to stop redo, but the database is still in recovery mode at this time. This parameter allows you to view the current state of the database. If the current state does not meet your expectations, you can stop the database and modify recovery.conf, start the database and continue redo .And use the pg_xlog_replay_resume() command to exit the recovery mode of the database.

Promote: after the recovery, directly promote the database to exit the recovery mode.

Shutdown: This is equivalent to adding an additional shutdown operation on the basis of pause.





#### TRACK_COMMIT_TIMESTAMP

Record the commit time for the committed transaction.



#### WAL_RETRIEVE_RETRY_INTERVAL

This is a waiting time. When the standby machine has replayed all the wal logs, the walreceiver process detects whether there is a new wal log interval.



## Multi synchronous standby(9.6)

The biggest highlight of version 9.6 is to support multiple synchronous standby machines. In addition, the wal level (minimum, replica, logical) has been modified.



#### WAL_LEVEL

Wal level supports three types: minimal, replica and logical.



#### SYNCHRONOUS_COMMIT

Add the 'remote_apply' option. Wal record applied on the synchronous standby specified by 'synchronous_standby_names' before the master transaction is completed.



#### WAL_WRITE_FLUSH_AFTER

When a transaction is submitted asynchronously, the wal writer process will be notified to swipe the wal cache, but this swipe only refers to the swiping into the operating system cache.

When the size of wal records that are flushed into the operating system cache but have not completed the hard disk synchronization is greater than the value of 'wal_write_flush_after', the process of synchronizing wal cache to the hard disk will be triggered once.



#### SYNCHRONOUS_STANDBY_NAMES

Support multiple synchronous standby, by new string.



## Logical Replication(10.0)

In version 10.0, the user's visible xlog tends to be changed to wal. The most intuitive thing is to change the name of the generated directory of wal from pg_xlog to pg_wal. In addition, many functions related to wal are also changed from xlog to wal. In version 9.4, wal log at logical level has been implemented. Since 10.0, the PostgreSQL kernel has officially added the built-in use of logical wal.

![wal10.0](http://movead.gitee.io/picture/wal_train/walhistory/wal10.0.jpg)



#### RECOVERY_TARGET_LSN

Specify a stop LSN point for the PITR.



#### MAX_LOGICAL_REPLICATION_WORKES

Maximum number of logical replication workers.



#### MAX_SYNC_WORKERS_PER_SUBSCRIPTION

When creating a subscription, the subscripter can choose to copy all the data in all tables of the publisher. This parameter refers to the maximum number of parallel copy worker.



## Replication configuration tuning(12)

In the stream copy, PITR, or warm standby functions, the recovery.conf configuration file is no longer used, and all relevant parameters are transferred to the postgresql.conf configuration file. Use an empty recovery.signal file to mark this as a recovery process.



#### WAL_INIT_ZERO

When creating the wal segment, initialize all the wal segment data to 0 to ensure that the wal segment has applied for space.



#### WAL_RECYCLE

Whether to reuse the old wal segment.



#### PROMOTE_TRIGGER_FILE

Replace 'trigger_file' parameter.



## Summary

This blog shows the growth process of PostgreSQL's wal. Some functions are slowly added to PostgreSQL, and there are more and more GUC parameters appear. Of course, some parameters are discarded. I want to introduce the meaning and reasons of each parameter in detail at first, but this will be a larger blog. Later, I will indirectly write some short blogs to introduce the meaning, origin and implementation of each parameter. Finally, some tables of parameter enable and disable are attached.



| GUC                               | Appear | Disappear |
| --------------------------------- | ------ | --------- |
| CHECKPOINT_SEGMENTS               | 7.1    | 9.5       |
| CHECKPOINT_TIMEOUT                | 7.1    | -         |
| WAL_FILES                         | 7.1    | 7.3       |
| CHECKPOINT_WARNING                | 7.4    | -         |
| CHECKPOINT_COMPLETION_TARGET      | 8.4    | -         |
| MAX_WAL_SIZE                      | 9.5    | -         |
| MIN_WAL_SIZE                      | 9.5    | -         |
| WAL_WRITE_FLUSH_AFTER             | 9.6    | -         |
| COMMIT_DELAY                      | 7.1    | -         |
| COMMIT_SIBLINGS                   | 7.1    | -         |
| WAL_DEBUG                         | 7.1    | -         |
| FSYNC                             | 7.1    | -         |
| WAL_FILES                         | 7.1    | -         |
| WAL_SYNC_METHOD                   | 7.1    | -         |
| WAL_BUFFERS                       | 7.1    | -         |
| SYNCHRONOUS_COMMIT                | 8.3    | -         |
| WAL_WRITE_DELAY                   | 8.3    | -         |
| ARCHIVE_COMMAND                   | 8      | -         |
| ARCHIVE_TIMEOUT                   | 8.2    | -         |
| ARCHIVE_MODE                      | 8.3    | -         |
| MAX_REPLICATION_SLOTS             | 9.4    |           |
| FULL_PAGE_WRITE                   | 8.2    | -         |
| WAL_LEVEL                         | 9      | -         |
| VACUUM_DEFER_CLEAN_AGE            | 9      | -         |
| WAL_LOG_HINTS                     | 9.4    | -         |
| WAL_COMPRESSION                   | 9.5    | -         |
| TRACK_COMMIT_TIMESTAMP            | 9.5    | -         |
| WAL_INIT_ZERO                     | 12     | -         |
| WAL_RECYCLE                       | 12     | -         |
| MAX_WAL_SENDERS                   | 9      | -         |
| WAL_SENDER_DELAY                  | 9      | -         |
| WAL_KEEP_SEGMENTS                 | 9      | -         |
| SYNCHRONOUS_STANDBY_NAMES         | 9.1    | -         |
| REPLICATE_TIMEOUT                 | 9.1    | 9.3       |
| WAL_SENDER_TIMEOUT                | 9.3    | -         |
| RESTORE_COMMAND                   | 8      | -         |
| RECOVERY_TARGET_TIME              | 8      | -         |
| RECOVERY_TARGET_XID               | 8      | -         |
| RECOVERY_TARGET_INCLUSIVE         | 8      | -         |
| RECOVERY_TARGET_TIMELINE          | 8      | -         |
| HOT_STANDBY                       | 9      | -         |
| MAX_STANDBY_ARCHIVE_DELAY         | 9      | -         |
| MAX_STANDBY_STREAM_DELAY          | 9      | -         |
| ARCHIVE_CLEANUP_COMMAND           | 9      | -         |
| RECOVERY_END_COMMAND              | 9      | -         |
| STANDBY_MODE                      | 9      | 12        |
| PRIMARY_CONNINFO                  | 9      | -         |
| TRIGGER_FILE                      | 9      | 12        |
| WAL_RECEIVER_STATUS_INTERVAL      | 9.1    | -         |
| HOT_STANDBY_FEEDBACK              | 9.1    | -         |
| RECOVERY_TARGET_NAME              | 9.1    | -         |
| PAUSE_AT_RECOVERY_TARGET          | 9.2    | 9.5       |
| WAL_RECEIVER_TIMEOUT              | 9.3    | -         |
| PRIMARY_SLOT_NAME                 | 9.4    | -         |
| RECOVERY_MIN_APPLY_DELAY          | 9.4    | -         |
| WAL_RETRIEVE_RETRY_INTERVAL       | 9.5    | -         |
| RECOVERY_TARGET_ACTION            | 9.5    | -         |
| RECOVERY_TARGET_LSN               | 10     | -         |
| MAX_LOGICAL_REPLICATION_WORKES    | 11     | -         |
| MAX_SYNC_WORKERS_PER_SUBSCRIPTION | 12     | -         |
