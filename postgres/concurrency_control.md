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

