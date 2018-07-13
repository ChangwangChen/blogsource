---
title: Redis源码--List实现分析
date: 2018-07-05 21:48:15
categories: C 
tags: [C, Redis]
---


## 前言

Redis的学习总结文章我大概分为以下四个部分:

- 数据结构
- 网络和IO
- Redis基本功能
- 其他

来进行讲述. 这是数据结构部分`List链表`的实现.

Redis实现List使用的是双向链表， 所以在 List 的头尾节点都能进行操作。我们来看下 List 的基本数据结构。

<!--more-->

```c
//List数据结构
typedef struct list {
    listNode *head; //头指针
    listNode *tail; //尾指针
    void *(*dup)(void *ptr); // value 复制函数指针
    void (*free)(void *ptr); // value 释放函数指针
    int (*match)(void *ptr, void *key); // value 比较函数指针
    unsigned long len; //链表长度
} list;

//List 节点数据结构
typedef struct listNode {
    struct listNode *prev; //前一个节点指针
    struct listNode *next; //后一个节点指针
    void *value; //节点保存的值， 类型是 void*  说明可以澳村人以类型的值
} listNode;

//List 的遍历器
typedef struct listIter {
    listNode *next; //下一个节点的指针
    int direction; //方向
} listIter;

```

List 的实现比较简单， 通过这个数据结构就能看出来具体的实现原理， 这里就不探讨实现的细节。

### 我们现在来讨论一下 `BLPOP` 命令的实现， 众所周知 Redis 的命令执行是单线程的， 怎么会 Block 客户端呢？ 阻塞的时候， Redis 是否还相应其他的请求呢?  下面我们就来讨论一下。

## Redis Server运行机制

Redis Server 程序在启动的时候， 会一直执行一个 EventLoop 用于监听 FilEvent（文件事件） 和 TimeEvent（时间事件）， 然后不同的执行事件， 这个后面会讲。 

## 怎么实现 Block （Block Client）

当 `BLPOP` 执行的时候， 流程是这样：

* 首先查看是否存在 key， key 存在并且 key 对应的 List 不为空， 直接返回
* key 不存在或者 List 为空， 执行 blockForKeys 操作 （这里 Client 没有返回， 是被阻塞的）

源码如下：

```c

void blockingPopGenericCommand(client *c, int where) {
    robj *o;
    mstime_t timeout;
    int j;

    if (getTimeoutFromObjectOrReply(c,c->argv[c->argc-1],&timeout,UNIT_SECONDS)
        != C_OK) return;

    for (j = 1; j < c->argc-1; j++) {
        o = lookupKeyWrite(c->db,c->argv[j]);
        if (o != NULL) {
            if (o->type != OBJ_LIST) {
                addReply(c,shared.wrongtypeerr);
                return;
            } else {
                if (listTypeLength(o) != 0) {
                    /* Non empty list, this is like a non normal [LR]POP. */
                    char *event = (where == LIST_HEAD) ? "lpop" : "rpop";
                    robj *value = listTypePop(o,where);
                    serverAssert(value != NULL);

                    addReplyMultiBulkLen(c,2);
                    addReplyBulk(c,c->argv[j]);
                    addReplyBulk(c,value);
                    decrRefCount(value);
                    notifyKeyspaceEvent(NOTIFY_LIST,event,
                                        c->argv[j],c->db->id);

                    //这里能看出来， 当 List 的元素为空的时候， 是会从 db 中删除 List 的
                    //这个是实现 UnBlock 的关键， 这样之后的 PUSH 操作的时候， 都会执行 dbAdd() 来新建 List
                    if (listTypeLength(o) == 0) {
                        dbDelete(c->db,c->argv[j]);
                        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",
                                            c->argv[j],c->db->id);
                    }
                    signalModifiedKey(c->db,c->argv[j]);
                    server.dirty++;

                    /* Replicate it as an [LR]POP instead of B[LR]POP. */
                    rewriteClientCommandVector(c,2,
                        (where == LIST_HEAD) ? shared.lpop : shared.rpop,
                        c->argv[j]);
                    return;
                }
            }
        }
    }

    /* If we are inside a MULTI/EXEC and the list is empty the only thing
     * we can do is treating it as a timeout (even with timeout 0). */
    if (c->flags & CLIENT_MULTI) {
        addReply(c,shared.nullmultibulk);
        return;
    }

    /* If the list is empty or the key does not exists we must block */
    blockForKeys(c, c->argv + 1, c->argc - 2, timeout, NULL);
}

```

