# Have an eye on locks of PostgreSQL

The lock is an essential part of a database system. In PostgreSQL, there are various locks, such as table lock, row lock, page lock, transaction lock, advisory lock, etc. Some of these locks are automatically added to complete the database functions during the operation of the database system, and some are manually added for the elements in PostgreSQL through some SQL commands. This blog explores the locks in PostgreSQL.



## 1. A Little Bit About *pg_locks*

It is a locks view in PostgreSQL. Except for row locks added by the SELECT ... FOR command, we can observe all other locks existing in the database in this lock view. There is a granted attribute in the lock view, if this attribute is true, it means that the process has acquired the lock. Otherwise, it means that the process is waiting for the lock.



## 2. Table-level Lock

*ACCESS SHARE, ROW SHARE, ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE,SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE* is all the table-level locks, and below is the block relationship among them.

|                        | ACCESS SHARE | ROW SHARE | ROW EXCLU SIVE | SHARE UPDATE EXCLU SIVE | SHARE | SHARE ROW EXCLU SIVE | EXCLU SIVE | ACCESS EXCLU SIVE |
| ---------------------- | ------------ | --------- | -------------- | ----------------------- | ----- | -------------------- | ---------- | ----------------- |
| ACCESS SHARE           |              |           |                |                         |       |                      |            | X                 |
| ROW SHARE              |              |           |                |                         |       |                      | X          | X                 |
| ROW EXCLUSIVE          |              |           |                |                         | X     | X                    | X          | X                 |
| SHARE UPDATE EXCLUSIVE |              |           |                | X                       | X     | X                    | X          | X                 |
| SHARE                  |              |           | X              | X                       |       | X                    | X          | X                 |
| SHARE ROW EXCLUSIVE    |              |           | X              | X                       | X     | X                    | X          | X                 |
| EXCLUSIVE              |              | X         | X              | X                       | X     | X                    | X          | X                 |
| ACCESS EXCLUSIVE       | X            | X         | X              | X                       | X     | X                    | X          | X                 |

These are table-level locks that exist in PostgreSQL and table-level locks are stored in memory, we can view the table-level locks we added in the pg_locks view. The database can also automatically add some table-level locks during the running process, we can manually add any table-level lock to a table through the lock command, such as *LOCK TABLE t1 IN ACCESS SHARE MODE*, this manually adds an *ACCESS SHARE* lock to t1. Next, I will show every lock.



### ACCESS SHARE

1. When will this lock appear
   - Access share lock is acquired when a read-only query is performed on a table
   - Use the Lock command 

2. Locks conflict with it

   *ACCESS EXCLUSIVE*

3. Lock instance(Not by lock command)

   We can't easily capture the moment when an access share lock is added to the table, but we can use its conflict lock to verify it. Here, I add an  *ACCESS EXCLUSIVE* which is a conflicting lock of *ACCESS SHAR* for table t1.

   ```sql
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             11751
   (1 row)
   postgres=# begin;
   BEGIN
   postgres=# lock table t1 in ACCESS EXCLUSIVE mode;
   LOCK TABLE
   postgres=#
   ```

   Then we query the t1 table in another session, and  we found that the SQL command hung.

   ```sql
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             11804
   (1 row)
   
   postgres=# select * from t1 where i = 1;
   ```

   OK, let's observe the lock view, 11751 session has acquired  *ACCESS EXCLUSIVE* lock on t1, 11804 session is waiting for *ACCESS SHAR* lock on t1.

   ```sql
   select locktype, relation::regclass as rel, pid, mode, granted
   from pg_locks
   where pid <> pg_backend_pid() and locktype = 'relation'
   order by pid;
    locktype | rel |  pid  |        mode         | granted 
   ----------+-----+-------+---------------------+---------
    relation | t1  | 11751 | AccessExclusiveLock | t
    relation | t1  | 11804 | AccessShareLock     | f
   (2 rows)
   
   postgres=#
   ```

   

