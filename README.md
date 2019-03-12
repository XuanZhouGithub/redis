# Redis
Redis是一个开源的高性能key-value存储系统,有这些特点:

1. 高性能:https://redis.io/topics/benchmarks

2. 支持丰富的数据类型(string, hash, list, set, sorted set等)

3. 所有操作都是原子性的,支持事务

4. 可以通过AOF/RDB方式将内存数据保存到磁盘上保证持久存储

5. 支持主从同步

6. 支持publish/subscribe, notify等特性

## 1. ziplist
### 1.1 整体结构
ziplist是redis的基本数据结构,实际上是一个双向链表, redis的hash, list, set都基于ziplist实现,它的主要优点是省内存

struct | zlbytes | zltail | zllen | entry | entry | ... | entry | zlend
------ | :-----: | :----: | :---: | :---: | :---: | :-: | :---: | :---:
size   | uint32_t|uint32_t|uint16_t|      |       |     |       |uint8_t

其中:
zlbytes: 4字节,表示ziplist的大小(包括zlbytes的4个字节)

zltail: 4字节,表示最后一个节点在ziplist的偏移,能快速查找到尾部节点

zllen: 2字节,表示ziplist中的节点数目,也就是最多能表示2^16 - 1个节点,如果ziplist节点数超过2^16-1, zllen的值会被固定为2^16-1,只能通过遍历得到节点数目

entry: 存放数据的节点,长度不定

zlend: 1字节,固定为255

### 1.2 entry结构

<prevlen> <encoding> <entry-data>
  
prevlen: 1字节或5字节,表示前一个节点所占字节数, 方便ziplist向前遍历找到前项节点:

1. 如果前一个节点所占字节数小于254, prevlen就占1字节

2. 如果前一个节点所占字节数大于254, prevlen就需要5字节,第一个字节为254, 后面4字节用于表示前一节点大小

encoding字段很复杂:

1. 1字节, |00pppppp| : 前两bit为00, 后6bit为entry-data的长度len(< 64), entry-data为len字节的字符数组

   1字节, |11000000| : entry-data为int16_t的整数
   
   1字节, |11010000| : entry-data为int32_t的整数
   
   1字节, |11100000| : entry-data为int64_t的整数
   
   1字节, |11110000| : entry-data为24bit有符号整数
   
   1字节, |11111110| : entry-data为8bit有符号整数
   
   1字节, |1111xxxx| : entry-data为xxxx - 1, xxxx只有1-13是可用的,所以需要减一来表示0-12
   
   1字节, |11111111| : zlend
   
2. 2字节, |01pppppp|qqqqqqqq| : 前2bit为01, 后14bit为entry-data的长度len(< 2^14), entry-data为len字节的字符数组

3. 5字节, |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| : 第一个字节为10000000, 后4字节为entry-data的长度len(< 2^32), entry-data为len字节的字符数组

### 1.3 创建ziplist


struct | zlbytes | zltail | zllen | zlend
------ | :-----: | :----: | :---: | :---:
size   |  4Bytes | 4Bytes | 2Bytes|1Byte
value  | 1011    | 1010   | 0     |11111111

### 1.4 插入元素(复杂度O(N) ~ O(N^2))

struct | zlbytes | zltail | zllen | ... | prev | next | ... | zlend
------ | :-----: | :----: | :---: | :-: | :---:|:---: | :-: | :---:
size   |  4Bytes | 4Bytes | 2Bytes| ... | 9Bytes| ... | ... |1Byte
value  | 1011    | 1010   |1000000| ... | ... |  prev_entry_length: 1001<br>... | ... | 11111111

在prev,next之间插入"test"

struct | zlbytes | zltail | zllen | ... | prev | entry | next | ... | zlend
------ | :-----: | :----: | :---: | :-: | :---:|:-----:|:---: | :-: | :---:
size   |  4Bytes | 4Bytes | 2Bytes| ... | 9Bytes|6Bytes|  ... | ... |1Byte
value  | 1011    | 1010   |1000000| ... | ... | prev_entry_length: 1001<br> encoding: 00000100<br> entry-data: "test" | prev_entry_length: 1000<br>... | ...  | 11111111

插入元素(entry)时,需要将entry之后的节点移位,所以一般情况时,插入entry需要O(N)复杂度,由于插入元素时需要更新next的prev_entry_length的值,如果prev_entry_length所占大小由1字节变成5字节,那么next的长度发生变化,引起next->next的prev_entry_length, 最坏情况可能变成O(N^2)的连锁更新
### 1.5 删除元素(复杂度O(N) ~ O(N^2))
删除操作可以看成插入操作的逆操作,与插入类似,可能引起连锁更新

### 1.6 遍历

header | e1 | e2 | e3 | e4 | ... | zlend
:----: |----|----|----|----|-----|------

向后遍历: 比如指向e1的p开始,计算e1的长度(len1), (p+len1)即指向e2
向前遍历: 比如指向e3的p开始,读取e3中的prev_entry_length(len2), (p-len2)即指向e2