接下来我们看看 `blockForKeys` 方法做了什么工作：

```c

void blockForKeys(client *c, robj **keys, int numkeys, mstime_t timeout, robj *target) {
    dictEntry *de;
    list *l;
    int j;

    //设置 Client 的 block 时间
    c->bpop.timeout = timeout;
    c->bpop.target = target;

    if (target != NULL) incrRefCount(target);

    //这里对每个 key， 都在 client->db->blocking_keys 字典中查找 key， 并把 client 加入到 key 对应的 链表中
    //client->db 对应的是执行当前 command 的 client 对应的 server->db
    for (j = 0; j < numkeys; j++) {
        /* If the key already exists in the dict ignore it. */
        if (dictAdd(c->bpop.keys,keys[j],NULL) != DICT_OK) continue;
        incrRefCount(keys[j]);

        /* And in the other "side", to map keys -> clients */
        de = dictFind(c->db->blocking_keys,keys[j]);
        if (de == NULL) {
            int retval;

            /* For every key we take a list of clients blocked for it */
            l = listCreate();
            retval = dictAdd(c->db->blocking_keys,keys[j],l);
            incrRefCount(keys[j]);
            serverAssertWithInfo(c,keys[j],retval == DICT_OK);
        } else {
            l = dictGetVal(de);
        }
        listAddNodeTail(l,c);
    }

    // 这里是真正的开始阻塞当前的 client
    blockClient(c,BLOCKED_LIST);
}

```

然后， 我们看下 `blockClient` 是怎么工作的。

```c

/* Block a client for the specific operation type. Once the CLIENT_BLOCKED
 * flag is set client query buffer is not longer processed, but accumulated,
 * and will be processed when the client is unblocked. */
void blockClient(client *c, int btype) {
    c->flags |= CLIENT_BLOCKED; //client 的标志位
    c->btype = btype; //被阻塞的类型
    server.bpop_blocked_clients++;
}

```

上面的的代码流程就是 Redis 在执行 BLPOP Command 的时候， 阻塞 Client 的流程， 只要是对 Client 不进行响应， 然后将 Client 加入到 server.db->blocking_keys 字典中每个 key 为 BLPOP 指令参数的链表中， 最后在 Client 的标志位中添加 CLIENT_BLOCKED 标志。
接下来我们来看看 Client 是怎么取消阻塞的。

## 怎么实现 UnBlock （Client UnBlock）

上面的介绍可以推断出， Client 的 UnBlock 操作一定会在 List 进行 PUSH 指令的时候触发， 事实上正是这样。

`Client 之所以能够被 Block， 说明当前的数据库中 key 不存在或者 List 的长度为 0， 上面已经能够得知当 List 上都为 0 的时候， Redis
会直接删除当前 List。 所以， 当再次进行 PUSH 操作的时候， Redis 会使用 dbAdd() 在数据库中新建 List， 正是这个操作会将当前的 key
添加进 server.ready_keys 哈希表中。 这些操作完成了 UnBlock 的上半部分。`

Redis 会在每次执行 Command 的操作的时候， 会进行 server.ready_keys 的检查， 不为空的时候就会执行 handleClientsBlockedOnLists() 
方法， 这是最终对 Client 进行 UnBlock 的关键步骤。 UnBlock 的操作分为两个步骤：

* 1) unblockClient() 将 Client 从 db->blocking_keys 中移除
* 2) serveClientBlockedOnList() 响应 Client 被阻塞时候的操作， 并返回数据

### 接下来我们查看源码是怎么完成这一系列操作的。首先我们看下 List PUSH 操作的代码， 这个步骤主要是将 PUSH 操作的 key 添加进 server.ready_keys 哈希表中