### ROW SHARE

   1. When will this lock appear

      - When using the SELECT ... FOR command to lock a row of data, a *ROW SHARE* lock is first added to the target table.
      - Use the Lock command 

   2. Locks conflict with it

      *EXCLUSIVE*, *ACCESS EXCLUSIVE*

   3. Lock instance(Not by lock command)

      ```sql
      postgres=# select pg_backend_pid();
       pg_backend_pid 
      ----------------
                11751
      (1 row)
      
      postgres=# begin;
      BEGIN
      postgres=# select * from t1 where i = 2 for share;
       i | j 
      ---+---
       2 | 2
      (1 row)
      
      postgres=# select locktype, relation::regclass as rel, pid, mode, granted
      from pg_locks where pid <> pg_backend_pid() and locktype = 'relation'  order by pid;
      locktype | rel |  pid  |     mode     | granted 
      ----------+-----+-------+--------------+---------
       relation | t1  | 19312 | RowShareLock | t
      (1 row)
      
      postgres=#
      ```
      
      Here we can see a *ROW SHARE* lock on table t1.
      
      

   ### ROW EXCLUSIVE

   1. When will this lock appear

      - Performing insert / delete / update on a table will add this lock to the table
      - Use the Lock command 

   2. Locks conflict with it

      SHARE，SHARE ROW EXCUSIVE，EXCUSIVE，ACCESS EXCUSIVE

   3. Lock instance(Not by lock command)

      When updating a table, first add a *ROW EXCLUSIVE* lock on the table

      ```sql
      postgres=# begin;
      BEGIN
      postgres=# select pg_backend_pid();
       pg_backend_pid 
      ----------------
                11751
      (1 row)
      
      postgres=# update t1 set j =j+1 where i = 2;
      UPDATE 1
      
      postgres=# select locktype, relation::regclass as rel, pid, mode, granted
      from pg_locks where pid <> pg_backend_pid() and locktype = 'relation'  order by pid;
       locktype | rel |  pid  |       mode       | granted 
      ----------+-----+-------+------------------+---------
       relation | t1  | 11751 | RowExclusiveLock | t
      (1 row)
      
      postgres=#
      ```
      
      After the above test, query the lock view and find that the *ROW EXCLUSIVE* lock has been added to the table t1.
      

   ### SHARE UPDATE EXCLUSIVE

   1. When will this lock appear

      - *analyse,vacuum, create index concur-ently*, these operations will acquire this lock
      - Use the Lock command 

   2. Locks conflict with it

      SHARE UPDATE EXCLUSIVE，SHARE ，SHARE ROW EXCLUSIVE，EXCLUSIVE，ACCESS EXCLUSIVE

   3. Lock instance(Not by lock command)

      Here we also use lock conflicts to prove that the *analyze* command adds a *SHARE UPDATE EXCLUSIVE* lock on the table.

      

      Add a *SHARE* lock which is a conflict lock of *SHARE UPDATE EXCLUSIVE* on t1 table
      
      ```
      postgres=# select pg_backend_pid();
       pg_backend_pid 
      ----------------
                11751
      (1 row)
      
      postgres=# begin;
      BEGIN
      postgres=# lock table t1 in share mode;
      LOCK TABLE
      postgres=#
      ```

      On another session, do a *analyse* command，and we can see the *analyse* command hung
      
      ```sql
      postgres=# select pg_backend_pid();
       pg_backend_pid 
      ----------------
                11804
      (1 row)
      postgres=# analyse t1;
      
	  ```
      
      Now look at the lock view, 11804 session is waiting for a *SHARE UPDATE EXCLUSIVE* lock
      
      ```
      postgres=# select locktype, relation::regclass as rel, pid, mode, granted
      from pg_locks where pid <> pg_backend_pid() and locktype = 'relation'  order by pid;
       locktype | rel |  pid  |           mode           | granted 
      ----------+-----+-------+--------------------------+---------
       relation | t1  | 11751 | ShareLock                | t
       relation | t1  | 11804 | ShareUpdateExclusiveLock | f
      (2 rows)
      
      ```

   



   ### SHARE

   1. When will this lock appear

      - CREATE INDEX will add a *SHARE* lock on the target table
      - Use the Lock command 

   2. Locks conflict with it

       ROW EXCLUSIVE， SHARE UPDATE EXCLUSIVE，SHARE ROW EXCLUSIVE，EXCLUSIVE，

      ACCESS EXCLUSIVE

   3. Lock instance(Not by lock command)

      Here we also use lock conflicts to prove that the *create index* command adds a share lock on the table.

      Add a *ROW EXCLUSIVE* lock which is a conflict lock of *SHARE* on t1 table

      ```sql
      postgres=# select pg_backend_pid();
       pg_backend_pid 
      ----------------
                11751
      (1 row)
      
      postgres=# begin;
      BEGIN
      postgres=# lock table t1 in row exclusive mode;
      LOCK TABLE
      postgres=#
      ```

      On another session, do a *create index* command，and we can see the *create index* command hung

      ```sql
      postgres=# select pg_backend_pid();
       pg_backend_pid 
      ----------------
                11804
      (1 row)
                  
      postgres=# create index on t1(i);
      
      ```

      Now look at the lock view, 11804 session is waiting for a *SHARE* lock

      ```sql
      postgres=# select locktype, relation::regclass as rel, pid, mode, granted
      from pg_locks where pid <> pg_backend_pid() and locktype = 'relation'  order by pid;
       locktype | rel |  pid  |       mode       | granted 
      ----------+-----+-------+------------------+---------
       relation | t1  | 11751 | RowExclusiveLock | t
       relation | t1  | 11804 | ShareLock        | f
      (2 rows)
      
      postgres=#
      ```

      

   ### SHARE ROW EXCLUSIVE

   1. When will this lock appear

      - This lock is not acquired automatically in PostgreSQL, it is only added in PostgreSQL through the LOCK command

   2. Locks conflict with it

      ROW EXCLUSIVE， SHARE UPDATE EXCLUSIVE，SHARE, SHARE ROW EXCLUSIVE，EXCLUSIVE，

      ACCESS EXCLUSIVE

   3. Lock instance(Not by lock command)

      None

   

   ### EXCLUSIVE

   1. When will this lock appear

      - This lock is not acquired automatically in PostgreSQL, it is only added in PostgreSQL through the LOCK command

   2. Locks conflict with it

      ROW SHARE, ROW EXCLUSIVE， SHARE UPDATE EXCLUSIVE，SHARE, SHARE ROW EXCLUSIVE，EXCLUSIVE，ACCESS EXCLUSIVE

   3. Lock instance(Not by lock command)

      None

   

   ### ACCESS EXCLUSIVE

   1. When will this lock appear

      - ALTER TABLE，DROP TABLE，TRUNCATE，REINDEX，CLUSTER，VACUUM FULL commands
      - Use the Lock command

   2. Locks conflict with it

      all table-level locks

   3. Lock instance(Not by lock command)

      Here we also use lock conflicts to prove that the *alter table* command adds an access exclusive lock on the table.

      Add a *ACCESS SHARE* lock which is a conflict lock of *ACCESS EXCLUSIVE* on t1 table

      ```sql
      postgres=# select pg_backend_pid();
       pg_backend_pid 
      ----------------
                11751
      (1 row)
      
      postgres=# begin;
      BEGIN
      postgres=# lock table t1 in access share mode;
      LOCK TABLE
      postgres=#
      ```

      On another session, do a *alter table* command，and we can see the *alter table* command hung

      ```
      postgres=# select pg_backend_pid();
       pg_backend_pid 
      ----------------
                11804
      (1 row)
      postgres=# alter table t1 add column k int;
      
      ```

      Now look at the lock view, 11804 session is waiting for a *ACCESS EXCLUSIVE* lock

      ```sql
      postgres=# select locktype, relation::regclass as rel, pid, mode, granted
      from pg_locks where pid <> pg_backend_pid() and locktype = 'relation'  order by pid;
       locktype | rel |  pid  |        mode         | granted 
      ----------+-----+-------+---------------------+---------
       relation | t1  | 11751 | AccessShareLock     | t
       relation | t1  | 11804 | AccessExclusiveLock | f
      (2 rows)
      
      postgres=#
      ```

   

   ## 3. ROW-level Lock

   FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE, FOR KEY SHARE are explicit row-level lock，below is the block relationship among them.

