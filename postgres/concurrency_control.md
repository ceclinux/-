## Concurrency Control

Concurrency Control is a mechanism that maintains consistency and isolation, which are two properties of the ACID, when several transactions run concurrently in the database.

There are three broad concurrency control techniques, MVCC, Strict Two-Phase Locking, and Optimistic Concurrency Control(OCC), and each technique has many variations. In MVCC, each write operation creates a new version of a data item where retaining the old version. When a transaction reads a data item, the system selects one of the versions to ensure isolation of the individual transaction. The main advantage of MVCC is that 'reader don't block writers, and writers don't' block readers', in contrast, for example, an S2PL-based system must block readers when a writer writes an item because the writer acquires an exclusive lock for them. PG and some RDBMSs use a variation of MVCC called Snapshot Isolation(SI).

To implement SI, a new data item is inserted directly into the relevant table page. When reading items, PostgresSQL selects the appropriate version of an item in response to an individual transaction by applying **visibility check rules**.

SI does not allow the three anomalies defined int the ANSI SQL-92 standard, i.e. Dirty Reads, Non-Repeatable Reads, and Phantom Reads.

Whenever a transaction begins, a unique identifier, referred to as a transaction id(txid), is assigned by the transaction manager. PG's txid a 32-bit unsigned integer, if you execute the `txid_current()` function after a transaction starts, the function returns the current txid as follows.

```sql
postgres=# BEGIN;
BEGIN
postgres=# select txid_current();
 txid_current
--------------
          529
(1 row)
```

Txids can be compared with each other. For example, at the viewpoint of txid 100, txids which are greater than 100 are 'in the future' and they are invisible from the txid 100; txids which are less than 100 are 'in the past' and visible.

The heap tuple comprises three parts, the **HeapTupleHeaderData** structure, Null bitmap, and user data.

Four fields of **HeapTuleHeaderData** are required in the subsequent sections.

*t_xmin* holds the txid of the transaction that inserted this tuple.
*t_xmax* holds the txid of the transaction that deleted or updated this tuple. If this tuple has not been deleted or updated, `t_xmax` is set to 0, which means INVALID.
*t_cid* holds the command id(cid), which means how many SQL commands were executed before this command was executed within the current transaction beginning from 0. For example, assume that we execute three INSERT commands within a single transaction: 'BEGIN; INSERT; INSERT;INSERT; COMMIT;'. It the first command inserts this tuple, `t_cid` is set to 0. If the second command inserts this, `t_cid` is set to 1, and so on.