```c

void lpushCommand(client *c) {
    pushGenericCommand(c,LIST_HEAD);
}

void pushGenericCommand(client *c, int where) {
    int j, pushed = 0;

    // 从 db 中查找 key
    robj *lobj = lookupKeyWrite(c->db,c->argv[1]);

    // 判断类型， 不是 List 直接返回类型错误
    if (lobj && lobj->type != OBJ_LIST) {
        addReply(c,shared.wrongtypeerr);
        return;
    }

    for (j = 2; j < c->argc; j++) {
        if (!lobj) {
            //这里 List 默认的存储结构是 ziplist （压缩链表）
            lobj = createQuicklistObject();

            //设置一些 ziplist 的选项
            quicklistSetOptions(lobj->ptr, server.list_max_ziplist_size,
                                server.list_compress_depth);
            //这个方法是问题的关键， 接下来我们来详细了解这个函数的执行流程
            dbAdd(c->db,c->argv[1],lobj);
        }
        listTypePush(lobj,c->argv[j],where);
        pushed++;
    }
    addReplyLongLong(c, (lobj ? listTypeLength(lobj) : 0));
    if (pushed) {
        char *event = (where == LIST_HEAD) ? "lpush" : "rpush";

        signalModifiedKey(c->db,c->argv[1]);
        notifyKeyspaceEvent(NOTIFY_LIST,event,c->argv[1],c->db->id);
    }
    server.dirty += pushed;
}

```

我们来看 dbAdd() 方法的执行流程

```c

/* Add the key to the DB. It's up to the caller to increment the reference
 * counter of the value if needed.
 *
 * The program is aborted if the key already exists. */
void dbAdd(redisDb *db, robj *key, robj *val) {
    sds copy = sdsdup(key->ptr);
    int retval = dictAdd(db->dict, copy, val);

    serverAssertWithInfo(NULL,key,retval == DICT_OK);
    //signalListAsReady() 是关键点
    if (val->type == OBJ_LIST) signalListAsReady(db, key);
    if (server.cluster_enabled) slotToKeyAdd(key);
 }

// signalListAsReady 操作源码

/* If the specified key has clients blocked waiting for list pushes, this
 * function will put the key reference into the server.ready_keys list.
 * Note that db->ready_keys is a hash table that allows us to avoid putting
 * the same key again and again in the list in case of multiple pushes
 * made by a script or in the context of MULTI/EXEC.
 *
 * The list will be finally processed by handleClientsBlockedOnLists() */
void signalListAsReady(redisDb *db, robj *key) {
    readyList *rl;

    /* No clients blocking for this key? No need to queue it. */
    // key 没有 block 任何 client, 不需要进行下一步操作
    if (dictFind(db->blocking_keys,key) == NULL) return;

    /* Key was already signaled? No need to queue it again. */
    // key 已经在 server.ready_keys 中， 直接返回
    if (dictFind(db->ready_keys,key) != NULL) return;

    /* Ok, we need to queue this key into server.ready_keys. */
    // key 加入进 server.ready_keys 中
    rl = zmalloc(sizeof(*rl));
    rl->key = key;
    rl->db = db;
    incrRefCount(key);
    listAddNodeTail(server.ready_keys,rl);

    /* We also add the key in the db->ready_keys dictionary in order
     * to avoid adding it multiple times into a list with a simple O(1)
     * check. */
    incrRefCount(key);
    serverAssert(dictAdd(db->ready_keys,key,NULL) == DICT_OK);
}

```
至此， 我们的 Client 仍然在 Block 中， 但是由于 PUSH 操作， 我们已经将 Block Client 的 key 添加进了 server.ready_keys 中， 接下来对 Redis 的指令操作都会进行检查， 对已经 Blocked 的 Clients 进行 UbBlock 和 Response。

### 最终对 Client 的 UnBlock 和 Response 源码分析

每次 Redis Server 响应 Command 的时候， 都会进行 server.ready_keys 的检查操作。我们来看下 processCommand 的具体实现。

`关于 Redis Server 怎么响应 Client Command 的流程， 我们以后在详细讨论。`