|                   | FOR KEY SHARE | FOR SHARE | FOR NO KEY UPDATE | FOR UPDATE |
| ----------------- | ------------- | --------- | ----------------- | ---------- |
| FOR KEY SHARE     |               |           |                   | X          |
| FOR SHARE         |               |           | X                 | X          |
| FOR NO KEY UPDATE |               | X         | X                 | X          |
| FOR UPDATE        | X             | X         | X                 | X          |

   

An explicit row-level lock is different from a table-level lock. A FOR UPDATE statement can lock many rows of a table at the same time. In this case, a downward lock cannot store lock information in memory like a table-level lock. Information is not displayed in the pg_locks view. The row lock indicates that the tuple is locked by modifying the t_infomask field and t_infomask2 information in the tuple. Through the code in heapam.c we can see the values of t_infomask and t_infomask2 in different row lock modes.

   ```c
   /*heapam.c*/
   switch (mode)
   			{
   				case LockTupleKeyShare: //FOR KEY SHARE
   					new_xmax = add_to_xmax;
   					new_infomask |= HEAP_XMAX_KEYSHR_LOCK;
   					break;
   				case LockTupleShare:	//FOR SHARE
   					new_xmax = add_to_xmax;
   					new_infomask |= HEAP_XMAX_SHR_LOCK;
   					break;
   				case LockTupleNoKeyExclusive://FOR NO KEY UPDATE
   					new_xmax = add_to_xmax;
   					new_infomask |= HEAP_XMAX_EXCL_LOCK;
   					break;
   				case LockTupleExclusive://FOR UPDATE
   					new_xmax = add_to_xmax;
   					new_infomask |= HEAP_XMAX_EXCL_LOCK;
   					new_infomask2 |= HEAP_KEYS_UPDATED;
   					break;
   				default:
   					new_xmax = InvalidTransactionId;	/* silence compiler */
   					elog(ERROR, "invalid lock mode");
   			}
   		}
   ```

   

   ```c
   /*htup_details.h*/
   
   /*
    * information stored in t_infomask:
    */
   #define HEAP_XMAX_KEYSHR_LOCK	0x0010	/* xmax is a key-shared locker */
   #define HEAP_COMBOCID			0x0020	/* t_cid is a combo cid */
   #define HEAP_XMAX_EXCL_LOCK		0x0040	/* xmax is exclusive locker */
   #define HEAP_XMAX_LOCK_ONLY		0x0080	/* xmax, if valid, is only a locker */
   #define HEAP_XMIN_COMMITTED		0x0100	/* t_xmin committed */
   ...
   /*
    * information stored in t_infomask2:
    */
   #define HEAP_KEYS_UPDATED		0x2000	/* tuple was updated and key cols
   										 * modified, or tuple deleted */
   
   ```

   

   

   ### FOR UPDATE