![](https://img.vim-cn.com/70/d1457b65090a2197a561ed51a9454c00442251.png)

![](https://img.vim-cn.com/f2/9b915a9593a6d469e623aa5a5ab2ad308b3abb.png)

Suppose that a tuple is inserted in a page by a transaction whose txid is 99. In this case, the header fields of the inserted tuple are set as follows.

### Insertion

```sql
postgres=# CREATE EXTENSION pageinspect;
CREATE EXTENSION
postgres=# CREATE TABLE tbl (data text);
CREATE TABLE
postgres=#
postgres=# insert into tbl values('A');
INSERT 0 1
postgres=# select lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid from heap_page_items(get_raw_page('tbl', 0));
 tuple | t_xmin | t_xmax | t_cid | t_ctid
-------+--------+--------+-------+--------
     1 |    540 |      0 |     0 | (0,1)
(1 row)
```

### DELETION

In the deletion operation, the target tuple is deleted logically. The value of the txid that executes the DELETE command is set to the `t_xmax` of the tuple.

![](//img.vim-cn.com/f7/cce17d9670f7d57cb6ace3c37c38f6335f7c46.png)

Suppose that Tuple_1 is deleted by txid 111. In this case, the header fields of Tuple_1 are set as follows.

Tuple_1:
  **t_xmax** is set to 111

If txid 111 is committed, Tuple_1 is no longer required. Generally, unneeded tuple are referred to as dead tuples in PostgresSQL.

Dead tuples should eventually be removed from pages. Cleaning dead tuples is referred to as `VACUUM` processing, which is described in Chap 6.

### Update

In the update operation, PostgresSQL logically deletes the latest tuple and insert a new one.

![](https://img.vim-cn.com/95/454cad632d3372e181b11f2898dd12eb1ba8f1.png)

Suppose that the row, which has been inserted by txid 99, is updated twice by txid 100.

When the first `UPDATE` command is executed, `Tuple_1` is logically deleted by setting `txid` 100 to the `t_xmax`, and then `Tuple_2` is inserted. Then, the `t_cid` of `Tuple_1` is rewritten to point to `Tuple_2`. The header fields of both `Tuple_1` and `Tuple_2` are as follows.

## Free Space Map

When inserting a heap or an index tuple, PostgresSQL uses the FSM of the corresponding table or index to select the page which can be inserted it.

All tables and indexes have respective FSMs. Each FSM stores the information about the free capacity of each page within the corresponding table or index file.

All FSMs are stored with the suffix 'fsm', and they are loaded into shared memory if necessary.

```sql
testdb=# CREATE EXTENSION pg_freespacemap;
CREATE EXTENSION

testdb=# SELECT *, round(100 * avail/8192 ,2) as "freespace ratio"
                FROM pg_freespace('accounts');
 blkno | avail | freespace ratio 
-------+-------+-----------------
     0 |  7904 |           96.00
     1 |  7520 |           91.00
     2 |  7136 |           87.00
     3 |  7136 |           87.00
     4 |  7136 |           87.00
     5 |  7136 |           87.00
....
```

## Commit Log

PostgreSQL holds the status of transactions in the Commit Log.The Commit Log, often called the `clog`, is allocated to the shared memory, and is used throughout transaction processing.

### Transaction Status

PostgresSQL defines four transaction states. `IN_PROGRESS, COMMITTED, ABORTED, and SUB_COMMITTED`.

The clog comprises one or more 8KB pages in shared memory. The clog logically forms an array. The indices of the array correspond to the respective transaction ids, and each item in the array holds the status of the corresponding transaction id. 

![](https://img.vim-cn.com/51/cc215a3ffa0939a6523de16a753f916f1c8520.png)

### How Clog Performs

When PostgresSQL shuts down or whenever the checkpoint process runs, the data of the clog are written into files stored under the `pg_clog` subdirectory. These files are named 0000, 0001, etc. The maximum file size is 256KB. 

When PostgreSQL starts up, the data stored in the `pg_clog`'s file are loaded to initialize the clog.

The size of the clog continuously increases because a new page is appended whenever the clog is filled up. However, not all data in the clog are necessary. However, not all data in the clog are necessary. Vacuum processing, described in `Chap 6`, regularly removes such old data(both the clog pages and files).

### Transaction Snapshot

A transaction snapshot is a dataset that stored information about whether all transactions are active, at certain point in time for an individual transaction. Here an active transaction means it is in progress or has not yet started.

PostgreSQL internally defines the textual representation format of transaction snapshots as '100:100'. For example, '100:100' means 'txids that are less than 99 are not active, not txids that are equal or greater than 100 are active.'

```sql
postgres=# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 544:544:
(1 row)
```

The textual representation of the `txid_current_snapshot` is `xmin:xmax:xip_list`, and the components are described as follows.

1. xmin

Easiest txid that is still active. All earlier transactions will either committed and visible, or rollback and dead.

2. xmax

First as-yet-unassigned txid. All txids greater than or equal to this are not yet started as of the time of the snapshot, and thus invisible.

3. xip_list

Active txids at the time of the snapshot. This list includes only active txids between xmin and xmax.

![](https://img.vim-cn.com/ae/9041da58bfc48707b3492cb0297a9405783ef9.png)

- txids that are equal or less than 99 are not active because xmin is 100
- txids that are equal or greater than 100 are active because xmax is 100

The second example is '100:104:100, 102'. This snapshot means the following(Fig. 5.8(b))

- txids that are equal or less than 99 are not active.
- txids that are equal or greater than 104 are active
- txids 100 and 102 are active since they exist in the xip list, whereas txids 101 and 103 are not active.

Transaction snapshots are provided by the transaction manager. In the READ COMMITTED isolation level, the transaction obtains a snapshot whenever an SQL command is executed; otherwise(REPEATABLE READ or SERIALIZABLE), the transaction only gets a snapshot when the first SQL command is executed. The obtained transaction snapshot is used for visibility check for tuples.

When using the obtained snapshot for the visibility check, active transactions in the snapshot must be treated as in progress even if they have actually been committed or aborted. This rule is important because it causes the diff in the behavior between RC and RR(or S).

![](//img.vim-cn.com/92/c290c210344bd8a7b0a9550e43f870c8573b5b.png)

The transaction manager always holds information about currently running transactions. 


**T1**

Transcation_A starts and executes the first SELECT command. When executing the first command, Transcation_A request the txid and snapshot of this moment. In this scenario, the transaction manager assigned txid 200, and returns the transaction snapshot '200:200'.

**T2**

Transcation_B starts and executes the first SELECT command. The transcation manager assigns txid 201, and returns the transcation snapshot '200:200:' because Transcation_A is in process. Thus, Transcation_A cannot be seen from Transcation_B.

**T3**

Transaction_C starts and executes the first SELECT command. The transcation manager assigns txid 202, and returns the transaction snapshot '200: 200:'. Thus, transcation_A and transcation_B cannot be seen from Transcation_C.

**T4**

Transcation_A has been committed. The transcation manager removes the information about this transcation.

**T5**

Transcation_B and transcation_C execute their respective SELECT commands. Transcation_B requires a transcation snapshot because it is in the READ COMMITTED level. In this scenario, Transcation_B obtains a new snapshot '201:201:' because Transcation_A(txid 200) is committed. Thus, Transcation_A is no longer invisible from Transcation_B. Transcation_C does not require a transcation snapshot because it is in the REPEATABLE READ level and uses the obtained snapshot, i.e '200:200:'. Thus, Transaction_A is still invisible from Transcation_C.

## Visibility Check rule

Visibility check rules are a set of rules used to determine whether each tuple is visible or invisible using both the `t_xmin` and `t_xmax` of the tuple, the clog, and the obtained transcation snapshot.

### Status of t_xmin is aborted

A tuple whose `t_xmin` status is aborted is always invisible because transcation that inserted this tuple has been aborted.

```
Rule 1:
If t_xmin status is 'ABORTED' THEN
  RETURN 'Invisible'
END IF
```

### Status of t_xmin is IN_PROGRESS

A tuple whose t_xmin status is `IN_PROGRESS` is essentially invisible(Rules 3 and 4), except under one condition.

```
IF t_xmin status is 'IN_PROGRESS' THEN
  IF t_xmin = current_txid THEN
     IF t_xmax = INVALID THEN
       RETURN 'Visible'
     ELSE
       RETURN 'Invisible'
     END IF
  ELSE 
    RETURN 'Invisible'
  END IF
END IF
```

If this tuple is inserted by another transcation and the status of `t_xmin` is `IN_PROGRESS`, this tuple is obviously invisible.

If `t_xmin` is equal to the current txid(i.e, this tuple is inserted by the current transcation) and `t_xmax` is not INVALID, this tuple is invisible because it has been updated or deleted by the current transcation.

The exception condition is the case whereby this tuple is inserted by the current transcation and `t_xmax` is INVALID. In this case, this tuple is visible from the current transcation.

```
Rule 2: If Status(t_xmin) = IN_PROGRESS ^ t_xmin = current_txid ^ t_xmax = INVALID => Visible

Rule 3: If Status(t_xmin) = IN_PROGRESS ^ t_xmin = current_txid ^ t_xmax not INVALID => Invisible

Rule 4: If Status(t_xmin) = IN_PROGRESS ^ t_xmin not current_txid => Invisible
```

### Status of t_xmin is COMMITTED

```
     IF t_xmin status is 'COMMITTED' THEN
     IF t_xmin is active in the obtained transcation snapshot THEN
     RETURN 'Invisible'
     ELSE IF t_xmax = INVALID OR status of t_xmax is 'ABORTED' THEN
     RETURN 'Visible'
     ELSE IF t_xmax status is 'IN_PROGRESS' THEN
     IF t_xmax = current_txid THEN
          RETURN 'Invisible'
     ELSE
          RETURN 'Visible'
     END IF
     ELSE IF t_xmax status is 'COMMITTED' THEN
     IF t_xmax is active in the obtained transcation snapshot THEN
          RETURN 'Visible'
     ELSE 
          RETURN 'Invisible'
     END IF
     END IF
     END IF
```

Rule 6 is obvious because `t_xmax` is `INVALID` or `ABORTED`. Three exception conditions and both Rule 8 and 9 are described as follows.

The first exception condition is that `t_xmin` is active in the obtained transcation snapshot. Under this condition, this tuple is invisible because `t_xmin` should be treated as in progress.

The second exception condition is that `t_xmax` is the current txid(Rule 7). Under this condition, as with Rule 3, this tuple is invisible because it has been updated or deleted by this transcation itself.

In contrast, if the status of `t_xmax` is `IN_PROGRESS` and `t_xmax` is not the current txid(Rule 8), the tuple is visible because it has not been deleted.

The third exception condition is that the status of `t_xmax` is `COMMITTED`  and `t_xmax` is not active in the obtained transcation snapshot. Under this condition, this tuple is invisible because it has been updated or deleted by another transcation.

In contrast, if the status of `t_xmax` is COMMITTED but `t_xmax` is active in the obtained transcation snapshot(Rule 9), the tuple is visible because `t_xmax` should be treated as in progress.

```
Rule 5: If Status(t_xmin) == COMMITTED ^ Snapshot(t_xmin) = active => Invisible

Rule 6: If Status(t_xmin) == COMMITTED ^ (t_xmax = INVALID or Status(t_max) = ABORTED) = Visible

Rule 7: If Status(t_xmin) = COMMITTED ^ Status(t_xmax) = INPROGRESS ^ t_xmax equal to current_txid => InVisible

Rule 8: If Status(t_xmin) = COMMITTED ^ Status(t_xmax) = IN_PROGRESS ^ t_xmax is not current_txid => Visible

Rule 9: If Statu(t_xmin) = COMMITTED ^ Status(t_xmax) = COMMITTED ^ Snapshot(t_xmax) = active => Visible

Rule 10: If Status(t_xmin) = COMMITTED ^ Status(t_xmax) = COMMITTED ^ Snapshot(t_xmax) is not active => Invisible
```

## Phantom Reads in PostgresSQL Repeatable Read Level

RR as defined in the ANSI SQL-92 standard allow Phantom Reads. However, Postgres's implementation does not allow them. In principle, SI does not allow phantom reads.

Assume that two transactions, i.e Tx_A and Tx_B, are running concurrently. The isolation level are READ COMMITTED and REPEATABLE READ, and their txids are 100 and 101, respectively. First, Tx_A inserts a tuple. Then, it is committed. Then `t_xmin` of the inserted tuple is 100. Next, `Tx_B` executes a `SELECT` command; however, the tuple inserted by `Tx_A` is invisible by Rule 5.

Rule 5(new tuple): Status(t_xmin:100) = COMMITTED ^ Snapshot(x_xmin:100) = active => invisible


## Preventing Lost Updates

A Lost update, also known as a `ww-conflict`, is an anomaly that occurs when concurrent transactions update the same rows, and it must be prevented in both the `REPEATABLE READ` and `SERIALIZABLE` levels. This section describe how PostgresSQL prevents Lost Update and show examples.

```

FOR each row that will be updated by this UPDATE command
  WHILE true
    IF the target row is being updated THEN
      WAIT for the termination of the transcation that updated the target row

      IF(the status of the terminated transcation is COMMITTED)
        AND (the isolation level of this transcation is REPEATABLE READ or SERIALIZABLE) THEN
          ABORT this transcation
      ELSE
        GOTO step(2)
      END IF
    ELSE IF the target row has been updated by the another concurrent transcation THEN
      IF the isolcation level of this transcation is READ COMMITTED THEN
        UPDATE the target row
      ELSE
        ABORT the transcation
      END IF
     ELSE
       UPDATEW the target row
     END IF
  END WHILE
 END FOR 

```

## Serializable Snapshot Isolation

Serializable Snapshot Isolation(SSI) has been embedded in SI since version 9.1 to realize a true SERIALIZABLE isolation level.

### Basic Strategy for SSI Implementation

If a cycle that is generated with some conflicts is present in the procedure graph, there will be a serialization anomaly. This is explained using the simplest anomaly. i.e Write-Skew

Figure 5.12(1) shows a schedule. Here, Transcation_A reads tuple_B and Transcation_B reads tuple A. Then, Transcation_A writes Tuple_A and Transcation_B writes Tuple_B. In this case, there are two rw-conflicts, and they make a cycle in the procedure graph of this schedule. Thus, this schedule has a serialization anomaly. i.e. Write-Skew.

![](https://img.vim-cn.com/03/b4ce6e467b63596761de878cf74bdf880c0a02.png)

### Implementing SSI in PostgresSQL

To realize the strategy described above, PostgreSQL has implemented many functions. However, here we uses only two data structures: SIREAD locks and rw-conflicts, to describe the SSI mechanism.

## SIREAD locks:

An SIREAD lock, internally called a predicate lock, is a pair of an object and (virtual) txids that store information about who has accessed with object. Note that the description of virtual txid is omitted. txid is used rather than virtual txid to simplify the following explanation.

`SIREAD` locks are crated by the `CheckTargetForConflictsOut` function whenever one `DNL` command is executed `SERIALIZABLE` mode. For example, if txid 100 reads Tuple_1 of given table, an `SIREAD` lock {Tuple_1, {100}} is created. If another transcation, e.g, txid 101, reads Tuple_1, the `SIREAD` lock is updated to `{Tuple_1, {100, 101}}`. Note that a `SIREAD` lock is also created when an index page is read when an index page is read.

`SIREAD` lock has three levels: tuple, page, and relation. If the `SIREAD` locks of all tuples within a single page are created, they are aggregated into a single `SIREAD` lock for that page, and all `SIREAD` locks of the associated tuples are released(removed), to reduce memory space. The same is true for all pages that are read.

When creating an `SIREAD` lock for an index, the page level `SIREAD` lock is created from the beginning. When using sequential scan, a relation level `SIREAD` lock is created from the beginning regardless of presence of indexes and/or WHERE clauses.  Note that ,in certain situations, this implementation can cause false-positive detection of serialization anomalies. 

A rw-conflict is a triplet of an `SIREAD` lock and two txids that reads and writes the `SIREAD` lock.

The `CheckTargetForConflictsIn` function is invoked whenever either an `INSERT, UPDATE or DELETE` command is executed in `SERIALIZABLE` mode, and it creates `rw-conflicts` when detecting conflicts by checking `SIREAD` locks. For example, assume that txid 100 reads Tuple_1 and then txid 101 updates Tuple_1. In this case, the `CheckTargetForConflictsIn` function, invoked by the `UPDATE` command in txid 101, detects a rw-conflict with Tuple_1 between txid 100 and 101 then creates a rw-conflict `{r=100, w=101, {Tuple_1}}`

The execution of concurrent SQL-transactions at isolation level SERIALIZABLE is guaranteed to be serializable. A serializable execution is defined to be an execution of the operations of concurrently executing SQL-transactions that produces the same effect as some serial execution of those same SQL-transactions.

### False-positive Serialization Anomalies(PG 9.5)

In `SERIALIZABLE` mode, the serializability of concurrent transactions is always fully guaranteed because false-negative serialization anomalies are never detected. However, under some circumstances, false-positive anomalies can be detected; therefore, users should keep this in mind when using `SERIALIZABLE` mode. In the following, the situation in which PostgreSQL detects false-positive anomalies are described.

![](https://img.vim-cn.com/ec/432b90d78742062b5d59c6967728174c673fc3.jpg
)

When using sequential scan, as mentioned in the explanation of `SIREAD` locks, PostgresSQL creates a relation level `SIREAD` lock. Next figure shows `SIREAD` locks and rw-conflicts when PostgreSQL uses sequential scan. In this case, `rw-conflicts` C1 and C2, which are associated with the tbl's `SIREAD` lock, and they create a cycle in precedence graph. Thus, a false-positive `Write-Skew` anomaly is detected (and either Tx_A or Tx_B will be aborted even though there is no conflict)

![](https://img.vim-cn.com/eb/4d1ea37880c7c6c4f202af7255ac76629174cf.png
)

## Required Maintenance Process

PostgreSQL's concurrency control mechanism requires the following maintenance processes.

1. Remove dead tuples and index tuples that point to corresponding dead tuples
2. Remove unnecessary parts of the clog
3. Freeze old txids
4. Update FSM, VM, and the statistics

## Freeing Processing

Here, we describe the txid wraparound problem.

Assume that tuple_1 is inserted with a txid of 100. i.e. The t_xmin Tuple_1 is 100. The server has been running for a very long period and Tuple_1 has not been modified. The current txid is 2.1 billion + 100 and a `SELECT` command is executed. At this time, Tuple_1 is visible because txid 100 is in the past. Then, the same `SELECT` command is execute; thus, the current txid is 2.1 billion + 101. However, Tuple_1 is no longer visible because txid 100 is in the future. This is so called transcation wraparound problem in PostgreSQL.

![](https://img.vim-cn.com/94/c8c4f5b80064bc12c9f9122519cca13e2599f5.png
)


To deal with this problem, PostgreSQL introduced a concept called frozen txid, and implemented a process called `FREEZE`.

In PostgreSQL, a frozen txid, which is a special reserved txid 2, is defined such that it is always older than all other txids. In other words, the frozen txid is always inactive and visible.

The freeze process is invoked by the vacuum process. The freeze process scans all tables files and rewrites the `t_xmin` of tuples to the frozen txid if the `t_xmin` value is older than the current txid minus the `vacuum_freeze_min_age`(the default is 50 million). 

For example, as can be seen in the next figure, the current txid is 50 million and the freeze process is invoked by the `VACUUM` command. In this case, the `t_xmin` of both `Tuple_1` and `Tuple_2` are rewritten to 2.

In Version 9.4 or later, the XMIN_FORZEN bit is set to the `t_infomask` field of tuples rather than rewriting the `t_xmin` of tuples to the frozen txid.


![](https://img.vim-cn.com/f7/5a71983e02da0ea032394264c7946ced0a2d4e.png
)

Please read 6.3.1