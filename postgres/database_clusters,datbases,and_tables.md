## Chap1 Database Cluster, Databases, and tables

A database is a collection of database objects. In the relational database theory, a database object is a data structure used either to store or to reference data. A(heap) **table** is a typical example of it, and there are many more like an index, a sequence, a view, a function and so on. In PostgresSQL, database themselves are also database objects and are logically separated from each other. All other database objects belong to their respective databases.

![](https://img.vim-cn.com/24/cb01d6415c0f7d2751255df32b3e807bfd19ea.png )

All the database objects in PostgresSQL are internally managed by respective object identifiers(OIDs), which are unsigned 4-byte integers. The relations between database objects and the respective OIDs  are stored in appropriate system catalogs, depending on the type of objects For example, OIDs of databases and heap tables are stored in `pg_database` and `pg_class` respectively, so you can find out the OIDs you want to know by issuing the queries such as the following:

```sql
postgres=# select datname, oid FROM pg_database where datname = 'postgres';
 datname  |  oid
----------+-------
 postgres | 13690
(1 row)

postgres=# select relname, oid from pg_class where relname = 'consumers';
  relname  |  oid
-----------+-------
 consumers | 16405
(1 row)
```

![an example of database cluster](https://img.vim-cn.com/73/17a4fddb2f79187c70f1444bb21adeece517be.png )

A database is a subdirectory under the `base` subdirectory; and the database directory names are identical to the respective OIDs. For example, when the OID of the database `sampledb` is 16384, its subdirectory name is 16384.

Each table or index whose size is less than 1GB is a single file stored under the database directory it belongs to. Tables and indexes as database objects are internally managed by individual OIDs, while those data files are managed by the variable, `reffilenode`. The `reffilenode` values of table and indexes basically but not always match the respective OIDs, the details are described below.

```sql
postgres=# select relname, oid, relfilenode FROM pg_class WHERE relname='consumers';
  relname  |  oid  | relfilenode
-----------+-------+-------------
 consumers | 16405 |       16405
(1 row)
```

the `refilename` values of tables and indexes are changed by issuing some commands(e.g `TRUNCATE`, `REDINDEX`, `CLUSTER`). For example, if we truncate the table `persons`, PG assigns a new `relfilenode` to the table, removes the old data file, and creates a new one.

```sql
test=# SELECT relname, oid, relfilenode FROM pg_class WHERE relname = 'persons';
 relname |  oid  | relfilenode
---------+-------+-------------
 persons | 24577 |       24577
(1 row)

test=# truncate persons;
TRUNCATE TABLE
test=# SELECT relname, oid, relfilenode FROM pg_class WHERE relname = 'persons';
 relname |  oid  | relfilenode
---------+-------+-------------
 persons | 24577 |       24583
(1 row)


test=# SELECT pg_relation_filepath('persons');
 pg_relation_filepath
----------------------
 base/24576/24583
(1 row)

```

Looking carefully at the database subdirectories, you will find out that each table has two associated files suffixed respective with '_fsm' and '_vm'. Those are referred to as **free space map** and **visibility map**, storing the information of the free space capacity and the visibility on each page within the table file, respectively. Indexes only have individual free space maps and don't have visibility map.

```sh
ceclinux@ceclinuxdeMacBook-Pro   /usr/local/var/postgres/base  ls  24576/*                                                                                                                                      ✔  8958  22:07:33    114.255.202.194
24576/112        24576/13530_fsm  24576/13550_fsm  24576/2600      24576/2607_fsm  24576/2616_fsm  24576/2658  24576/2682      24576/2753_vm   24576/2841      24576/3394      24576/3534      24576/3602_vm   24576/4146  24576/4166  24576/826
24576/113        24576/13530_vm   24576/13550_vm   24576/2600_fsm  24576/2607_vm   24576/2616_vm   24576/2659  24576/2683      24576/2754      24576/2995      24576/3394_fsm  24576/3541      24576/3603      24576/4147  24576/4167  24576/827
24576/1247       24576/13532      24576/13552      24576/2600_vm   24576/2608      24576/2617      24576/2660  24576/2684      24576/2755      24576/2996      24576/3394_vm   24576/3541_fsm  24576/3603_fsm  24576/4148  24576/4168  24576/828
24576/1247_fsm   24576/13534      24576/13554      24576/2601      24576/2608_fsm  24576/2617_fsm  24576/2661  24576/2685      24576/2756      24576/3079      24576/3395      24576/3541_vm   24576/3603_vm   24576/4149  24576/4169  24576/PG_VERSION
```

The may also be internally referred to as the forks of each relation; the free space map is the first fork of the table/index data file(the fork number is 1), the visibility map the second fork of the table's data file(the fork number is 2). The fork number of the data file is 0.


### Tablespaces

A Tablespace in PostgresSQL is an additional data area outside the base directory.

![](https://img.vim-cn.com/7a/bfb224a408f9253cd70ca6216b33cbf3f56d58.png )
