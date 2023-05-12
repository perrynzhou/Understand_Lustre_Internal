# ost内核相关写日志分析

## 1. 测试环境

lustre版本：2.15.0

## 2. 操作步骤

1. 在ost服务器上执行`lctl debug_kernel /tmp/log`清空日志
2. 在客户端中使用`echo 1 > /mnt/client/test`命令往lustre文件系统内写数据
3. 在ost服务器上执行`lctl debug_kernel /tmp/log`获取ost写日志

## 3. ost写日志内容

```bash
00000100:00000040:0.0F:1683537053.478728:0:3924:0:(events.c:358:request_in_callback()) incoming req@00000000ff55337f x1765282925303744 msgsize 440
00000020:00000040:0.0:1683537053.478743:0:4098:0:(genops.c:911:class_conn2export()) looking for export cookie 0x2250937c6e18e05c
00000020:00000040:0.0:1683537053.478745:0:4098:0:(lustre_handles.c:151:class_handle2object()) GET export 00000000a6c95d59 refcount=6
00000100:00000040:0.0:1683537053.478752:0:4098:0:(service.c:1263:ptlrpc_at_set_timer()) armed ost_io at +1s
00010000:00000040:0.0:1683537053.478760:0:4098:0:(ldlm_resource.c:1567:ldlm_resource_getref()) getref res: 00000000d17bf069 count: 2
00010000:00000040:0.0:1683537053.478762:0:4098:0:(ldlm_resource.c:1601:ldlm_resource_putref()) putref res: 00000000d17bf069 count: 1
00000100:00000040:0.0:1683537053.478768:0:4098:0:(service.c:2031:ptlrpc_server_request_get()) RPC GETting export 00000000a6c95d59 : new rpc_count 1
00000100:00000040:0.0:1683537053.478771:0:4098:0:(lustre_net.h:2402:ptlrpc_rqphase_move()) @@@ move request phase from New to Interpret  req@00000000ff55337f x1765282925303744/t0(0) o10->1aa3d865-a097-4b6b-9bdc-e58c401e8889@192.168.242.10@tcp:309/0 lens 440/0 e 0 to 0 dl 1683537059 ref 1 fl New:/0/ffffffff rc 0/-1 job:''
00000001:00000040:0.0:1683537053.478850:0:4098:0:(tgt_lastrcvd.c:947:tgt_last_commit_cb_add()) callback GETting export 00000000a6c95d59 : new cb_count 1
00000020:00000040:0.0:1683537053.478851:0:4098:0:(genops.c:968:class_export_get()) GET export 00000000a6c95d59 refcount=7
00010000:00000040:0.0:1683537053.478860:0:4098:0:(ldlm_resource.c:1567:ldlm_resource_getref()) getref res: 00000000d17bf069 count: 2
00010000:00000040:0.0:1683537053.478862:0:4098:0:(ldlm_resource.c:1601:ldlm_resource_putref()) putref res: 00000000d17bf069 count: 1
00010000:00000040:0.0:1683537053.478863:0:4098:0:(ldlm_lib.c:3178:target_committed_to_req()) last_committed 4294967300, transno 4294967301, xid 1765282925303744
00000100:00000040:0.0:1683537053.478866:0:4098:0:(connection.c:145:ptlrpc_connection_addref()) conn=00000000f30635f5 refcount 3 to 192.168.242.10@tcp
00000100:00000040:0.0:1683537053.478867:0:4098:0:(niobuf.c:58:ptl_send_buf()) peer_id 12345-192.168.242.10@tcp
00000100:00000040:0.0:1683537053.478875:0:4098:0:(lustre_net.h:1943:ptlrpc_connection_put()) PUT conn=00000000f30635f5 refcount 2 to 192.168.242.10@tcp
00000100:00000040:0.0:1683537053.478876:0:4098:0:(lustre_net.h:2402:ptlrpc_rqphase_move()) @@@ move request phase from Interpret to Complete  req@00000000ff55337f x1765282925303744/t4294967301(0) o10->1aa3d865-a097-4b6b-9bdc-e58c401e8889@192.168.242.10@tcp:309/0 lens 440/432 e 0 to 0 dl 1683537059 ref 1 fl Interpret:/0/0 rc 0/0 job:''
00000100:00000040:0.0:1683537053.478878:0:4098:0:(service.c:1102:ptlrpc_server_finish_active_request()) RPC PUTting export 00000000a6c95d59 : new rpc_count 0
00010000:00000040:0.0:1683537053.478879:0:4098:0:(ldlm_resource.c:1567:ldlm_resource_getref()) getref res: 00000000d17bf069 count: 2
00010000:00000040:0.0:1683537053.478879:0:4098:0:(ldlm_resource.c:1601:ldlm_resource_putref()) putref res: 00000000d17bf069 count: 1
00000020:00000040:0.0:1683537053.478881:0:4098:0:(genops.c:979:class_export_put()) PUTting export 00000000a6c95d59 : new refcount 6
```

