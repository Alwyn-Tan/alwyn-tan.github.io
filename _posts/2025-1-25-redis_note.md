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
- [Expiration Algorithm](#expiration-algorithm)
  - [Expiring Commands](#expiring-commands)
- [Eviction policy](#eviction-policy)


## Redis presistence
### RDB
Redis Database performs point-in-time snapshots of the dataset at specified intervals.
#### Snapshotting
By default, Redis saves snapshots in a binary file called dump.rdb. Redis forks a child process to write the dataset to a temporary RDB file, and when the child is done, the new RDB file will replace the old one.
To configure it, use SAVE or BGSAVE like this:
`save 60 1000`
#### RDB pros and cons
<div style="border: 2px solid #43CD80; margin: 10px 0; padding: 5px; border-radius: 15px; background-color: #9AFF9A">
RDB Advantage:<br>

● RDB is a very compatct single-file point-in-time representation of Redis data, which is perfect for backups. Allowing you to easily restore different version of the dataset in case of disasters.<br>  

● Good performances and faster restarts with big datasets compared to AOF.
</div>

<div style="border: 2px solid #CD2626; padding:5px;margin: 10px 0;border-radius: 15px; background-color: #FF7256">
RDB Disadvantage:<br>

●  Will lose latest data, so it is NOT good if you need to minimize the chance of data loss<br>  

●  Fork can be time consuming when the dataset is too big
</div>

### AOF
#### AOF rewriting
To compact AOF, Redis is able to rebuild the AOF in the background without interrupting service to clients.
When *bgrewriteaof* is issued, Redis will write the shortest sequence of commands needed to rebuild the current dataset.
#### AOF pros and cons
<div style="border: 2px solid #43CD80; margin: 10px 0; padding: 5px; border-radius: 15px; background-color: #9AFF9A">
AOF Advantage:<br>

●  AOF is much more durable: different fsync policies are provided: no fsync, every second and every query. So is can save the data with high integrity.<br>

●  AOF rewriting is performed by child process *bgrewriteaof*, letting parent goes well.<br>

●  AOF records Redis operation so it is easier to understand exactly how the data is manipulated.
</div>

<div style="border: 2px solid #CD2626; padding:5px;margin: 10px 0;border-radius: 15px; background-color: #FF7256">
AOF Disadvantage:<br>

●  Usually bigger than the equivalent RDB file<br>

●  Fork can be time consuming when the dataset is too big
</div>

## Expiration Algorithm
### Expiring Commands
```
PRESIST # Removes the expiration
TTL / PTTL # Returns the amount of time remaining time
EXPIRE / PEXPIRE # Sets the key to expire in the given time
EXPIREAT / PEXPIREAT # Exact expiration timestamp
```
We can see that Redis keys are expired in two ways:

● **Passive way:**  Redis checks whether a key is expired when it is accessed, and then the expired key will be deleted.

● **Active way:** Redis periodically tests a few keys by random sampling among keys with an expired set.

<div style="border: 2px solid #00868B; margin: 10px 0; padding: 5px; border-radius: 15px; background-color: #00E5EE">
But the passive way will make expired key fewer and fewer, which means we do resource-intensive operation while do not produce a relevant number of evictions.<br>

Because of this, the random sampling algorithm would stop as a threshold of 25%.<br>

In addition, to further improve memory usability, if a sample is about to expire, it will be stored in a radix tree. Then the sampling iteration will begin from the radix to delete keys that are more likely to expire first
</div>

## Eviction policy
Redis eviction policy determines what happens when a database reaches its memory limit.<br>
| Policy | Detail |
| :-----:| :-----:|
|noeviction|new value are not saved|
|allkeys-lru|keeps most recently used keys; removes least recently used keys|
|allkeys-lfu|keeps most frequently used keys; removes least frequently used keys|
|allkeys-random|...|
|volatile-lru, volatile-lfu, volatile-lfu| keys with expire filed set to true|
|volatile-ttl|removes LFU keys with shortest TTL value|

Generally, volatile-lru is the default eviction policy for most databases.









