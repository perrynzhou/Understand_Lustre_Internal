# ost内核相关日志分析

## 1. 测试环境

| key | value | 补充说明 |
| --- | --- | --- |
| lustre version | 2.15.0 |  |
| fsname | debugfs |  |
| mds ip | 192.168.242.21 | mds充当mgs且仅有一个mdt |
| oss ip | 192.168.242.22 | oss仅有一个ost |
| client ip | 192.168.242.10 |  |
| client ip | 192.168.242.12 |  |

## 2. 写日志分析

### 2.1. 操作步骤

1. 在oss上执行 `lctl set_param debug=+info`使用info级别日志
2. 在oss上执行`lctl debug_kernel /tmp/log`清空日志
3. 在客户端中使用`echo 123 > /mnt/client/test`命令往lustre文件系统内写数据
4. 在oss上执行`lctl debug_kernel /tmp/log`获取ost写日志

### 2.2. ost写日志及分析

```bash
00000100:00000040:0.0F:1684440225.185283:0:3551:0:(events.c:358:request_in_callback()) incoming req@0000000000b958be x1766202290688064 msgsize 520
# （ptlrpc模块）oss服务器ptlrpc模块接收到请求，事务xid为1766202290688064
00000020:00000040:0.0:1684440225.185299:0:3613:0:(genops.c:911:class_conn2export()) looking for export cookie 0x0
00000100:00000040:0.0:1684440225.185310:0:3613:0:(lustre_net.h:2402:ptlrpc_rqphase_move()) @@@ move request phase from New to Interpret  req@0000000000b958be x1766202290688064/t0(0) o8-><?>@<unknown>:0/0 lens 520/0 e 0 to 0 dl 1684440325 ref 1 fl New:/0/ffffffff rc 0/-1 job:''
# 可能从service.c:2224:ptlrpc_server_handle_request函数调用
# 处理请求00000000bbe1354c，并改变xid 1766202290688064事务状态为New
00000020:00000040:0.0:1684440225.185318:0:3613:0:(obd_config.c:919:class_incref()) incref debugfs-OST0000 (000000005e629f65) now 5 - find
00010000:00080000:0.0:1684440225.185323:0:3613:0:(ldlm_lib.c:1365:target_handle_connect()) debugfs-OST0000: connection from c533e830-451b-4996-a72c-008e7736bf8b@192.168.242.10@tcp t0 exp 0000000028b3ebb2 cur 29963 last 0
# 当超过一定时间未连接后，oss会断开与client的连接，此时在client上执行写入或读取操作都会重新建立连接
00000020:00000040:0.0:1684440225.185326:0:3613:0:(lustre_handles.c:97:class_handle_hash()) added object 0000000061b8a918 with handle 0xaefde85a03e1190b to hash
# 由genops.c:1020:__class_new_export()调用，创建export，并添加其到hash表，然后返回得到一个到export的指针
# object数据结构为portals_handle，用作通过网络引用的对象的泛型类型，例如文件句柄或导出对象
# oss上与该客户端连接对应的export（object）为 0000000061b8a918，client连接cookie缓存为 0xaefde85a03e1190b（作为hash值使用），用于在服务端验证和定位object目标
00000020:00000040:0.0:1684440225.185337:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000061b8a918 refcount=3
00000020:00000040:0.0:1684440225.185338:0:3613:0:(obd_config.c:919:class_incref()) incref debugfs-OST0000 (000000005e629f65) now 6 - export
00000020:00000040:0.0:1684440225.185339:0:3613:0:(genops.c:979:class_export_put()) PUTting export 0000000061b8a918 : new refcount 2
00000020:00000080:0.0:1684440225.185339:0:3613:0:(genops.c:1359:class_connect()) connect: client c533e830-451b-4996-a72c-008e7736bf8b, cookie 0xaefde85a03e1190b
# 上述class_handle_hash()函数到此处都是class_connect()的调用
# （obdclass模块）生成export，用作连接的上下文以便管理
00000020:00000040:0.0:1684440225.185340:0:3613:0:(genops.c:911:class_conn2export()) looking for export cookie 0xaefde85a03e1190b
00000020:00000040:0.0:1684440225.185340:0:3613:0:(lustre_handles.c:151:class_handle2object()) GET export 0000000061b8a918 refcount=3
# class_conn2export()调用class_handle2object()，通过cookie(hash)0xaefde85a03e1190b，从哈希表中得到export 0000000061b8a918
# 每次调用GET export，refount加一，调用PUT export，refount减一，用于统计调用次数，调用次数为0时回收
00000001:00000040:0.0:1684440225.185342:0:3613:0:(nodemap_handler.c:96:nodemap_getref()) GETting nodemap default(p=000000001a18b89f) : new refcount 4
# nodemap 将客户端传入的uid和gid转为服务端的uid和gid
# 如无特殊指定，客户端uid 1000传入服务端由nodemap转为得到uid 1000
# 如使用lctl设置nodemap规则，将uid 1000映射为uid 10000，则客户端uid 1000传入服务端由nodemap转为uid 10000
# 如果客户端传入的 uid 和 gid 在服务端中不存在，则应在 LDAP 上创建，或设置 identity_upcall 参数为 NONE 使客户端可信，即使服务端不存在该uid/gid也存储该文件
00000020:00000040:0.0:1684440225.185343:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000061b8a918 refcount=4
00000001:00000040:0.0:1684440225.185343:0:3613:0:(nodemap_handler.c:96:nodemap_getref()) GETting nodemap default(p=000000001a18b89f) : new refcount 5
00000001:00000040:0.0:1684440225.185344:0:3613:0:(nodemap_handler.c:112:nodemap_putref()) PUTting nodemap default(p=000000001a18b89f) : new refcount 4
00000001:00000040:0.0:1684440225.185350:0:3613:0:(tgt_lastrcvd.c:1068:tgt_client_new()) debugfs-OST0000: new client at index 1 (8320) with UUID 'c533e830-451b-4996-a72c-008e7736bf8b' generation 0
# （target模块）Add new client to the last_rcvd upon new connection.We use a bitmap to locate a free space in the last_rcvd file and initialize tg_export_data.
# 根据last_rcvd定位空闲空间并初始化
00000001:00000040:0.0:1684440225.185383:0:3613:0:(tgt_lastrcvd.c:559:tgt_new_client_cb_add()) callback GETting export 0000000061b8a918 : new cb_count 1
00000020:00000040:0.0:1684440225.185384:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000061b8a918 refcount=5
00000001:00000040:0.0:1684440225.185399:0:3613:0:(tgt_lastrcvd.c:640:tgt_client_data_update()) debugfs-OST0000: update last_rcvd client data for UUID = debugfs-OST0000_UUID, last_transno = 30064771094: rc = 0
# Update client data in last_rcvd, last_rcvd is a file that stores information about the last received operation on an Object Storage Target (OST) in Lustre
00000020:01000000:0.0:1684440225.185424:0:3613:0:(lprocfs_status_server.c:544:lprocfs_exp_setup()) using hash 00000000f2b17619
00000020:00000040:0.0:1684440225.185427:0:3613:0:(lprocfs_status_server.c:559:lprocfs_exp_setup()) Found stats 00000000a369d3cc for nid 192.168.242.10@tcp - ref 2
# （ofd模块）涉及到linux系统上procfs。在lustre源码中可知，在oss服务器上仅ofd模块会调用该函数
#  函数位置为ofd_obd.c:65:ofd_export_stats_init()，注释为Initialize OFD per-export statistics. This function sets up procfs entries for various OFD export counters. These counters are for per-client statistics tracked on the server.
# 将nid 192.168.242.10@tcp 进行hash，如果已经存在则返回，否则添加到hash列表中
00002000:00080000:0.0:1684440225.185428:0:3613:0:(ofd_obd.c:391:ofd_obd_connect()) debugfs-OST0000: get connection from MDS 0
# （ofd模块）Initialize new client connection.This function handles new connection to the OFD. The new export is created (in context of class_connect()) and persistent client data is initialized on storage.
# debugfs-OST0000和MDS0建立连接，在存储上初始化客户端的持久数据
00000020:00000040:0.0:1684440225.185429:0:3613:0:(obd_config.c:919:class_incref()) incref debugfs-OST0000 (000000005e629f65) now 7 - import
00000020:00000040:0.0:1684440225.185430:0:3613:0:(genops.c:1204:class_import_put()) import 0000000059a80778 refcount=1 obd=debugfs-OST0000
00000020:00000040:0.0:1684440225.185431:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000061b8a918 refcount=6
00000100:00000040:0.0:1684440225.185431:0:3613:0:(service.c:1069:ptlrpc_request_change_export()) RPC GETting export 0000000061b8a918 : new rpc_count 1
# Change request export and move hp request from old export to new
# ldlm_lib.c:1080:target_handle_connect()调用
00000100:00000040:0.0:1684440225.185435:0:3613:0:(connection.c:145:ptlrpc_connection_addref()) conn=000000006f7fb23f refcount 1 to 192.168.242.10@tcp
00000100:00000040:0.0:1684440225.185435:0:3613:0:(connection.c:133:ptlrpc_connection_get()) conn=000000006f7fb23f refcount 1 to 192.168.242.10@tcp
00000020:00000040:0.0:1684440225.185437:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000061b8a918 refcount=7
00000100:00000040:0.0:1684440225.185438:0:3613:0:(connection.c:145:ptlrpc_connection_addref()) conn=000000006f7fb23f refcount 2 to 192.168.242.10@tcp
00000100:00080000:0.0:1684440225.185471:0:3613:0:(import.c:85:import_set_state_nolock()) 0000000059a80778 8 changing import state from RECOVER to FULL
# Updates import \a imp current state to provided \a state value Helper function.
00000020:00000040:0.0:1684440225.185472:0:3613:0:(genops.c:979:class_export_put()) PUTting export 0000000061b8a918 : new refcount 6
00000020:00000040:0.0:1684440225.185473:0:3613:0:(obd_config.c:930:class_decref()) Decref debugfs-OST0000 (000000005e629f65) now 7 - find
00010000:00000040:0.0:1684440225.185476:0:3613:0:(ldlm_lib.c:3178:target_committed_to_req()) last_committed 0, transno 0, xid 1766202290688064
# xid事务标识符，是分配给客户端向服务器发送的每个请求的唯一编号。它用于匹配请求和回复，并检测重复或丢失的消息。
00000100:00000040:0.0:1684440225.185478:0:3613:0:(connection.c:145:ptlrpc_connection_addref()) conn=000000006f7fb23f refcount 3 to 192.168.242.10@tcp
00000100:00000040:0.0:1684440225.185479:0:3613:0:(niobuf.c:58:ptl_send_buf()) peer_id 12345-192.168.242.10@tcp
# niobuf.c:579:ptlrpc_send_reply()调用以上两个函数，该函数从request buffer中发送request reply
00000100:00000040:0.0:1684440225.185488:0:3613:0:(lustre_net.h:1943:ptlrpc_connection_put()) PUT conn=000000006f7fb23f refcount 2 to 192.168.242.10@tcp
00000100:00000040:0.0:1684440225.185489:0:3613:0:(lustre_net.h:2402:ptlrpc_rqphase_move()) @@@ move request phase from Interpret to Complete  req@0000000000b958be x1766202290688064/t0(0) o8->c533e830-451b-4996-a72c-008e7736bf8b@192.168.242.10@tcp:0/0 lens 520/416 e 0 to 0 dl 1684440325 ref 1 fl Interpret:/0/0 rc 0/0 job:''
# 改xid为1766202290688064的事务状态为Complete状态
00000100:00000040:0.0:1684440225.185492:0:3613:0:(service.c:1102:ptlrpc_server_finish_active_request()) RPC PUTting export 0000000061b8a918 : new rpc_count 0
# to finish an active request: stop sending more early replies, and release the request. should be called after we finished handling the request.
# 写操作结束
```