```c

// server.c 文件
/* If this function gets called we already read a whole
 * command, arguments are in the client argv/argc fields.
 * processCommand() execute the command or prepare the
 * server for a bulk read from the client.
 *
 * If C_OK is returned the client is still alive and valid and
 * other operations can be performed by the caller. Otherwise
 * if C_ERR is returned the client was destroyed (i.e. after QUIT). */
int processCommand(client *c) {
    /* The QUIT command is handled separately. Normal command procs will
     * go through checking for replication and QUIT will cause trouble
     * when FORCE_REPLICATION is enabled and would be implemented in
     * a regular command proc. */
     //Client 退出指令
    if (!strcasecmp(c->argv[0]->ptr,"quit")) {
        addReply(c,shared.ok);
        c->flags |= CLIENT_CLOSE_AFTER_REPLY;
        return C_ERR;
    }

    /* Now lookup the command and check ASAP about trivial error conditions
     * such as wrong arity, bad command name and so forth. */
     //查找指令执行的方法
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    if (!c->cmd) {
        flagTransaction(c);
        addReplyErrorFormat(c,"unknown command '%s'",
            (char*)c->argv[0]->ptr);
        return C_OK;
    } else if ((c->cmd->arity > 0 && c->cmd->arity != c->argc) ||
               (c->argc < -c->cmd->arity)) {
        flagTransaction(c);
        addReplyErrorFormat(c,"wrong number of arguments for '%s' command",
            c->cmd->name);
        return C_OK;
    }

    /* Check if the user is authenticated */
    //是否授权
    if (server.requirepass && !c->authenticated && c->cmd->proc != authCommand)
    {
        flagTransaction(c);
        addReply(c,shared.noautherr);
        return C_OK;
    }

    /* If cluster is enabled perform the cluster redirection here.
     * However we don't perform the redirection if:
     * 1) The sender of this command is our master.
     * 2) The command has no key arguments. */
    if (server.cluster_enabled &&
        !(c->flags & CLIENT_MASTER) &&
        !(c->flags & CLIENT_LUA &&
          server.lua_caller->flags & CLIENT_MASTER) &&
        !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0 &&
          c->cmd->proc != execCommand))
    {
        int hashslot;
        int error_code;
        clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,
                                        &hashslot,&error_code);
        if (n == NULL || n != server.cluster->myself) {
            if (c->cmd->proc == execCommand) {
                discardTransaction(c);
            } else {
                flagTransaction(c);
            }
            clusterRedirectClient(c,n,hashslot,error_code);
            return C_OK;
        }
    }

    /* Handle the maxmemory directive.
     *
     * First we try to free some memory if possible (if there are volatile
     * keys in the dataset). If there are not the only thing we can do
     * is returning an error. */
    if (server.maxmemory) {
        int retval = freeMemoryIfNeeded();
        /* freeMemoryIfNeeded may flush slave output buffers. This may result
         * into a slave, that may be the active client, to be freed. */
        if (server.current_client == NULL) return C_ERR;

        /* It was impossible to free enough memory, and the command the client
         * is trying to execute is denied during OOM conditions? Error. */
        if ((c->cmd->flags & CMD_DENYOOM) && retval == C_ERR) {
            flagTransaction(c);
            addReply(c, shared.oomerr);
            return C_OK;
        }
    }

    /* Don't accept write commands if there are problems persisting on disk
     * and if this is a master instance. */
    if (((server.stop_writes_on_bgsave_err &&
          server.saveparamslen > 0 &&
          server.lastbgsave_status == C_ERR) ||
          server.aof_last_write_status == C_ERR) &&
        server.masterhost == NULL &&
        (c->cmd->flags & CMD_WRITE ||
         c->cmd->proc == pingCommand))
    {
        flagTransaction(c);
        if (server.aof_last_write_status == C_OK)
            addReply(c, shared.bgsaveerr);
        else
            addReplySds(c,
                sdscatprintf(sdsempty(),
                "-MISCONF Errors writing to the AOF file: %s\r\n",
                strerror(server.aof_last_write_errno)));
        return C_OK;
    }

    /* Don't accept write commands if there are not enough good slaves and
     * user configured the min-slaves-to-write option. */
    if (server.masterhost == NULL &&
        server.repl_min_slaves_to_write &&
        server.repl_min_slaves_max_lag &&
        c->cmd->flags & CMD_WRITE &&
        server.repl_good_slaves_count < server.repl_min_slaves_to_write)
    {
        flagTransaction(c);
        addReply(c, shared.noreplicaserr);
        return C_OK;
    }

    /* Don't accept write commands if this is a read only slave. But
     * accept write commands if this is our master. */
    if (server.masterhost && server.repl_slave_ro &&
        !(c->flags & CLIENT_MASTER) &&
        c->cmd->flags & CMD_WRITE)
    {
        addReply(c, shared.roslaveerr);
        return C_OK;
    }

    /* Only allow SUBSCRIBE and UNSUBSCRIBE in the context of Pub/Sub */
    if (c->flags & CLIENT_PUBSUB &&
        c->cmd->proc != pingCommand &&
        c->cmd->proc != subscribeCommand &&
        c->cmd->proc != unsubscribeCommand &&
        c->cmd->proc != psubscribeCommand &&
        c->cmd->proc != punsubscribeCommand) {
        addReplyError(c,"only (P)SUBSCRIBE / (P)UNSUBSCRIBE / PING / QUIT allowed in this context");
        return C_OK;
    }

    /* Only allow INFO and SLAVEOF when slave-serve-stale-data is no and
     * we are a slave with a broken link with master. */
    if (server.masterhost && server.repl_state != REPL_STATE_CONNECTED &&
        server.repl_serve_stale_data == 0 &&
        !(c->cmd->flags & CMD_STALE))
    {
        flagTransaction(c);
        addReply(c, shared.masterdownerr);
        return C_OK;
    }

    /* Loading DB? Return an error if the command has not the
     * CMD_LOADING flag. */
    if (server.loading && !(c->cmd->flags & CMD_LOADING)) {
        addReply(c, shared.loadingerr);
        return C_OK;
    }

    /* Lua script too slow? Only allow a limited number of commands. */
    if (server.lua_timedout &&
          c->cmd->proc != authCommand &&
          c->cmd->proc != replconfCommand &&
        !(c->cmd->proc == shutdownCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'n') &&
        !(c->cmd->proc == scriptCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'k'))
    {
        flagTransaction(c);
        addReply(c, shared.slowscripterr);
        return C_OK;
    }

    /* Exec the command */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        // 这里是 MULTI 指令中， 直接将指令放进队列， 知道遇见上面的指令才会一次性执行
        // 有关 MULTI （事务） 后面会在介绍
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;

        //直接看这里， 关键部分来了！
        if (listLength(server.ready_keys))
            handleClientsBlockedOnLists();
    }
    return C_OK;
}


// t_list.c 文件
/* This function should be called by Redis every time a single command,
 * a MULTI/EXEC block, or a Lua script, terminated its execution after
 * being called by a client.
 *
 * All the keys with at least one client blocked that received at least
 * one new element via some PUSH operation are accumulated into
 * the server.ready_keys list. This function will run the list and will
 * serve clients accordingly. Note that the function will iterate again and
 * again as a result of serving BRPOPLPUSH we can have new blocking clients
 * to serve because of the PUSH side of BRPOPLPUSH. */
void handleClientsBlockedOnLists(void) {
    while(listLength(server.ready_keys) != 0) {
        list *l;

        /* Point server.ready_keys to a fresh list and save the current one
         * locally. This way as we run the old list we are free to call
         * signalListAsReady() that may push new elements in server.ready_keys
         * when handling clients blocked into BRPOPLPUSH. */
        l = server.ready_keys;
        server.ready_keys = listCreate();

        //遍历 server.ready_keys 哈希表
        while(listLength(l) != 0) {
            listNode *ln = listFirst(l);
            readyList *rl = ln->value;

            /* First of all remove this key from db->ready_keys so that
             * we can safely call signalListAsReady() against this key. */
            dictDelete(rl->db->ready_keys,rl->key);

            /* If the key exists and it's a list, serve blocked clients
             * with data. */
            robj *o = lookupKeyWrite(rl->db,rl->key);
            if (o != NULL && o->type == OBJ_LIST) {
                dictEntry *de;

                /* We serve clients in the same order they blocked for
                 * this key, from the first blocked to the last. */
                 // 这里是在 db->blocking_keys 中查找 server.ready_keys 中的每一个 key
                 // 查找的结果是一个 List， 每一个节点都是一个被 Blocked 的 Client
                de = dictFind(rl->db->blocking_keys,rl->key);
                if (de) {
                    list *clients = dictGetVal(de);
                    int numclients = listLength(clients);

                    // 遍历当前 key 所 Blocked 的 Clients 
                    while(numclients--) {
                        listNode *clientnode = listFirst(clients);
                        client *receiver = clientnode->value; // 获取的 Client 信息
                        robj *dstkey = receiver->bpop.target; // BRPOPLPUSH 命令中会有一个最终 PUSH 的地址
                        // Client Blocked 之前的最后一个 Command 
                        // 我们在 Client UnBlocked 之后， 会接着响应这个命令
                        int where = (receiver->lastcmd &&
                                     receiver->lastcmd->proc == blpopCommand) ?
                                    LIST_HEAD : LIST_TAIL;
                        robj *value = listTypePop(o,where);

                        if (value) {
                            /* Protect receiver->bpop.target, that will be
                             * freed by the next unblockClient()
                             * call. */
                            if (dstkey) incrRefCount(dstkey);
                            // 第一步， UnBlock Client
                            unblockClient(receiver);

                            // 执行最后一个 Command
                            if (serveClientBlockedOnList(receiver,
                                rl->key,dstkey,rl->db,value,
                                where) == C_ERR)
                            {
                                /* If we failed serving the client we need
                                 * to also undo the POP operation. */
                                    listTypePush(o,value,where);
                            }

                            if (dstkey) decrRefCount(dstkey);
                            decrRefCount(value);
                        } else {
                            break;
                        }
                    }
                }

                if (listTypeLength(o) == 0) {
                    dbDelete(rl->db,rl->key);
                }
                /* We don't call signalModifiedKey() as it was already called
                 * when an element was pushed on the list. */
            }

            /* Free this item. */
            decrRefCount(rl->key);
            zfree(rl);
            listDelNode(l,ln);
        }
        listRelease(l); /* We have the new list on place at this point. */
    }
}
```

