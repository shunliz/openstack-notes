## OVSDB 概览

ovsdb-server提供RPC接口给一个或者多个Open vSwitch database, 它支持JSON-RPC客户端通过主动或者被动TPC/IP或者unix domain socket连接。 每一个ovsdb文件通过命令航传递，如果没有传，默认是/etc/openvswitch/conf.db

ovsdbd保存交换机级别的配置：

* 交换机，接口，和隧道信息
* ovsdb和openflow控制器地址

配置保存在磁盘，重启不会丢失

通过ovsdb协议和控制和ovs-vswitchd通信，命令行工具：

ovs-vsctl:通过配置ovs-vswitchd修改ovsdb

ovsdb-tool： ovsdb管理，创建，整理和转化ovsdb，显示ovsdb信息



### 关键数据结构

* `ovsdb_schema`

* `ovsdb`
* `ovsdb_server`
* `ovsdb_table_schema`
* `ovsdb_table`

### **ovsdb**

```
/* Database schema. */
struct ovsdb_schema {
    char *name;
    char *version;
    char *cksum;
    struct shash tables;        /* Contains "struct ovsdb_table_schema *"s. */
};

/* Database. */
struct ovsdb {
    struct ovsdb_schema *schema;
    struct ovs_list replicas;   /* Contains "struct ovsdb_replica"s. */
    struct shash tables;        /* Contains "struct ovsdb_table *"s. */

    /* Triggers. */
    struct ovs_list triggers;   /* Contains "struct ovsdb_trigger"s. */
    bool run_triggers;
};
```

### OVSDB Table {#22-ovsdb-table}

```
/* Schema for a database table. */
struct ovsdb_table_schema {
    char *name;
    bool mutable;
    bool is_root;               /* Part of garbage collection root set? */
    unsigned int max_rows;      /* Maximum number of rows. */
    struct shash columns;       /* Contains "struct ovsdb_column *"s. */
    struct ovsdb_column_set *indexes;
    size_t n_indexes;
};

/* Database table. */
struct ovsdb_table {
    struct ovsdb_table_schema *schema;
    struct ovsdb_txn_table *txn_table; /* Only if table is in a transaction. */
    struct hmap rows;           /* Contains "struct ovsdb_row"s. */

    /* An array of schema->n_indexes hmaps, each of which contains "struct
     * ovsdb_row"s.  Each of the hmap_nodes in indexes[i] are at index 'i' at
     * the end of struct ovsdb_row, following the 'fields' member. */
    struct hmap *indexes;
};
```

Open\_vswitch是根表，只有一条记录。完整的实体关系图在ovs-vswitchd.conf.db man page.

![](/assets/network-virtualnet-ovs-ovsdbtb1.png)

### ovsdb-server流程图

![](/assets/network-virtualnet-ovs-ovsdb2.png)

ovsdb/ovsdb-server.c

```
int
main(int argc, char *argv[])
{
    /* step.1. handle configs, open db */
    SSET_FOR_EACH (db_filename, &db_filenames)
        open_db(&server_config, db_filename);

    /* step.2. create unixctl server and register commands */
    unixctl_server_create(unixctl_path, &unixctl);
    unixctl_command_register("exit", "", 0, 0, ovsdb_server_exit, &exiting);
    ...

    /* step.3. enter main loop */
    main_loop();
      |--while (!*exiting) {
            /* step.3.1 handle control messages from CLI and RPC */
            unixctl_server_run(unixctl);       // handle CLI commands (turn into RPC requests)
            ovsdb_jsonrpc_server_run(jsonrpc); // handle RPC requests

            /* step.3.2 ovsdb session execute */
            SHASH_FOR_EACH(node, all_dbs) {
                ovsdb_trigger_run(db->db, time_msec()); // execute session, commit changes
            }

            /* step.3.3 update Manager status(es) every 2.5 seconds */
            if (time_msec() >= status_timer)
                update_remote_status(jsonrpc, remotes, all_dbs);

            /* step.3.4 wait events arriving */

            /* step.3.5 block until events arrive */
            poll_block();
        }

    /* step.4. clean and exit */
}
```

### 过程和模块

**创建ovsdb**

每一个配置的ovsdb文件，ovsdb-server都会创建一个struct ovsdb实例， 通过open\_db\(\)函数