### 2.3.ost写流程总结

1. ptlrpc模块收到请求，事务xid为1766202290688064
2. ptlrpc模块从未执行队列中取得事务xid 1766202290688064，并将其状态设置为New
3. ldlm模块协助oss与客户端建立连接
4. obdclass模块生成与客户端连接对应的export 0000000061b8a918，本次连接cookie（hash值）为0xaefde85a03e1190b
5. nodemap处理export中客户端信息中的uid和gid
6. target模块初始化客户端信息
7. ofd模块与mds0建立连接，协调文件锁与布局，允许客户端写入持久数据
8. ptlrpc模块将事务xid 1766202290688064状态设置为Complete
9. ptlrpc释放请求，释放export

涉及到的模块有：

ptlrpc，ofd，obdclass，target，ldlm

## 3. ost读日志分析

### 3.1. 操作步骤

1. 在oss上执行 `lctl set_param debug=+info`使用info级别日志
2. 在客户端 `192.168.242.12`上写入文件 `echo 123456 > /mnt/client/test` 
3. 在oss上执行`lctl debug_kernel /tmp/log`清空日志
4. 为避免缓存因素影响，在另一客户端 `192.168.242.10`上使用`cat /mnt/client/test`命令向lustre文件系统内读数据
5. 在oss上执行`lctl debug_kernel /tmp/log`获取ost读日志