接下来看下 `unblockClient()` 的具体源码

```c

// blocked.c 文件
/* Unblock a client calling the right function depending on the kind
 * of operation the client is blocking for. */
void unblockClient(client *c) {
    if (c->btype == BLOCKED_LIST) {
        unblockClientWaitingData(c); // 我们是分析 BLPOP ， 所以直接看这个函数
    } else if (c->btype == BLOCKED_WAIT) {
        unblockClientWaitingReplicas(c);
    } else if (c->btype == BLOCKED_MODULE) {
        unblockClientFromModule(c);
    } else {
        serverPanic("Unknown btype in unblockClient().");
    }

    /* Clear the flags, and put the client in the unblocked list so that
     * we'll process new commands in its query buffer ASAP. */
     //在 Client 的标志位中清除 CLIENT_BLOCKED
    c->flags &= ~CLIENT_BLOCKED;
    c->btype = BLOCKED_NONE;
    server.bpop_blocked_clients--;
    /* The client may already be into the unblocked list because of a previous
     * blocking operation, don't add back it into the list multiple times. */
    if (!(c->flags & CLIENT_UNBLOCKED)) {
        c->flags |= CLIENT_UNBLOCKED;
        listAddNodeTail(server.unblocked_clients,c);
    }
}


// t_list.c 文件
/* Unblock a client that's waiting in a blocking operation such as BLPOP.
 * You should never call this function directly, but unblockClient() instead. */
void unblockClientWaitingData(client *c) {
    dictEntry *de;
    dictIterator *di;
    list *l;

    serverAssertWithInfo(c,NULL,dictSize(c->bpop.keys) != 0);
    di = dictGetIterator(c->bpop.keys);
    /* The client may wait for multiple keys, so unblock it for every key. */
    while((de = dictNext(di)) != NULL) {
        robj *key = dictGetKey(de);

        /* Remove this client from the list of clients waiting for this key. */
        l = dictFetchValue(c->db->blocking_keys,key);
        serverAssertWithInfo(c,key,l != NULL);
        listDelNode(l,listSearchKey(l,c));
        /* If the list is empty we need to remove it to avoid wasting memory */
        if (listLength(l) == 0)
            dictDelete(c->db->blocking_keys,key);
    }
    dictReleaseIterator(di);

    /* Cleanup the client structure */
    dictEmpty(c->bpop.keys,NULL);
    if (c->bpop.target) {
        decrRefCount(c->bpop.target);
        c->bpop.target = NULL;
    }
}

```