## 2 持久存储
Redis之所以性能好,读写速度快,是因为它的所有操作都基于内存,但内存的数据如果进程崩溃或系统重启就会丢失,所以数据持久化对于内存数据库很重要,它保证了数据库的可靠性, Redis提供了两种持久化方案,AOF及RDB(4.0开始支持AOF-RDB混合)
### 2.1 AOF(Append-only file)

#### 2.1.1 AOF流程
AOF实际上是一份执行日志,所有redis修改相关的命令追加到AOF文件中,通过回放这些命令就能恢复数据库,更新AOF文件的流程如图:

![Alt text](https://github.com/XuanZhouGit/Redis/blob/master/redis_aof.PNG)


Redis现在支持3种刷新策略:

1. AOF_FSYNC_NO :Write由主线程完成,不做fsync,只在redis被关闭或是AOF被关闭的时候进行fsync,写性能高但可靠性低,可能丢失上次fsync之后的数据

2. AOF_FSYNC_ALWAYS: Write和fsync都由主线程程完成, 每次都进行阻塞write和fsync,写性能低但可靠性高,最多丢失一条数据

3. AOF_FSYNC_EVERYSEC: Write在主线程完成,fsync在子线程非阻塞进行,2秒钟最多进行一次,最多丢失2秒的数据,写性能高且可靠性高

```
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    mstime_t latency;

    if (sdslen(server.aof_buf) == 0) return;

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        sync_in_progress = bioPendingJobsOfType(BIO_AOF_FSYNC) != 0;

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
        /* With this append fsync policy we do background fsyncing.
         * If the fsync is still in progress we can try to delay
         * the write for a couple of seconds. */
        if (sync_in_progress) {
            if (server.aof_flush_postponed_start == 0) {
                /* No previous write postponing, remember that we are
                 * postponing the flush and return. */
                server.aof_flush_postponed_start = server.unixtime;
                return;
            } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                /* We were already waiting for fsync to finish, but for less
                 * than two seconds this is still ok. Postpone again. */
                return;
            }
            /* Otherwise fall trough, and go write since we can't wait
             * over two seconds. */
            server.aof_delayed_fsync++;
            serverLog(LL_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }
    /* We want to perform a single write. This should be guaranteed atomic
     * at least if the filesystem we are writing is a real physical one.
     * While this will save us against the server being killed I don't think
     * there is much to do about the whole server stopping for power problems
     * or alike */

    latencyStartMonitor(latency);
    nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
    latencyEndMonitor(latency);
    /* We want to capture different events for delayed writes:
     * when the delay happens with a pending fsync, or with a saving child
     * active, and when the above two conditions are missing.
     * We also use an additional event name to save all samples which is
     * useful for graphing / monitoring purposes. */
    if (sync_in_progress) {
        latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
    } else if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) {
        latencyAddSampleIfNeeded("aof-write-active-child",latency);
    } else {
        latencyAddSampleIfNeeded("aof-write-alone",latency);
    }
    latencyAddSampleIfNeeded("aof-write",latency);

    /* We performed the write so reset the postponed flush sentinel to zero. */
    server.aof_flush_postponed_start = 0;

    if (nwritten != (ssize_t)sdslen(server.aof_buf)) {
        static time_t last_write_error_log = 0;
        int can_log = 0;

        /* Limit logging rate to 1 line per AOF_WRITE_LOG_ERROR_RATE seconds. */
        if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
            can_log = 1;
            last_write_error_log = server.unixtime;
        }

        /* Log the AOF write error and record the error code. */
        if (nwritten == -1) {
            if (can_log) {
                serverLog(LL_WARNING,"Error writing to the AOF file: %s",
                    strerror(errno));
                server.aof_last_write_errno = errno;
            }
        } else {
            if (can_log) {
                serverLog(LL_WARNING,"Short write while writing to "
                                       "the AOF file: (nwritten=%lld, "
                                       "expected=%lld)",
                                       (long long)nwritten,
                                       (long long)sdslen(server.aof_buf));
            }

            if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
                if (can_log) {
                    serverLog(LL_WARNING, "Could not remove short write "
                             "from the append-only file.  Redis may refuse "
                             "to load the AOF the next time it starts.  "
                             "ftruncate: %s", strerror(errno));
                }
            } else {
                /* If the ftruncate() succeeded we can set nwritten to
                 * -1 since there is no longer partial data into the AOF. */
                nwritten = -1;
            }
            server.aof_last_write_errno = ENOSPC;
        }

        /* Handle the AOF write error. */
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
            /* We can't recover when the fsync policy is ALWAYS since the
             * reply for the client is already in the output buffers, and we
             * have the contract with the user that on acknowledged write data
             * is synced on disk. */
            serverLog(LL_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            /* Recover from failed write leaving data into the buffer. However
             * set an error to stop accepting writes as long as the error
             * condition is not cleared. */
            server.aof_last_write_status = C_ERR;

            /* Trim the sds buffer if there was a partial write, and there
             * was no way to undo it with ftruncate(2). */
            if (nwritten > 0) {
                server.aof_current_size += nwritten;
                sdsrange(server.aof_buf,nwritten,-1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        /* Successful write(2). If AOF was in error state, restore the
         * OK state and log the event. */
        if (server.aof_last_write_status == C_ERR) {
            serverLog(LL_WARNING,
                "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = C_OK;
        }
    }
    server.aof_current_size += nwritten;

    /* Re-use AOF buffer when it is small enough. The maximum comes from the
     * arena size of 4k minus some overhead (but is otherwise arbitrary). */
    if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
        sdsclear(server.aof_buf);
    } else {
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. */
    if (server.aof_no_fsync_on_rewrite &&
        (server.aof_child_pid != -1 || server.rdb_child_pid != -1))
            return;

    /* Perform the fsync if needed. */
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* aof_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        latencyStartMonitor(latency);
        aof_fsync(server.aof_fd); /* Let's try to get this data on the disk */
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        server.aof_last_fsync = server.unixtime;
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
        if (!sync_in_progress) aof_background_fsync(server.aof_fd);
        server.aof_last_fsync = server.unixtime;
    }
}
```
#### 3.1.2 bgrewriteaof命令

bgrewriteaof命令用于执行AOF文件重写,用于创建一个当前AOF文件的体积优化版本,这个操作由子进程异步进行,不会阻塞主进程,且新的AOF会先写入到一个临时文件中,所以bgrewriteaof操作失败不会有数据丢失


```
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;
    long long start;

    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;
    if (aofCreatePipes() != C_OK) return C_ERR;
    openChildInfoPipe();
    start = ustime();
    if ((childpid = fork()) == 0) {
        char tmpfile[256];

        /* Child */
        closeListeningSockets(0);
        redisSetProcTitle("redis-aof-rewrite");
        snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int) getpid());
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
            size_t private_dirty = zmalloc_get_private_dirty(-1);

            if (private_dirty) {
                serverLog(LL_NOTICE,
                    "AOF rewrite: %zu MB of memory used by copy-on-write",
                    private_dirty/(1024*1024));
            }

            server.child_info_data.cow_size = private_dirty;
            sendChildInfo(CHILD_INFO_TYPE_AOF);
            exitFromChild(0);
        } else {
            exitFromChild(1);
        }
    } else {
        /* Parent */
        server.stat_fork_time = ustime()-start;
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
        if (childpid == -1) {
            closeChildInfoPipe();
            serverLog(LL_WARNING,
                "Can't rewrite append only file in background: fork: %s",
                strerror(errno));
            aofClosePipes();
            return C_ERR;
        }
        serverLog(LL_NOTICE,
            "Background append only file rewriting started by pid %d",childpid);
        server.aof_rewrite_scheduled = 0;
        server.aof_rewrite_time_start = time(NULL);
        server.aof_child_pid = childpid;
        updateDictResizePolicy();
        /* We set appendseldb to -1 in order to force the next call to the
         * feedAppendOnlyFile() to issue a SELECT command, so the differences
         * accumulated by the parent into server.aof_rewrite_buf will start
         * with a SELECT statement and it will be safe to merge. */
        server.aof_selected_db = -1;
        replicationScriptCacheFlush();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```
![Alt text](https://github.com/XuanZhouGit/Redis/blob/master/BGREWRITEAOF.PNG)

### 2.2 RDB

rdb是将内存中的数据以快照的形式保存到文件中,重启redis时加载rdb文件就能恢复数据库, 用户通过save,bgsave或flushall(save)命令来生成快照文件,那么save跟bgsave有什么区别呢:

![Alt text](https://github.com/XuanZhouGit/Redis/blob/master/redis_rdb.PNG)

save和bgsave实际上都是调用rdbSave来生成快照文件, 但是save操作是在主进程进行的,也就是save会阻塞主进程, 而bgSave是fork一个子进程,在子进程中进行rdbSave,并在save完成之后向主进程信号,通知主进程save完成,而主进程可以在rdbSave的时候继续处理用户请求
```
int rdbSaveBackground(char *filename, rdbSaveInfo *rsi) {
    pid_t childpid;
    long long start;

    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;

    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);
    openChildInfoPipe();

    start = ustime();
    if ((childpid = fork()) == 0) {
        int retval;

        /* Child */
        closeListeningSockets(0);
        redisSetProcTitle("redis-rdb-bgsave");
        retval = rdbSave(filename,rsi);
        if (retval == C_OK) {
            size_t private_dirty = zmalloc_get_private_dirty(-1);

            if (private_dirty) {
                serverLog(LL_NOTICE,
                    "RDB: %zu MB of memory used by copy-on-write",
                    private_dirty/(1024*1024));
            }

            server.child_info_data.cow_size = private_dirty;
            sendChildInfo(CHILD_INFO_TYPE_RDB);
        }
        exitFromChild((retval == C_OK) ? 0 : 1);
    } else {
        /* Parent */
        server.stat_fork_time = ustime()-start;
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
        if (childpid == -1) {
            closeChildInfoPipe();
            server.lastbgsave_status = C_ERR;
            serverLog(LL_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
            return C_ERR;
        }
        serverLog(LL_NOTICE,"Background saving started by pid %d",childpid);
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_pid = childpid;
        server.rdb_child_type = RDB_CHILD_TYPE_DISK;
        updateDictResizePolicy();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```

## 3 cluster

### 3.1 redis主流集群
redis集群目前有几种实现方式:
1. 客户端分片, jedis支持的,使用一致性hash

![Alt text](https://github.com/XuanZhouGit/Redis/blob/master/redis_sharding.PNG)

2. 基于代理的分片, Codis和Twemproxy

![Alt text](https://github.com/XuanZhouGit/Redis/blob/master/redis_proxy.PNG)

3. 路由查询, redis cluster(3.0版本开始支持)

![Alt text](https://github.com/XuanZhouGit/Redis/blob/master/redis_cluster.PNG)

这些实现方式有这些不同:

 cluster | Codis | Twemproxy | Redis Cluster
 :-----: | :---: | :-------: | :-----------:
 数据分片 | | consistency hash | hash slot
 resharding | Y | N | Y
 Pipeline | Y | Y | Y(只支持对单个node mset、mget、pipeline)
 multi-key when resharding | Y | | N
 hashtag for multi-key | Y | Y | Y
 client | any | any | smart client
 friendly to maintain | Y | N | N
 
### 3.2 数据分片
Redis cluster使用hash slot进行数据分片, 一个cluster有16384个hash slots, 所有的key都会被映射到某个slot上, 计算key对应的slot的函数为:
```
/* We have 16384 hash slots. The hash slot of a given key is obtained
 * as the least significant 14 bits of the crc16 of the key.
 *
 * However if the key contains the {...} pattern, only the part between
 * { and } is hashed. This may be useful in the future to force certain
 * keys to be in the same node (assuming no resharding is in progress). */
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e; /* start-end indexes of { and } */

    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    /* No '{' ? Hash the whole key. This is the base case. */
    if (s == keylen) return crc16(key,keylen) & 0x3FFF;

    /* '{' found? Check if we have the corresponding '}'. */
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    /* No '}' or nothing betweeen {} ? Hash the whole key. */
    if (e == keylen || e == s+1) return crc16(key,keylen) & 0x3FFF;

    /* If we are here there is both a { and a } on its right. Hash
     * what is in the middle between { and }. */
    return crc16(key+s+1,e-s-1) & 0x3FFF;
}
```
因为cluster支持对单个node支持multi-key的操作,所以cluster支持通过hash key将某些keys放到同一个node

集群中每个master负责部分slot,它负责的slots信息存在cluster.slots中:
```
typedef struct clusterNode {
    mstime_t ctime; /* Node object creation time. */
    char name[CLUSTER_NAMELEN]; /* Node name, hex string, sha1-size */
    int flags;      /* CLUSTER_NODE_... */
    uint64_t configEpoch; /* Last configEpoch observed for this node */
    unsigned char slots[CLUSTER_SLOTS/8]; /* slots handled by this node */
    int numslots;   /* Number of slots handled by this node */
    int numslaves;  /* Number of slave nodes, if this is a master */
    struct clusterNode **slaves; /* pointers to slave nodes */
    struct clusterNode *slaveof; /* pointer to the master node. Note that it
                                    may be NULL even if the node is a slave
                                    if we don't have the master node in our
                                    tables. */
    mstime_t ping_sent;      /* Unix time we sent latest ping */
    mstime_t pong_received;  /* Unix time we received the pong */
    mstime_t fail_time;      /* Unix time when FAIL flag was set */
    mstime_t voted_time;     /* Last time we voted for a slave of this master */
    mstime_t repl_offset_time;  /* Unix time we received offset for this node */
    mstime_t orphaned_time;     /* Starting time of orphaned master condition */
    long long repl_offset;      /* Last known repl offset for this node. */
    char ip[NET_IP_STR_LEN];  /* Latest known IP address of this node */
    int port;                   /* Latest known clients port of this node */
    int cport;                  /* Latest known cluster port of this node. */
    clusterLink *link;          /* TCP/IP link with this node */
    list *fail_reports;         /* List of nodes signaling this as failing */
} clusterNode;
```
而集群中slots的分配信息存在ClusterState.slots中:
```
typedef struct clusterState {
    clusterNode *myself;  /* This node */
    uint64_t currentEpoch;
    int state;            /* CLUSTER_OK, CLUSTER_FAIL, ... */
    int size;             /* Num of master nodes with at least one slot */
    dict *nodes;          /* Hash table of name -> clusterNode structures */
    dict *nodes_black_list; /* Nodes we don't re-add for a few seconds. */
    clusterNode *migrating_slots_to[CLUSTER_SLOTS];
    clusterNode *importing_slots_from[CLUSTER_SLOTS];
    clusterNode *slots[CLUSTER_SLOTS];
    uint64_t slots_keys_count[CLUSTER_SLOTS];
    rax *slots_to_keys;
    /* The following fields are used to take the slave state on elections. */
    mstime_t failover_auth_time; /* Time of previous or next election. */
    int failover_auth_count;    /* Number of votes received so far. */
    int failover_auth_sent;     /* True if we already asked for votes. */
    int failover_auth_rank;     /* This slave rank for current auth request. */
    uint64_t failover_auth_epoch; /* Epoch of the current election. */
    int cant_failover_reason;   /* Why a slave is currently not able to
                                   failover. See the CANT_FAILOVER_* macros. */
    /* Manual failover state in common. */
    mstime_t mf_end;            /* Manual failover time limit (ms unixtime).
                                   It is zero if there is no MF in progress. */
    /* Manual failover state of master. */
    clusterNode *mf_slave;      /* Slave performing the manual failover. */
    /* Manual failover state of slave. */
    long long mf_master_offset; /* Master offset the slave needs to start MF
                                   or zero if stil not received. */
    int mf_can_start;           /* If non-zero signal that the manual failover
                                   can start requesting masters vote. */
    /* The followign fields are used by masters to take state on elections. */
    uint64_t lastVoteEpoch;     /* Epoch of the last vote granted. */
    int todo_before_sleep; /* Things to do in clusterBeforeSleep(). */
    /* Messages received and sent by type. */
    long long stats_bus_messages_sent[CLUSTERMSG_TYPE_COUNT];
    long long stats_bus_messages_received[CLUSTERMSG_TYPE_COUNT];
    long long stats_pfail_nodes;    /* Number of nodes in PFAIL status,
                                       excluding nodes without address. */
} clusterState;
```
当集群添加或删除节点时,只需要移动相应的slot就行了, 比如:
当添加一个新节点N + 1, 只需要从0...N节点中移动一些slots给N+1
当删除一个节点 N, 只需要将节点N的slots移动到0...N - 1,再删除slot N

### 3.3 handshake
Redis cluster一般由多个节点组成,这些节点开始是相互独立的,需要将这些节点连接起来, 组成集群, 向某个node发送:

```CLUSTER MEET <ip> <port> [cport]```

node就会与指定ip, port的节点进行握手,握手成功,这个ip,port的节点就加入node所在的集群,过程如图:

![Alt text](https://github.com/XuanZhouGit/Redis/blob/master/handshake.PNG)

(1) client向node1发送CLUSTER MEET,将node2的ip, port发给node1

(2) node1会使用,ip,port创建一个cluster node,加到自己的clusternodes里,并标记状态为CLUSTER_NODE_HANDSHAKE, ,并向client发送ok message

(3) 在node1的clusterCron里会遍历所有的cluster node, 向CLUSTER_NODE_HANDSHAKE状态的node发送CLUSTERMSG_TYPE_MEET,并将node的clusterReadHandler加入事件循环

(4) node2接收到CLUSTERMSG_TYPE_MEET,会向node1返回CLUSTERMSG_TYPE_PONG,node2就加入了node1所在的集群,而node2的信息是通过msg中的gossip fileds进行传播的

### 3.4 clusterCron
每个redis节点每100ms会clusterCron执行这些操作:

(1) 向未建立TCP连接的节点发送ping或meet(handshake中的步骤(3))

(2) keepalive: 

 (2.1) 每秒钟随机从已经建立连接的5个节点中选1个pong_received最久的节点发送ping
 
 (2.2) 对等待pong时间超过timeout/2的节点断开tcp连接,将通过步骤(1)进行重连
 
 (2.3) 对已经接受pong超过timeout/2但没有发送ping的节点发送ping
 
(3) 如果这个节点是slave节点, 且其master有最多的non failing slave节点,就尝试将这个slave移给cluster中的孤儿master(clusterHandleSlaveMigration):

```
/* -----------------------------------------------------------------------------
 * CLUSTER slave migration
 *
 * Slave migration is the process that allows a slave of a master that is
 * already covered by at least another slave, to "migrate" to a master that
 * is orpaned, that is, left with no working slaves.
 * ------------------------------------------------------------------------- */

/* This function is responsible to decide if this replica should be migrated
 * to a different (orphaned) master. It is called by the clusterCron() function
 * only if:
 *
 * 1) We are a slave node.
 * 2) It was detected that there is at least one orphaned master in
 *    the cluster.
 * 3) We are a slave of one of the masters with the greatest number of
 *    slaves.
 *
 * This checks are performed by the caller since it requires to iterate
 * the nodes anyway, so we spend time into clusterHandleSlaveMigration()
 * if definitely needed.
 *
 * The fuction is called with a pre-computed max_slaves, that is the max
 * number of working (not in FAIL state) slaves for a single master.
 *
 * Additional conditions for migration are examined inside the function.
 */
void clusterHandleSlaveMigration(int max_slaves)
```

(4) 如果本节点是master节点,向自己执行failover的slave节点发送ping,更新flag

(5) 将超时未收到pong的节点设置为CLUSTER_NODE_PFAIL

(6) 如果本节点是slave节点,但没开始从master进行复制,更新master信息并设置复制信息

(7) 检查manual failover是不是已经超时,如果已经超时,重置manual failover(3.4会详细介绍failover)

(8) slave节点在必要的时候处理failover(3.4会详细介绍failover)

(9) 更新节点状态

### 3.4 failover
#### 3.4.1 slave failover
##### 3.4.1.1 gossip field
gossip filed是节点通信的重要部分,主要用于消息广播及更新, gossip主要包含两种消息:

(1) node连接的随机10%的节点,收到的节点如果发现没有连接这些节点,就会与这些节点进行handshake,这样新加入的节点能快速的连接集群中其它节点

(2) 被标记了CLUSTER_NODE_PFAIL的节点, 收到的节点会统计这个节点被多少其它节点标记成了CLUSTER_NODE_PFAIL,当数目超过(server.cluster->size / 2) + 1,这个这点会被标记为CLUSTER_NODE_FAIL, 如果收到的节点是master节点,它会向整个集群广播这个节点的CLUSTERMSG_TYPE_FAIL信息, 让集群中所有节点把这个节点标记为CLUSTER_NODE_FAIL

##### 3.4.1.2 failover flow
当slave node发现其master node的状态为CLUSTER_NODE_FAIL,就会进行failover,希望成为新的master, failover会通过一次选举过程,让集群中其它master选出新的master:

(1) slave发现其master状态为为CLUSTER_NODE_FAIL

(2) slave会根据自己的offset在slave中的data offset延迟+随机时间做为failover_auth_time开始发起选举,这样数据较新的slave更可能成为新master,减少数据丢失

(3) 到了failover_auth_time, slave会将server.cluster->currentEpoch++, 并向集群广播CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST

(4) 当集群中的其它master接到CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST, 会检测请求中的configEpoch(也就是failover的master的Epoch)是不是比master服务的slots的configEpoch更新, 及currentEpoch是不是比当前的server.cluster->currentEpoch更新(master只接受更新的请求),且该master没有对currentEpoch进行过投票, master会向请求的slave发送CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK

(5) 当slave收到了超过半数的CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK, slave会成为新的master

(6) 新的master更新相关信息, 并向集群广播更新

slave handle flow:

```
/* This function is called if we are a slave node and our master serving
 * a non-zero amount of hash slots is in FAIL state.
 *
 * The gaol of this function is:
 * 1) To check if we are able to perform a failover, is our data updated?
 * 2) Try to get elected by masters.
 * 3) Perform the failover informing all the other nodes.
 */
void clusterHandleSlaveFailover(void) {
    mstime_t data_age;
    mstime_t auth_age = mstime() - server.cluster->failover_auth_time;
    int needed_quorum = (server.cluster->size / 2) + 1;
    int manual_failover = server.cluster->mf_end != 0 &&
                          server.cluster->mf_can_start;
    mstime_t auth_timeout, auth_retry_time;

    server.cluster->todo_before_sleep &= ~CLUSTER_TODO_HANDLE_FAILOVER;

    /* Compute the failover timeout (the max time we have to send votes
     * and wait for replies), and the failover retry time (the time to wait
     * before trying to get voted again).
     *
     * Timeout is MAX(NODE_TIMEOUT*2,2000) milliseconds.
     * Retry is two times the Timeout.
     */
    auth_timeout = server.cluster_node_timeout*2;
    if (auth_timeout < 2000) auth_timeout = 2000;
    auth_retry_time = auth_timeout*2;

    /* Pre conditions to run the function, that must be met both in case
     * of an automatic or manual failover:
     * 1) We are a slave.
     * 2) Our master is flagged as FAIL, or this is a manual failover.
     * 3) We don't have the no failover configuration set, and this is
     *    not a manual failover.
     * 4) It is serving slots. */
    if (nodeIsMaster(myself) ||
        myself->slaveof == NULL ||
        (!nodeFailed(myself->slaveof) && !manual_failover) ||
        (server.cluster_slave_no_failover && !manual_failover) ||
        myself->slaveof->numslots == 0)
    {
        /* There are no reasons to failover, so we set the reason why we
         * are returning without failing over to NONE. */
        server.cluster->cant_failover_reason = CLUSTER_CANT_FAILOVER_NONE;
        return;
    }

    /* Set data_age to the number of seconds we are disconnected from
     * the master. */
    if (server.repl_state == REPL_STATE_CONNECTED) {
        data_age = (mstime_t)(server.unixtime - server.master->lastinteraction)
                   * 1000;
    } else {
        data_age = (mstime_t)(server.unixtime - server.repl_down_since) * 1000;
    }

    /* Remove the node timeout from the data age as it is fine that we are
     * disconnected from our master at least for the time it was down to be
     * flagged as FAIL, that's the baseline. */
    if (data_age > server.cluster_node_timeout)
        data_age -= server.cluster_node_timeout;

    /* Check if our data is recent enough according to the slave validity
     * factor configured by the user.
     *
     * Check bypassed for manual failovers. */
    if (server.cluster_slave_validity_factor &&
        data_age >
        (((mstime_t)server.repl_ping_slave_period * 1000) +
         (server.cluster_node_timeout * server.cluster_slave_validity_factor)))
    {
        if (!manual_failover) {
            clusterLogCantFailover(CLUSTER_CANT_FAILOVER_DATA_AGE);
            return;
        }
    }

    /* If the previous failover attempt timedout and the retry time has
     * elapsed, we can setup a new one. */
    if (auth_age > auth_retry_time) {
        server.cluster->failover_auth_time = mstime() +
            500 + /* Fixed delay of 500 milliseconds, let FAIL msg propagate. */
            random() % 500; /* Random delay between 0 and 500 milliseconds. */
        server.cluster->failover_auth_count = 0;
        server.cluster->failover_auth_sent = 0;
        server.cluster->failover_auth_rank = clusterGetSlaveRank();
        /* We add another delay that is proportional to the slave rank.
         * Specifically 1 second * rank. This way slaves that have a probably
         * less updated replication offset, are penalized. */
        server.cluster->failover_auth_time +=
            server.cluster->failover_auth_rank * 1000;
        /* However if this is a manual failover, no delay is needed. */
        if (server.cluster->mf_end) {
            server.cluster->failover_auth_time = mstime();
            server.cluster->failover_auth_rank = 0;
        }
        serverLog(LL_WARNING,
            "Start of election delayed for %lld milliseconds "
            "(rank #%d, offset %lld).",
            server.cluster->failover_auth_time - mstime(),
            server.cluster->failover_auth_rank,
            replicationGetSlaveOffset());
        /* Now that we have a scheduled election, broadcast our offset
         * to all the other slaves so that they'll updated their offsets
         * if our offset is better. */
        clusterBroadcastPong(CLUSTER_BROADCAST_LOCAL_SLAVES);
        return;
    }

    /* It is possible that we received more updated offsets from other
     * slaves for the same master since we computed our election delay.
     * Update the delay if our rank changed.
     *
     * Not performed if this is a manual failover. */
    if (server.cluster->failover_auth_sent == 0 &&
        server.cluster->mf_end == 0)
    {
        int newrank = clusterGetSlaveRank();
        if (newrank > server.cluster->failover_auth_rank) {
            long long added_delay =
                (newrank - server.cluster->failover_auth_rank) * 1000;
            server.cluster->failover_auth_time += added_delay;
            server.cluster->failover_auth_rank = newrank;
            serverLog(LL_WARNING,
                "Slave rank updated to #%d, added %lld milliseconds of delay.",
                newrank, added_delay);
        }
    }

    /* Return ASAP if we can't still start the election. */
    if (mstime() < server.cluster->failover_auth_time) {
        clusterLogCantFailover(CLUSTER_CANT_FAILOVER_WAITING_DELAY);
        return;
    }

    /* Return ASAP if the election is too old to be valid. */
    if (auth_age > auth_timeout) {
        clusterLogCantFailover(CLUSTER_CANT_FAILOVER_EXPIRED);
        return;
    }

    /* Ask for votes if needed. */
    if (server.cluster->failover_auth_sent == 0) {
        server.cluster->currentEpoch++;
        server.cluster->failover_auth_epoch = server.cluster->currentEpoch;
        serverLog(LL_WARNING,"Starting a failover election for epoch %llu.",
            (unsigned long long) server.cluster->currentEpoch);
        clusterRequestFailoverAuth();
        server.cluster->failover_auth_sent = 1;
        clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                             CLUSTER_TODO_UPDATE_STATE|
                             CLUSTER_TODO_FSYNC_CONFIG);
        return; /* Wait for replies. */
    }

    /* Check if we reached the quorum. */
    if (server.cluster->failover_auth_count >= needed_quorum) {
        /* We have the quorum, we can finally failover the master. */

        serverLog(LL_WARNING,
            "Failover election won: I'm the new master.");

        /* Update my configEpoch to the epoch of the election. */
        if (myself->configEpoch < server.cluster->failover_auth_epoch) {
            myself->configEpoch = server.cluster->failover_auth_epoch;
            serverLog(LL_WARNING,
                "configEpoch set to %llu after successful failover",
                (unsigned long long) myself->configEpoch);
        }

        /* Take responsability for the cluster slots. */
        clusterFailoverReplaceYourMaster();
    } else {
        clusterLogCantFailover(CLUSTER_CANT_FAILOVER_WAITING_VOTES);
    }
}
```

master handle flow:

```
/* Vote for the node asking for our vote if there are the conditions. */
void clusterSendFailoverAuthIfNeeded(clusterNode *node, clusterMsg *request) {
    clusterNode *master = node->slaveof;
    uint64_t requestCurrentEpoch = ntohu64(request->currentEpoch);
    uint64_t requestConfigEpoch = ntohu64(request->configEpoch);
    unsigned char *claimed_slots = request->myslots;
    int force_ack = request->mflags[0] & CLUSTERMSG_FLAG0_FORCEACK;
    int j;

    /* IF we are not a master serving at least 1 slot, we don't have the
     * right to vote, as the cluster size in Redis Cluster is the number
     * of masters serving at least one slot, and quorum is the cluster
     * size + 1 */
    if (nodeIsSlave(myself) || myself->numslots == 0) return;

    /* Request epoch must be >= our currentEpoch.
     * Note that it is impossible for it to actually be greater since
     * our currentEpoch was updated as a side effect of receiving this
     * request, if the request epoch was greater. */
    if (requestCurrentEpoch < server.cluster->currentEpoch) {
        serverLog(LL_WARNING,
            "Failover auth denied to %.40s: reqEpoch (%llu) < curEpoch(%llu)",
            node->name,
            (unsigned long long) requestCurrentEpoch,
            (unsigned long long) server.cluster->currentEpoch);
        return;
    }

    /* I already voted for this epoch? Return ASAP. */
    if (server.cluster->lastVoteEpoch == server.cluster->currentEpoch) {
        serverLog(LL_WARNING,
                "Failover auth denied to %.40s: already voted for epoch %llu",
                node->name,
                (unsigned long long) server.cluster->currentEpoch);
        return;
    }

    /* Node must be a slave and its master down.
     * The master can be non failing if the request is flagged
     * with CLUSTERMSG_FLAG0_FORCEACK (manual failover). */
    if (nodeIsMaster(node) || master == NULL ||
        (!nodeFailed(master) && !force_ack))
    {
        if (nodeIsMaster(node)) {
            serverLog(LL_WARNING,
                    "Failover auth denied to %.40s: it is a master node",
                    node->name);
        } else if (master == NULL) {
            serverLog(LL_WARNING,
                    "Failover auth denied to %.40s: I don't know its master",
                    node->name);
        } else if (!nodeFailed(master)) {
            serverLog(LL_WARNING,
                    "Failover auth denied to %.40s: its master is up",
                    node->name);
        }
        return;
    }

    /* We did not voted for a slave about this master for two
     * times the node timeout. This is not strictly needed for correctness
     * of the algorithm but makes the base case more linear. */
    if (mstime() - node->slaveof->voted_time < server.cluster_node_timeout * 2)
    {
        serverLog(LL_WARNING,
                "Failover auth denied to %.40s: "
                "can't vote about this master before %lld milliseconds",
                node->name,
                (long long) ((server.cluster_node_timeout*2)-
                             (mstime() - node->slaveof->voted_time)));
        return;
    }

    /* The slave requesting the vote must have a configEpoch for the claimed
     * slots that is >= the one of the masters currently serving the same
     * slots in the current configuration. */
    for (j = 0; j < CLUSTER_SLOTS; j++) {
        if (bitmapTestBit(claimed_slots, j) == 0) continue;
        if (server.cluster->slots[j] == NULL ||
            server.cluster->slots[j]->configEpoch <= requestConfigEpoch)
        {
            continue;
        }
        /* If we reached this point we found a slot that in our current slots
         * is served by a master with a greater configEpoch than the one claimed
         * by the slave requesting our vote. Refuse to vote for this slave. */
        serverLog(LL_WARNING,
                "Failover auth denied to %.40s: "
                "slot %d epoch (%llu) > reqEpoch (%llu)",
                node->name, j,
                (unsigned long long) server.cluster->slots[j]->configEpoch,
                (unsigned long long) requestConfigEpoch);
        return;
    }

    /* We can vote for this slave. */
    server.cluster->lastVoteEpoch = server.cluster->currentEpoch;
    node->slaveof->voted_time = mstime();
    clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_FSYNC_CONFIG);
    clusterSendFailoverAuth(node);
    serverLog(LL_WARNING, "Failover auth granted to %.40s for epoch %llu",
        node->name, (unsigned long long) server.cluster->currentEpoch);
}

```

#### 3.4.2 manual failover
manual failover是一种运维功能,由client通过CLUSTER FAILOVER command手动将slave设置为master节点:

```
CLUSTER FAILOVER [FORCE|TAKEOVER]
```

force:设置mf_can_start = 1, 跳过3.4.1.2(1)(2), 执行(3) ~ (6)

takeover:跳过3.4.1.2(1) ~ (5),执行(6)