Use SELECT ... FOR UPDATE to complete the lock on some rows and a row with a FOR UPDATE lock will block FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE or FOR KEY SHARE operation.



Adding a *FOR UPDATE* lock to table t1 in a session

   ```sql
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             25019
   (1 row)
   
   postgres=# select * from t1;
    i | j 
   ---+---
    1 | 2
   (1 row)
   
   postgres=# begin;
   BEGIN
   postgres=# select * from t1 where i = 1 for update;
    i | j 
   ---+---
    1 | 2
   (1 row)
   
   postgres=#
   ```

In another session to update this record of t1, I found that the update could not be completed, it hung.

   ```
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             26596
   (1 row)
   
   postgres=# update t1 set j = j + 1 where i = 1;
   
   ```

Let's check the pg_locks view:

   ```sql
   postgres=# select locktype, relation::regclass as rel, pid, mode, granted
   postgres-# from pg_locks where pid <> pg_backend_pid() order by pid;
      locktype    | rel |  pid  |       mode       | granted 
   ---------------+-----+-------+------------------+---------
    virtualxid    |     | 25019 | ExclusiveLock    | t
    relation      | t1  | 25019 | RowShareLock     | t
    transactionid |     | 25019 | ExclusiveLock    | t
    transactionid |     | 26596 | ShareLock        | f
    transactionid |     | 26596 | ExclusiveLock    | t
    tuple         | t1  | 26596 | ExclusiveLock    | t
    virtualxid    |     | 26596 | ExclusiveLock    | t
    relation      | t1  | 26596 | RowExclusiveLock | t
   (8 rows)
   ```

