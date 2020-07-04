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

Tablespaces in PostgresSQL allow databases administrators to define locations in the file system where the files representing database objects can be stored.Once created, a tablespace can be referred to by name when creating database objects.

By using tablespaces, an administrator can control the disk layout of a PostgresSQL installation. This is useful in at least two ways. First, if the partition or volume on which the cluster was initialized runs out of space and cannot be extended, a tablespace can be created on a different partition and used until the system can be reconfigured.

Second, tablespaces allow an administrator to use knowledge of the usage pattern of database objects to optimize performance. For example, an index which is very heavily used can be placed on a very fast, highly available disk, such as expensive solid state device. At the same time a table storing archived data which is rarely used or not performed critical could be stored on a less expensive, slower disk system.

A Tablespace in PostgresSQL is an additional data area outside the base directory.

![](https://img.vim-cn.com/7a/bfb224a408f9253cd70ca6216b33cbf3f56d58.png )

Inside the data file(heap table and index, as well as the free space map and visibility map), it is divided into pages(or blocks) of fixed length, the default is 8192 byte(8 KB). Those pages with each file are numbered sequentially from 0, and such numbers are called as **block numbers**. If the file has been filled up, Postgres adds a new empty page to the end of the file to increate the file size. 

Internal layout of pages depends on the data file types. In this section, the table layout is described as the information will be required the following chapters.

A page within a table contains three kinds of data described as follows:

1. **heap tuple** - A heap tuple is a record data itself. They are stacked in order from the bottom of the page.

2. **line pointer** - A line pointer is 4 byte long and holds a pointer to each heap tuple. It is also called an item pointer.

Line pointers form a simple array, which plays the role of index to the tuples. Each index is numbered sequentially from 1, and called offset number. When a new tuple is added to the page, a new line pointer is also pushed n the the array to point to the new one.
   
3. **header data** -- A header data defined by the structure PageHeaderData is allocated in the beginning of the page. It is 24 byte long and contains general information about the page. The major variables of the structure are described below.
   1. `pd_lsn` - This variable store the LSN of XLOG record written by the last change of the page. It is an 8-byte unsigned integer, related to the WAL.
   2. `pg_checksum` - This variable store the checksum value of this page.
   3. `pd_lower, pd_upper`  points to the end of line pointers, and `pd_upper` to the beginning of the newest heap tuple.
   4. `pd_special` - This variable is for indexes. In the page within tables, it points to the end of the page. (In the page within indexes, it points to the beginning of special space which is the data area held only by the indexes and contains the particular data according the the kind of index types such as B-tree, Gist, Gin)

An empty space between the end of line pointers and the beginning of the newest tuple is referred to as **free space or hole**.

To identity a tuple with the table, **tuple identifier** is internally used. A TID comprises a pair of values: the block number that contains the tuple, and the offset number of the line pointer that points to the tuple.

In addition, heap tuple whose size is greater than about 2KB (about 1/4 of 8KB) is stored and managed using a method called **TOAST**

![](https://img.vim-cn.com/38/9f35e56e927269dc9dcd58f86ef69f5a87f1b6.png )

### The methods of writing and reading tuples

#### Write Heap tuples

Suppose a table composed of one page which contains just one heap tuple. The `pd_lower` of this page point to the first line pointer, and both the line pointer and the `pd_upper` point to the first heap tuple.

When the second tuple is inserted, it is placed after the first one. The second line pointer is pushed onto the first one, and it points to the second tuple. The `pd_lower` changes to point to the second line pointer, and the `pd_upper` to the second heap tuple.

![](https://img.vim-cn.com/f5/cf9d0b4cb8189a8cefae6628950d9c65d8babd.png)

#### Read Heap tables

1. **Sequential scan** - All tuples in all pages are sequentially read by scanning all line pointers in each page.
2. **B-tree index scan** - An index file contains index tuples, each of which is composed of an index key and a TID pointing to the target heap tuple. If the index tuple with the key that you are looking for has been found, PostgresSQL reads the desired tuple using the obtained TID value. For example, in the below figure, TID value of the obtained index tuple is  `(block = 7, Offset = 2)`. It means that the target heap tuple is 2nd tuple in the 7th page within the table, so PostgresSQL can read the desired heap tuple without unnecessary scanning in the pages.

![](//img.vim-cn.com/c1/3121c81cc0542a1a61486cbe11767ba77ef640.png)