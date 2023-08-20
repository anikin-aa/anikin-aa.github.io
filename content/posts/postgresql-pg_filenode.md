---
title: "Who broke my pg_filenode.map?"
date: 2023-08-20T05:30:03+00:00
--- 

On one of the working days next problem with PostgreSQL was reported. 

Some of the applications were not able to connect to PostgreSQL with the next error:

```
FATAL: could not read file "base/12323695/pg_filenode.map": read 0 of 512 (SQLSTATE XX001)
```

According to [pgpedia](https://pgpedia.info/p/pg_filenode_map.html), that's a special file that contains relation information about the OID of a catalog and its filenode number.

There are two types of such files, global (per cluster) and for each of the logical databases.

After some investigation, I found out, that `pg_filenode.map` is broken for `template1` database. It was just empty! Strange, right? 

And that's how all of the newly created databases are affected by this.

When you issue 

```sql 
CREATE DATABASE <DB_NAME>
``` 

statement `pg_filenode.map` file is being copied from template database. In my case template database is a default one `template1`.


To check content of broken `pg_filenode.map` I used next utility: [pg_filedump](https://github.com/df7cb/pg_filedump). 

Just compile it and use it like that:

```
% ./pg_filedump -m pg_filenode.map

*******************************************************************
* PostgreSQL File/Block Formatted Dump Utility
*
* File: pg_filenode.map
* Options used: -m
*******************************************************************
Read 0 bytes, expected 512 
```

That's an output for the broken `pg_filenode.map` and here is for working one:

```
% ./pg_filedump -m pg_filenode2.map                                                                                                                      

*******************************************************************
* PostgreSQL File/Block Formatted Dump Utility
*
* File: pg_filenode2.map
* Options used: -m
*******************************************************************
Magic Number: 0x592717 (CORRECT)
Num Mappings: 17
Detailed Mappings list:
OID: 1259	Filenode: 1259
OID: 1249	Filenode: 1249
OID: 1255	Filenode: 1255
```

To repair `template0/1` databases, I re-created them with `postgres` database as a template. 

And dropped (via base folder delete) logical databases that had been created earlier.

As for now, why `pg_filenode.map` suddenly became broken is not clear. 

I tried to check the source code of PostgreSQL, but didn't find any suspicious parts when working with `pg_filenode.map`.

Maybe later I will update this post with the reason why it became broken.