然后通过调用reconfigure\_remotes\(\)配置ovsdb-server的远端.一个远端是一个主动或者被动的流式连接，比如 pssl:或者tpc:1.2.3.4

最后调用：

```
static struct ovsdb_jsonrpc_remote *
ovsdb_jsonrpc_server_add_remote(struct ovsdb_jsonrpc_server *svr,
                                const char *name,
                                const struct ovsdb_jsonrpc_options *options)
{
    jsonrpc_pstream_open(name, &listener, options->dscp);

    remote = xmalloc(sizeof *remote);
    remote->server = svr;
    remote->listener = listener;
    ovs_list_init(&remote->sessions);

    ovsdb_jsonrpc_session_create(remote, jsonrpc_session_open(name, true),
                                      svr->read_only || remote->read_only);
    return remote;
}
```

然后注册IPC方法，这样就可以通过命令行执行ovsdb-server命令

### 处理IPC消息

在ovs-vswitchd的主循环中，`unixctl_server_run()`和`ovsdb_jsonrpc_server_run()被调用。`

unixctl server从unix IPC socket获取消息，socket默认在/var/run/openvswitch/ovsdb-server.xxx.ctl. 然后转化消息称一个RPC调用，最后被jsonrpc server处理。

```
void
unixctl_server_run(struct unixctl_server *server)
{
    pstream_accept(server->listener, &stream);
    conn->rpc = jsonrpc_open(stream);

    LIST_FOR_EACH_SAFE (conn, next, node, &server->conns) {
        run_connection(conn);
          |
          |--jsonrpc_run()
          |    |--jsonrpc_send()
          |--jsonrpc_recv()
          |--process_command(conn, msg) // format output text
    }
}
```

```
void
ovsdb_jsonrpc_server_run(struct ovsdb_jsonrpc_server *svr)
{
    SHASH_FOR_EACH (node, &svr->remotes) {
        struct ovsdb_jsonrpc_remote *remote = node->data;

        pstream_accept(remote->listener, &stream);
        jsonrpc_session_open_unreliably(jsonrpc_open(stream), remote->dscp);
        ovsdb_jsonrpc_session_create(remote, js, svr->read_only || remote->read_only);

        ovsdb_jsonrpc_session_run_all(remote);
          |
          |--LIST_FOR_EACH_SAFE (s, next, node, &remote->sessions)
                ovsdb_jsonrpc_session_run(s);
                  |--jsonrpc_sesion_recv()
                     msg->type:
                       case: ovsdb_jsonrpc_session_got_request(s, msg);
                       case: ovsdb_jsonrpc_session_got_notify(s, msg);
                       default: got unexpected msg();
    }
}
```

jsonrpc处理

```
static void
ovsdb_jsonrpc_session_got_request(struct ovsdb_jsonrpc_session *s,
                                  struct jsonrpc_msg *request)
{
    switch(request->method) {
    case "transact"            if (!reply) reply = execute_transaction(s, db, request);
    case "monitor"             if (!reply) reply = ovsdb_jsonrpc_monitor_create();
    case "monitor_cond_change" reply = ovsdb_jsonrpc_monitor_cond_change();
    case "monitor_cancel"      reply = ovsdb_jsonrpc_monitor_cancel();
    case "get_schema"          if (!reply) reply = jsonrpc_create_reply();
    case "list_dbs"            reply = jsonrpc_create_reply();
    case "lock"                reply = ovsdb_jsonrpc_session_lock();
    case "steal"               reply = ovsdb_jsonrpc_session_lock();
    case "unlock"              reply = ovsdb_jsonrpc_session_unlock(s, request);
    case "echo"                reply = jsonrpc_create_reply();
    }

    if (reply) {
        jsonrpc_msg_destroy(request);
        ovsdb_jsonrpc_session_send(s, reply);
    }
}
```

### 执行 OVSDB 会话 {#43-execute-ovsdb-session}

```
void
ovsdb_trigger_run(struct ovsdb *db, long long int now)
{
    LIST_FOR_EACH_SAFE (t, next, node, &db->triggers) {
        if (run_triggers || now - t->created >= t->timeout_msec) {
            ovsdb_trigger_try(t, now);
              |--ovsdb_execute(t->db, t->session, t->request, ...)
                  |--ovsdb_txn_create()
                  |--parse and execute operations
                  |--ovsdb_txn_commit()
        }
    }
}
```