We found that there is no 25019 session lock on a table record(tuple) in the lock view, but the subsequent 26596 session has added an *EXCLUSIVE* lock to this record.This also proves that row locks do not appear in the pg_locks view. Explicit *FOR UPDATE* row lock do not block *EXCLUSIVE* row locks.

   

Let's take a look at the data of table t1 through pageinspace's heap_page_items () function

   ```sql
   postgres=# select ctid,* from t1;
    ctid  | i | j 
   -------+---+---
    (0,1) | 1 | 2
   (1 row)
   
   postgres=# select lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask from  heap_page_items(get_raw_page('t1', 0));
    lp | t_xmin | t_xmax | t_ctid | t_infomask2 | t_infomask 
   ----+--------+--------+--------+------------+------------
     1 |    631 |    633 | (0,1)  |       8194 |       448
   (1 row)
   
   postgres=#
   ```



t_infomask value: 448=0x1C0

t_infomask2 value: 8194=0x2002

So t_infomask & HEAP_XMAX_EXCL_LOCK is true and t_infomask2 & HEAP_KEYS_UPDATED is true

*(Here you should drop back to see the code in htup_details.h)*

So we can say the tuple(lp=1) has a FOR UPDATE lock now.

   

   

   ### FOR NO KEY UPDATE

Use SELECT ... FOR NO KEY UPDATE to complete the lock on some rows and a row with a FOR UPDATE lock will block FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE operation.



Based on the 25019 session above, we newly insert a test data and start a new transaction, and add a FOR NO KEY UPDATE lock on this new row

   ```
   postgres=# commit;
   COMMIT
   postgres=# insert into t1 values(2,2);
   INSERT 0 1
   postgres=# begin;
   BEGIN
   postgres=# select * from t1 where i = 2 for no key update;
    i | j 
   ---+---
    2 | 2
   (1 row)
   
   postgres=#
   ```

   

Similarly we look at the data of the t1 table on the hard disk.

*(Some people may wonder why there is one more record, because an update statement is executed in the 26596 session. After his blocking transaction is committed, the session has also completed the update operation, so there is one more tuple)*

   ```sql
   postgres=# select ctid,* from t1; 
    ctid  | i | j 
   -------+---+---
    (0,2) | 1 | 3
    (0,3) | 2 | 2
   (2 rows)
   
   postgres=# select lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask from  heap_page_items(get_raw_page('t1', 0));
    lp | t_xmin | t_xmax | t_ctid | t_infomask2 | t_infomask 
   ----+--------+--------+--------+------------+------------
     1 |    631 |    634 | (0,2)  |      16386 |        1280
     2 |    634 |    636 | (0,2)  |      32770 |       10496
     3 |    635 |    637 | (0,3)  |          2 |         448
   (3 rows)
   
   postgres=#
   
   ```