最后是 `serveClientBlockedOnList()` 的源码分析

```c

// t_list.c 文件
/* This is a helper function for handleClientsBlockedOnLists(). It's work
 * is to serve a specific client (receiver) that is blocked on 'key'
 * in the context of the specified 'db', doing the following:
 *
 * 1) Provide the client with the 'value' element.
 * 2) If the dstkey is not NULL (we are serving a BRPOPLPUSH) also push the
 *    'value' element on the destination list (the LPUSH side of the command).
 * 3) Propagate the resulting BRPOP, BLPOP and additional LPUSH if any into
 *    the AOF and replication channel.
 *
 * The argument 'where' is LIST_TAIL or LIST_HEAD, and indicates if the
 * 'value' element was popped fron the head (BLPOP) or tail (BRPOP) so that
 * we can propagate the command properly.
 *
 * The function returns C_OK if we are able to serve the client, otherwise
 * C_ERR is returned to signal the caller that the list POP operation
 * should be undone as the client was not served: This only happens for
 * BRPOPLPUSH that fails to push the value to the destination key as it is
 * of the wrong type. */
int serveClientBlockedOnList(client *receiver, robj *key, robj *dstkey, redisDb *db, robj *value, int where)
{
    robj *argv[3];

    if (dstkey == NULL) {
        /* Propagate the [LR]POP operation. */
        argv[0] = (where == LIST_HEAD) ? shared.lpop :
                                          shared.rpop;
        argv[1] = key;
        propagate((where == LIST_HEAD) ?
            server.lpopCommand : server.rpopCommand,
            db->id,argv,2,PROPAGATE_AOF|PROPAGATE_REPL);

        /* BRPOP/BLPOP */
        addReplyMultiBulkLen(receiver,2);
        addReplyBulk(receiver,key);
        addReplyBulk(receiver,value);
    } else {
        /* BRPOPLPUSH */
        robj *dstobj =
            lookupKeyWrite(receiver->db,dstkey);
        if (!(dstobj &&
             checkType(receiver,dstobj,OBJ_LIST)))
        {
            /* Propagate the RPOP operation. */
            argv[0] = shared.rpop;
            argv[1] = key;
            propagate(server.rpopCommand,
                db->id,argv,2,
                PROPAGATE_AOF|
                PROPAGATE_REPL);
            rpoplpushHandlePush(receiver,dstkey,dstobj,
                value);
            /* Propagate the LPUSH operation. */
            argv[0] = shared.lpush;
            argv[1] = dstkey;
            argv[2] = value;
            propagate(server.lpushCommand,
                db->id,argv,3,
                PROPAGATE_AOF|
                PROPAGATE_REPL);
        } else {
            /* BRPOPLPUSH failed because of wrong
             * destination type. */
            return C_ERR;
        }
    }
    return C_OK;
}

```

### BLPOP 命令的整体执行流程就是这样。从上面的分析中我们可以看出List是按照“先阻塞先服务”的策略来处理阻塞解除的。

## 总结

这篇文章主要分析了 Redi 中 List 的基本实现和 BLPOP 命令执行对于 Client 的 Block 和 UnBlock 的详细流程。

当然， List 的内容肯定不止这些， 比如 List 会使用到 zipist 的数据结构来减小内存的使用， Client 的 UnBlock 还可能是由阻塞超时引起的等等。 我们在其他的文章中会在进行详细的解答。