## 4. 日志分析

### 4.1. 代码中所需结构体及其作用

将4.2中所使用到的lustre结构体在此展示

### 4.2. ost写操作流程代码分析

按照章节3中流程列出代码位置与函数注释

./lustre/ptlrpc/events.c

```c
// 服务端收到请求
/*
 * Server's incoming request callback
 */
void request_in_callback(struct lnet_event *ev)
```

./lustre/obdclass/genops.c

```c
// 映射连接到客户端
/* map connection to client */
struct obd_export *class_conn2export(struct lustre_handle *conn)
```

./lustre/obdclass/lustre_handles.c

```c
// 句柄到目标
void *class_handle2object(u64 cookie, const char *owner)
```

./lustre/ptlrpc/service.c

```c
// 设置RPC的截止时间？
static void ptlrpc_at_set_timer(struct ptlrpc_service_part *svcpt)
```

./lustre/ldlm/ldlm_resource.c

```c
// 获取ldlm资源锁？
struct ldlm_resource *ldlm_resource_getref(struct ldlm_resource *res)
/* Returns 1 if the resource was freed, 0 if it remains. */
int ldlm_resource_putref(struct ldlm_resource *res)
```

./lustre/ptlrpc/service.c

```c
// 处理ptlrpc队列中的请求
/**
 * Fetch a request for processing from queue of unprocessed requests.
 * Favors high-priority requests.
 * Returns a pointer to fetched request.
 */
static struct ptlrpc_request *
ptlrpc_server_request_get(struct ptlrpc_service_part *svcpt, bool force)
```

./lustre/include/lustre_net.h

```c
// 改变已处理请求?
/** Change request phase of \\a req to \\a new_phase */
static inline void
ptlrpc_rqphase_move(struct ptlrpc_request *req, enum rq_phase new_phase)
```

./lustre/target/tgt_lastrcvd.c

```c
// 同步事务函数？
/**
 * Add commit callback function, it returns a non-zero value to inform
 * caller to use sync transaction if necessary.
 */
static int tgt_last_commit_cb_add(struct thandle *th, struct lu_target *tgt,
				  struct obd_export *exp, __u64 transno)
```

./lustre/obdclass/genops.c

```c
// obd_export结构体服务于客户端连接服务端时的obd设备
struct obd_export *class_export_get(struct obd_export *exp)
```

./lustre/ldlm/ldlm_resource.c

```csharp
// 获取ldlm资源锁？
struct ldlm_resource *ldlm_resource_getref(struct ldlm_resource *res)
/* Returns 1 if the resource was freed, 0 if it remains. */
int ldlm_resource_putref(struct ldlm_resource *res)
```

./lustre/ldlm/ldlm_lib.c

```c
// ptlrpc请求
void target_committed_to_req(struct ptlrpc_request *req)
```

./lustre/ptlrpc/connection.c

```c
// ptlrpc_connection为单一正面连接？
struct ptlrpc_connection *
ptlrpc_connection_addref(struct ptlrpc_connection *conn)
```

./lustre/ptlrpc/niobuf.c

```c
// 辅助函数？发送偏移字节连接到正面连接中
/**
 * Helper function. Sends \\a len bytes from \\a base at offset \\a offset
 * over \\a conn connection to portal \\a portal.
 * Returns 0 on success or error code.
 */
static int ptl_send_buf(struct lnet_handle_md *mdh, void *base, int len,
			enum lnet_ack_req ack, struct ptlrpc_cb_id *cbid,
			lnet_nid_t self, struct lnet_process_id peer_id,
			int portal, __u64 xid, unsigned int offset,
			struct lnet_handle_md *bulk_cookie)
```