lp=3 is the tuple we want,

t_infomask value:448=0x1C0

t_infomask & HEAP_XMAX_EXCL_LOCK is true

t_infomask2 & HEAP_KEYS_UPDATED is false

So we can say the tuple(lp=3) has a FOR NO KEY UPDATE lock now.

   

   ### FOR SHARE

Use SELECT ... FOR SHARE to complete the lock on some rows and a row with a FOR UPDATE lock will block FOR UPDATE, FOR NO KEY UPDATE operation.



Based on the 25019 session above, we newly insert a test data and start a new transaction, and add a FOR SHARE lock on this new row.

   ```sql
   postgres=# commit;
   COMMIT
   postgres=# insert into t1 values(3,3);
   INSERT 0 1
   postgres=# begin;
   BEGIN
   postgres=# select * from t1 where i = 3 for share;
    i | j 
   ---+---
    3 | 3
   (1 row)
   
   postgres=#
   ```

We look at the data of the t1 table on the hard disk.

   ```sql
   postgres=# select ctid,* from t1; 
    ctid  | i | j 
   -------+---+---
    (0,2) | 1 | 3
    (0,3) | 2 | 2
    (0,4) | 3 | 3
   (3 rows)
   
   postgres=# select lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask from  heap_page_items(get_raw_page('t1', 0));
    lp | t_xmin | t_xmax | t_ctid | t_infomask2 | t_infomask 
   ----+--------+--------+--------+------------+------------
     1 |    631 |    634 | (0,2)  |      16386 |       9984
     2 |    634 |    636 | (0,2)  |      32770 |       8640
     3 |    635 |    637 | (0,3)  |          2 |        448
     4 |    638 |    639 | (0,4)  |          2 |        464
   (4 rows)
   
   postgres=#
   ```

lp=4 is the tuple we want,

t_infomask value:464=0x1D0

t_infomask & HEAP_XMAX_SHR_LOCK is true

So we can say the tuple(lp=3) has a FOR SHARE lock now.

   

   ### FOR KEY SHARE

Use SELECT ... FOR KEY SHARE to complete the lock on some rows and a row with a FOR UPDATE lock will block FOR UPDATE operation.



Based on the 25019 session above, we newly insert a test data and start a new transaction, and add a FOR KEY SHARE lock on this new row.

   ```sql
   postgres=# commit;
   COMMIT
   postgres=# insert into t1 values(4,4);
   INSERT 0 1
   postgres=# begin;
   BEGIN
   postgres=# select * from t1 where i = 4 for key share;
    i | j 
   ---+---
    4 | 4
   (1 row)
   ```

   

We look at the data of the t1 table on the hard disk.

   ```
   postgres=# select lp, t_xmin, t_xmax, t_ctid, t_infomask2, t_infomask from  heap_page_items(get_raw_page('t1', 0));
    lp | t_xmin | t_xmax | t_ctid | t_infomask2 | t_infomask 
   ----+--------+--------+--------+-------------+------------
     1 |    631 |    634 | (0,2)  |       16386 |       9984
     2 |    634 |    636 | (0,2)  |       32770 |       8640
     3 |    635 |    637 | (0,3)  |           2 |        448
     4 |    638 |    639 | (0,4)  |           2 |        464
     5 |    640 |    642 | (0,5)  |           2 |        400
   (5 rows)
   
   postgres=#
   ```

lp=5 is the tuple we want,

t_infomask value:400=0x190

t_infomask & HEAP_XMAX_KEYSHR_LOCK is true

So we can say the tuple(lp=5) has a FOR KEY SHARE lock now.

   

   ### EXCLUSIVE

In addition to using the FOR ... series of explicit row-level locks, there be row-level lock stored in memory in the database, which can be displayed in the pg_locks view, let's show it.

