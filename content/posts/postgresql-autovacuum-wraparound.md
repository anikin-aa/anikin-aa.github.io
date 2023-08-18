---
title: "PostgreSQL Autovacuum Wraparound"
date: 2023-08-18T05:30:03+00:00
--- 

I think, that every PostgreSQL user/administrator heard about `autovacuum` mechanism in PostgreSQL and got used to it in some way or another.

Long story short - a vacuum is a garbage collect for a particular logical database or table. https://www.postgresql.org/docs/current/routine-vacuuming.html


But! This topic is not about regular `autovacuum`, but about wraparound autovacuum:

```bash
autovacuum: VACUUM public.<table_name> (to prevent wraparound)
```

Kinda a strange description, right?

I've faced such autovacuum process on one of our DBs and was a little bit confused why it's running and locking the whole table.

It was running for several minutes and I decided to stop it somehow (because it was locking a critical table for a long time).

Without reading documentation and googling how to stop such autovacuum, I tried next things:

1. Turning `autovacuum` PostgreSQL parameter to `off` -> didn't help
2. Killing `autovacuum` process via `pg_terminate_backend()` -> didn't help

After that, I found two good articles:

* https://www.cybertec-postgresql.com/en/autovacuum-wraparound-protection-in-postgresql/
* https://www.percona.com/blog/overcoming-vacuum-wraparound/

Both of them helped me a lot to understand how and why `autovacuum to prevent wraparound` is triggered.

Every transaction in PostgreSQL got own ID (transaction ID). And number of transaction IDs is fixed due to the limitation of 32-bit number (2 ^ 32).

So, from time to time (actually based on `autovacuum_freeze_max_age`) PostgreSQL runs `autovacuum to prevent wraparound`.

Ok, we've got an idea about `autovacuum to prevent wraparound`, why and how it works, but, what should I do now if `autovacuum to prevent wraparound` is completely stuck? OK, I've got the steps:

1. Increase `autovacuum_freeze_max_age` to maximum value
2. Do pg_dump of table or database
3. Restore it with a new name 
4. Run vacuum: `vacuumdb -z -j 10 -v <db_name>`
5. Rename it back 

Also, it's worth monitoring `datfrozenxid` per logical database and adding alerting based on % of the wraparound risk.