./lustre/include/lustre_net.h

```c
// Lustre不会释放连接而是将连接放在缓存中以便需要时重新调用
static inline void  ptlrpc_connection_put(struct ptlrpc_connection *conn)
	/*
	 * We do not remove connection from hashtable and
	 * do not free it even if last caller released ref,
	 * as we want to have it cached for the case it is
	 * needed again.
	 *
	 * Deallocating it and later creating new connection
	 * again would be wastful. This way we also avoid
	 * expensive locking to protect things from get/put
	 * race when found cached connection is freed by
	 * ptlrpc_connection_put().
	 *
	 * It will be freed later in module unload time,
	 * when ptlrpc_connection_fini()->lh_exit->conn_exit()
	 * path is called.
	 */

/** Change request phase of \\a req to \\a new_phase */
static inline void
ptlrpc_rqphase_move(struct ptlrpc_request *req, enum rq_phase new_phase)
```

./lustre/ptlrpc/service.c

```c
// 删除引用计数
/**
 * drop a reference count of the request. if it reaches 0, we either
 * put it into history list, or free it immediately.
 */
void ptlrpc_server_drop_request(struct ptlrpc_request *req)
```

./lustre/ldlm/ldlm_resource.c

```c
// 锁操作
struct ldlm_resource *ldlm_resource_getref(struct ldlm_resource *res)
/* Returns 1 if the resource was freed, 0 if it remains. */
int ldlm_resource_putref(struct ldlm_resource *res)
```

./lustre/obdclass/genops.c

```c
void class_export_put(struct obd_export *exp)
```

## 4.3. 日志内容分析

服务端接收到请求后寻找cookie 0x2250937c6e18e05c

服务端生成句柄到目标为**00000000a6c95d59**，引用计数为6次

服务端设置ost io请求时间为1s

ldlm get生成写锁00000000d17bf069，确认锁处于空闲状态，并put出去

ptlrpc从未执行队列中获取**00000000a6c95d59**

obdclass获取**00000000a6c95d59**，引用计数为7次

ldlm 再次获得锁00000000d17bf069，确认锁处于空闲状态，并put出去

ldlm 服务端最后一次提交转换为4294967300，Transaction number事务号码为4294967301（两个号码相邻），xid事务标识符1765282925303744

ptlrpc报告连接为00000000f30635f5，引用计数为3，远端（即客户端）nid为192.168.242.10@tcp

辅助函数测试远端nid是否能ping通？

ptlrpc报告连接为00000000f30635f5，引用计数为2，远端（即客户端）nid为192.168.242.10@tcp（引用计数是不断往下减少的？如果到0则进入历史队列或直接释放？）

将请求事务改成完成状态

服务端ptlrpc结束活跃的请求**00000000a6c95d59**

ldlm再次获得锁00000000d17bf069，确认锁处于空闲状态，并put出去

obdclass获取**00000000a6c95d59**，引用计数为6次

## 5. ost写流程汇总

1. ptlrpc：服务端收到请求
2. obd_export：服务端建立连接并映射到客户端
3. 句柄到目标
4. ptlrpc：设置RPC的截止时间
5. 锁操作（服务于写？）并进行判断资源是否空闲
6. 处理ptlrpc队列中的请求
7. 改变已处理请求状态
8. 中间桥梁：同步事务函数
9. obd_export：
10. 锁操作（服务于写？）并进行判断资源是否空闲
11. 服务端回复ptlrpc请求
12. 返回ptlrpc_connection ，ptlrpc建立正面连接
13. ptlrpc辅助函数，发送偏移后的字节到正面连接
14. 连接发出，Lustre不会释放连接而是将连接放在缓存中以便需要时重新调用
15. 改变已处理请求状态
16. ptlrpc：服务端删除引用计数，计数到0后将会将该引用推进历史列表中，或者立即释放
17. 锁操作（服务于写？）并进行判断资源是否空闲
18. obd_export：最后obdclass处理export put

## 6. ost写设计的内核模块

ptlrpc：是LNet上的一个RPC协议，它处理状态性服务器，并且具有语义和内置的恢复支持

obdclass：负责数据的输出？

ldlm：负责锁操作