The *EXCLUSIVE* row block  *EXCLUSIVE* row block only, it do not matter with FOR ... series locks.

 

Start a transaction and update a row of data in a session

   ```sql
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             17501
   (1 row)
   
   postgres=# begin;
   BEGIN
   postgres=# update t1 set j = j+1 where i = 2;
   UPDATE 1
   postgres=#
   ```

 

In another session, update the same row of data and found that the update command hung.

   ```sql
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             17559
   (1 row)
   
   postgres=# update t1 set j = j+1 where i = 2;
   
   ```

We look at the pg_locks view

   ```
   postgres=#  select locktype, relation::regclass as rel, pid, mode, granted
   from pg_locks where pid <> pg_backend_pid() and locktype = 'tuple'  order by pid;
    locktype | rel |  pid  |     mode      | granted 
   ----------+-----+-------+---------------+---------
    tuple    | t1  | 17559 | ExclusiveLock | t
   (1 row)
   
   postgres=#
   ```

At this point we found that session 17559 added a row lock to a row in the t1 table.

 

   ## 4. Transaction Lock

There is a transaction lock which can't be added manually.

   ```sql
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             17501
   (1 row)
   
   postgres=# begin;
   BEGIN
   postgres=# update t1 set j = j+1 where i = 2;
   UPDATE 1
   postgres=select locktype, transactionid, pid, mode, granted
   from pg_locks where pid <> pg_backend_pid() and locktype = 'transactionid'  order by pid;
      locktype    | transactionid |  pid  |     mode      | granted 
   ---------------+---------------+-------+---------------+---------
    transactionid |           674 | 17501 | ExclusiveLock | t
   (1 row)
   ```

In this example, we started a transaction and performed a row of data update in the transaction. After querying the lock view, we can see that an exclusive lock was added on transaction 674. When other concurrent processes want to determine the visibility of the old tuple will be blocked by this lock.



   ## 5. Page Lock

The page lock is also a pass in the official document, because page locks are only used in some indexes in PostgreSQL, and these are basically invisible to us and are not testable. According to the official document, it says 'but they are mentioned here for completeness'.



   ## 6. Advisory Lock

PostgreSQL provides a way to create locks whose meaning is defined by the application, called advisory locks. PostgreSQL is only responsible for creating advisory locks or unlocking after receiving the advisory lock command, but PostgreSQL does not use these locks, these locks will be used by the application that created them. Advisory locks are also explicit in the pg_locks view. The following table is the function of the advisory lock:



| **Name**                                                 | **Return Type** | **Description**                  |
| ------------------------------------------------------------ | --------- | ------------------------------------ |
| pg_advisory_lock(key bigint)                                 | `void`    | Obtain exclusive session level advisory lock |
| pg_advisory_lock(*`key1`* int, *`key2`* int)                 | `void`    | Obtain exclusive session level advisory lock |
| pg_advisory_lock_shared(*`key`* bigint)                      | `void`    | Obtain share session level advisory lock |
| pg_advisory_lock_shared(*`key1`* int, *`key2`* int)          | `void`    | Obtain share session level advisory lock |
| pg_advisory_unlock(*`key`* bigint)                           | `boolean` | Release an exclusive session level advisory lock |
| pg_advisory_unlock(*`key1`* int, *`key2`* int)               | `boolean` | Release an exclusive session level advisory lock |
| pg_advisory_unlock_all()                                     | `void`    | Release all session level advisory locks held by the current session |
| pg_advisory_unlock_shared(*`key`* bigint) | `boolean`  |Release a shared session level advisory lock|
| pg_advisory_unlock_shared(*`key1`* int, *`key2`* int)        | `boolean` | Release a shared session level advisory lock |
| pg_advisory_xact_lock(*`key`* bigint)                        | `void`    | Obtain exclusive transaction level advisory lock |
| pg_advisory_xact_lock(*`key1`* int, *`key2`* int)            | `void`    | Obtain exclusive transaction level advisory lock |
| pg_advisory_xact_lock_shared(*`key`* bigint)                 | `void`    | Obtain share transaction level advisory lock |
| pg_advisory_xact_lock_shared(*`key1`* int, *`key2`* int)     | `void`    | Obtain share transaction level advisory lock |
| pg_try_advisory_lock(*`key`* bigint)                         | `boolean` | Obtain exclusive session level advisory lock if available |
| pg_try_advisory_lock(*`key1`* int, *`key2`* int)             | `boolean` | Obtain exclusive session level advisory lock if available |
| pg_try_advisory_lock_shared(*`key`* bigint)                  | `boolean` | Obtain share session level advisory lock if available |
| pg_try_advisory_lock_shared(*`key1`* int, *`key2`* int)      | `boolean` | Obtain share session level advisory lock if available |
| pg_try_advisory_xact_lock(*`key`* bigint)                    | `boolean` | Obtain exclusive session level advisory lock if available |
| pg_try_advisory_xact_lock(*`key1`* int, *`key2`* int)        | `boolean` | Obtain exclusive session level advisory lock if available |
| pg_try_advisory_xact_lock_shared(*`key`* bigint)             | `boolean` | Obtain share session level advisory lock if available |
| pg_try_advisory_xact_lock_shared(*`key1`* int, *`key2`* int) | `boolean` | Obtain share session level advisory lock if available |

Next we simply test the advisory lock.



   ### Transaction-level advisory

   

Acquire transaction-level exclusive locks

   ```
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             11751
   (1 row)
   
   postgres=# begin;
   BEGIN
   postgres=# select pg_advisory_xact_lock(100);
    pg_advisory_xact_lock 
   -----------------------
    
   (1 row)
   
   postgres=#
   ```

Check in the pg_locks view, there's an advisory lock here.

   ```sql
   postgres=# select locktype, relation::regclass as rel, pid, objid, mode, granted
   from pg_locks where pid <> pg_backend_pid() and locktype='advisory';
    locktype | rel |  pid  | objid |     mode      | granted 
   ----------+-----+-------+-------+---------------+---------
    advisory |     | 11751 |   100 | ExclusiveLock | t
   (1 row)
   
   postgres=#
   ```

After committing the transaction, check the lock view and find that the transaction advisory lock is gone

   ```
   postgres=# select pg_backend_pid();
    pg_backend_pid 
   ----------------
             11751
   (1 row)
   
   postgres=# commit;
   COMMIT
   postgres=# select locktype, relation::regclass as rel, pid, objid, mode, granted
   from pg_locks where pid <> pg_backend_pid() and locktype='advisory';
    locktype | rel | pid | objid | mode | granted 
   ----------+-----+-----+-------+------+---------
   (0 rows)
   
   postgres=#
   
   ```

   

   ### Session-level advisory

Get session-level advisory lock, which are independent of transactions and do not need to open an additional transaction block

   ```
   postgres=# select pg_advisory_lock(1000);
    pg_advisory_lock 
   ------------------
    
   (1 row)
   
   postgres=# select locktype, relation::regclass as rel, pid, objid, mode, granted
   from pg_locks where pid <> pg_backend_pid() and locktype='advisory';
    locktype | rel |  pid  | objid |     mode      | granted 
   ----------+-----+-------+-------+---------------+---------
    advisory |     | 11751 |  1000 | ExclusiveLock | t
   (1 row)
   
   postgres=#
   ```

After the current session ends, the lock will be released automatically. Of course, you can also use the pg_advisory_unlock () function to release the lock in advance.

   

   ## 7. In the end

This blog analyzes every postgres lock and adds experimental use cases to it. This module is not a conclusion but a postscript, because this is a PostgreSQL lock exploration blog. Only after you fully understand the locks that exist in the database can you calmly cope with various query problems that occur in the database.

   

