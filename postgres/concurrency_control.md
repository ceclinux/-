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
*t_xmax* holds the txid of the transaction that deleted or updated this tuple. If this tuple has not been deleted or updated, t_xmax is set to 0, which means INVALID.
*t_cid* holds the command id(cid), which means how many SQL commands were executed before this command was executed within the current transaction beginning from 0. For example, assume that we execute three INSERT commands within a single transaction: 'BEGIN; INSERT; INSERT;INSERT; COMMIT;'. It the first command inserts this tuple, `t_cid` is set to 0. If the second command inserts this, `t_cid` is set to 1, and so on.