### 3.2. ost读日志及分析

```bash
00000100:00000040:0.0:1684492672.923251:0:3551:0:(events.c:358:request_in_callback()) incoming req@00000000d4eb3d26 x1766202290912192 msgsize 520
# 请求 00000000d4eb3d26，事务xid 1766202290912192
00000020:00000040:0.0:1684492672.923267:0:3613:0:(genops.c:911:class_conn2export()) looking for export cookie 0x0
00000100:00000040:0.0:1684492672.923280:0:3613:0:(lustre_net.h:2402:ptlrpc_rqphase_move()) @@@ move request phase from New to Interpret  req@00000000d4eb3d26 x1766202290912192/t0(0) o8-><?>@<unknown>:0/0 lens 520/0 e 0 to 0 dl 1684492772 ref 1 fl New:/0/ffffffff rc 0/-1 job:''
# 将请求 00000000d4eb3d26，事务xid 1766202290912192 改为New状态
00000020:00000040:0.0:1684492672.923288:0:3613:0:(obd_config.c:919:class_incref()) incref debugfs-OST0000 (000000005e629f65) now 7 - find
00010000:00080000:0.0:1684492672.923295:0:3613:0:(ldlm_lib.c:1365:target_handle_connect()) debugfs-OST0000: connection from c533e830-451b-4996-a72c-008e7736bf8b@192.168.242.10@tcp t0 exp 0000000028b3ebb2 cur 70445 last 0
# 客户端 c533e830-451b-4996-a72c-008e7736bf8b@192.168.242.10@tcp 与服务端 debugfs-OST0000 进行连接
00000020:00000040:0.0:1684492672.923297:0:3613:0:(lustre_handles.c:97:class_handle_hash()) added object 0000000069149a0d with handle 0xaefde85a03e11c0d to hash
# 由genops.c:1020:__class_new_export()调用，创建export，并添加其到hash表，然后返回得到一个到export的指针
# object数据结构为portals_handle，用作通过网络引用的对象的泛型类型，例如文件句柄或导出对象
# oss上与该客户端连接对应的export（object）为 0000000061b8a918，client连接cookie缓存为 0xaefde85a03e1190b（作为hash值使用），用于在服务端验证和定位object目标
00000020:00000040:0.0:1684492672.923306:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000069149a0d refcount=3
00000020:00000040:0.0:1684492672.923307:0:3613:0:(obd_config.c:919:class_incref()) incref debugfs-OST0000 (000000005e629f65) now 8 - export
00000020:00000040:0.0:1684492672.923308:0:3613:0:(genops.c:979:class_export_put()) PUTting export 0000000069149a0d : new refcount 2
00000020:00000080:0.0:1684492672.923309:0:3613:0:(genops.c:1359:class_connect()) connect: client c533e830-451b-4996-a72c-008e7736bf8b, cookie 0xaefde85a03e11c0d
# 上述class_handle_hash()函数到此处都是class_connect()的调用
# （obdclass模块）生成export，用作连接的上下文以便管理
00000020:00000040:0.0:1684492672.923309:0:3613:0:(genops.c:911:class_conn2export()) looking for export cookie 0xaefde85a03e11c0d
00000020:00000040:0.0:1684492672.923310:0:3613:0:(lustre_handles.c:151:class_handle2object()) GET export 0000000069149a0d refcount=3
00000001:00000040:0.0:1684492672.923311:0:3613:0:(nodemap_handler.c:96:nodemap_getref()) GETting nodemap default(p=00000000c0878ec0) : new refcount 5
# nodemap 处理客户端的uid和gid
00000020:00000040:0.0:1684492672.923312:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000069149a0d refcount=4
00000001:00000040:0.0:1684492672.923314:0:3613:0:(nodemap_handler.c:96:nodemap_getref()) GETting nodemap default(p=00000000c0878ec0) : new refcount 6
00000001:00000040:0.0:1684492672.923315:0:3613:0:(nodemap_handler.c:112:nodemap_putref()) PUTting nodemap default(p=00000000c0878ec0) : new refcount 5
00000001:00000040:0.0:1684492672.923322:0:3613:0:(tgt_lastrcvd.c:1068:tgt_client_new()) debugfs-OST0000: new client at index 2 (8448) with UUID 'c533e830-451b-4996-a72c-008e7736bf8b' generation 0
# （target模块）Add new client to the last_rcvd upon new connection.We use a bitmap to locate a free space in the last_rcvd file and initialize tg_export_data.
# 根据last_rcvd定位空闲空间并初始化
00000001:00000040:0.0:1684492672.923339:0:3613:0:(tgt_lastrcvd.c:559:tgt_new_client_cb_add()) callback GETting export 0000000069149a0d : new cb_count 1
00000020:00000040:0.0:1684492672.923340:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000069149a0d refcount=5
00000001:00000040:0.0:1684492672.923347:0:3613:0:(tgt_lastrcvd.c:640:tgt_client_data_update()) debugfs-OST0000: update last_rcvd client data for UUID = debugfs-OST0000_UUID, last_transno = 30064771097: rc = 0
# （target模块）Add new client to the last_rcvd upon new connection.
00000020:01000000:0.0:1684492672.923348:0:3613:0:(lprocfs_status_server.c:544:lprocfs_exp_setup()) using hash 00000000f2b17619
00000020:00000040:0.0:1684492672.923352:0:3613:0:(lprocfs_status_server.c:559:lprocfs_exp_setup()) Found stats 00000000a369d3cc for nid 192.168.242.10@tcp - ref 2
# （ofd模块）涉及到linux系统上procfs。在lustre源码中可知，在oss服务器上仅ofd模块会调用该函数
#  函数位置为ofd_obd.c:65:ofd_export_stats_init()，注释为Initialize OFD per-export statistics. This function sets up procfs entries for various OFD export counters. These counters are for per-client statistics tracked on the server.
# 将nid 192.168.242.10@tcp 进行hash，如果已经存在则返回，否则添加到hash列表中
00002000:00080000:0.0:1684492672.923352:0:3613:0:(ofd_obd.c:391:ofd_obd_connect()) debugfs-OST0000: get connection from MDS 0
# （ofd模块）Initialize new client connection.This function handles new connection to the OFD.  The new export is created (in context of class_connect()) and persistent client data is initialized on storage. 
# debugfs-OST0000和MDS0建立连接
00000020:00000040:0.0:1684492672.923354:0:3613:0:(obd_config.c:919:class_incref()) incref debugfs-OST0000 (000000005e629f65) now 9 - import
00000020:00000040:0.0:1684492672.923355:0:3613:0:(genops.c:1204:class_import_put()) import 00000000b319152a refcount=1 obd=debugfs-OST0000
# class_import_put will get rid of the additional connections
00000020:00000040:0.0:1684492672.923355:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000069149a0d refcount=6
00000100:00000040:0.0:1684492672.923356:0:3613:0:(service.c:1069:ptlrpc_request_change_export()) RPC GETting export 0000000069149a0d : new rpc_count 1
# Change request export and move hp request from old export to new
# （存疑）目前不知道为什么需要创建新的export
00000100:00000040:0.0:1684492672.923359:0:3613:0:(connection.c:145:ptlrpc_connection_addref()) conn=000000006f7fb23f refcount 1 to 192.168.242.10@tcp
00000100:00000040:0.0:1684492672.923360:0:3613:0:(connection.c:133:ptlrpc_connection_get()) conn=000000006f7fb23f refcount 1 to 192.168.242.10@tcp
00000020:00000040:0.0:1684492672.923361:0:3613:0:(genops.c:968:class_export_get()) GET export 0000000069149a0d refcount=7
00000100:00000040:0.0:1684492672.923362:0:3613:0:(connection.c:145:ptlrpc_connection_addref()) conn=000000006f7fb23f refcount 2 to 192.168.242.10@tcp
00000100:00080000:0.0:1684492672.923364:0:3613:0:(import.c:85:import_set_state_nolock()) 00000000b319152a 8 changing import state from RECOVER to FULL
# Updates import current state to provided state value Helper function.
# import的状态从RECOVER（The import is recovering from a disconnection and trying to replay requests that have not been committed by the server.）转为FULL（The import is fully recovered and connected to the server.）
# （存疑）
00000020:00000040:0.0:1684492672.923365:0:3613:0:(genops.c:979:class_export_put()) PUTting export 0000000069149a0d : new refcount 6
00000020:00000040:0.0:1684492672.923365:0:3613:0:(obd_config.c:930:class_decref()) Decref debugfs-OST0000 (000000005e629f65) now 9 - find
00010000:00000040:0.0:1684492672.923369:0:3613:0:(ldlm_lib.c:3178:target_committed_to_req()) last_committed 0, transno 0, xid 1766202290912192
# 当last_committed和transno一样时，等于对事务xid 1766202290912192 进行重播
00000100:00000040:0.0:1684492672.923370:0:3613:0:(connection.c:145:ptlrpc_connection_addref()) conn=000000006f7fb23f refcount 3 to 192.168.242.10@tcp
00000100:00000040:0.0:1684492672.923372:0:3613:0:(niobuf.c:58:ptl_send_buf()) peer_id 12345-192.168.242.10@tcp
00000100:00000040:0.0:1684492672.923379:0:3613:0:(lustre_net.h:1943:ptlrpc_connection_put()) PUT conn=000000006f7fb23f refcount 2 to 192.168.242.10@tcp
00000100:00000040:0.0:1684492672.923380:0:3613:0:(lustre_net.h:2402:ptlrpc_rqphase_move()) @@@ move request phase from Interpret to Complete  req@00000000d4eb3d26 x1766202290912192/t0(0) o8->c533e830-451b-4996-a72c-008e7736bf8b@192.168.242.10@tcp:0/0 lens 520/416 e 0 to 0 dl 1684492772 ref 1 fl Interpret:/0/0 rc 0/0 job:''
# 将请求 00000000d4eb3d26，事务xid 1766202290912192 改为Complete状态
00000100:00000040:0.0:1684492672.923383:0:3613:0:(service.c:1102:ptlrpc_server_finish_active_request()) RPC PUTting export 0000000069149a0d : new rpc_count 0
00000020:00000040:0.0:1684492672.923386:0:3613:0:(genops.c:979:class_export_put()) PUTting export 0000000069149a0d : new refcount 5
```

### 3.3.ost读流程总结

1. ptlrpc模块收到请求，事务xid为1766202290912192
2. ptlrpc模块从未执行队列中取得事务xid 1766202290912192，并将其状态设置为New
3. ldlm模块协助oss与客户端建立连接
4. obdclass模块生成与客户端连接对应的export 0000000069149a0d，本次连接cookie（hash值）为0xaefde85a03e11c0d
5. nodemap处理export中客户端信息中的uid和gid
6. target模块初始化客户端相关信息
7. ofd模块与mds0建立连接，协调文件锁与布局，允许客户端读取持久数据
8. ptlrpc模块将事务xid 1766202290912192状态设置为Complete
9. ptlrpc释放请求，释放export

涉及到的模块有：

ptlrpc，ofd，obdclass，target，ldlm