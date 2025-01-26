---
layout: post
title: "Redis Notes"
date: 2024-1-25
tags: [web]
comments: true
author: Alwyn Tan
---

- [Redis presistence](#redis-presistence)
  - [RDB](#rdb)
    - [Snapshotting](#snapshotting)
    - [RDB pros and cons](#rdb-pros-and-cons)
  - [AOF](#aof)
    - [AOF rewriting](#aof-rewriting)
    - [AOF pros and cons](#aof-pros-and-cons)


## Redis presistence
### RDB
Redis Database performs point-in-time snapshots of the dataset at specified intervals.
#### Snapshotting
By default, Redis saves snapshots in a binary file called dump.rdb. Redis forks a child process to write the dataset to a temporary RDB file, and when the child is done, the new RDB file will replace the old one.
To configure it, use SAVE or BGSAVE like this:
`save 60 1000`
#### RDB pros and cons
<div style="border: 2px solid #43CD80; margin: 10px 0; padding: 5px; border-radius: 15px; background-color: #9AFF9A">
RDB Advantage:

+ RDB is a very compatct single-file point-in-time representation of Redis data, which is perfect for backups. Allowing you to easily restore different version of the dataset in case of disasters.
+ Good performances and faster restarts with big datasets compared to AOF
</div>

<div style="border: 2px solid #CD2626; padding:5px;margin: 10px 0;border-radius: 15px; background-color: #FF7256">
RDB Disadvantage:

+ Will lose latest data, so it is NOT good if you need to minimize the chance of data loss
+ Fork can be time consuming when the dataset is too big
</div>

### AOF
#### AOF rewriting
To compact AOF, Redis is able to rebuild the AOF in the background without interrupting service to clients.
When *bgrewriteaof* is issued, Redis will write the shortest sequence of commands needed to rebuild the current dataset.
#### AOF pros and cons
<div style="border: 2px solid #43CD80; margin: 10px 0; padding: 5px; border-radius: 15px; background-color: #9AFF9A">
AOF Advantage:

+ AOF is much more durable: different fsync policies are provided: no fsync, every second and every query. So is can save the data with high integrity。
+ AOF rewriting is performed by child process *bgrewriteaof*, letting parent goes well.
+ AOF records Redis operation so it is easier to understand exactly how the data is manipulated.
</div>

<div style="border: 2px solid #CD2626; padding:5px;margin: 10px 0;border-radius: 15px; background-color: #FF7256">
AOF Disadvantage:

+ Usually bigger than the equivalent RDB file

+ Fork can be time consuming when the dataset is too big
</div>
