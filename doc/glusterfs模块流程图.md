# GlusterFS 各模块流程图（Mermaid）

本文用 **Mermaid** 描述 `glusterfs-10.5` 各主要模块的**典型控制流**与**数据流**，便于在支持 Mermaid 的 Markdown 预览中查看。流程为逻辑归纳，与具体补丁版本可能略有差异。

**相关文档：** [`glusterfs模块详解与实现原理.md`](./glusterfs模块详解与实现原理.md) · [`glusterfs副本与纠删码实现分析.md`](./glusterfs副本与纠删码实现分析.md) · [`glusterfs代码层次架构.md`](./glusterfs代码层次架构.md) · [`glusterfs架构概览.md`](./glusterfs架构概览.md)

---

## 1. `libglusterfs`：Volfile → Graph 激活

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  A[读取 volfile 内容<br/>路径或内存流] --> B[graph.y 词法语法分析<br/>preprocess + yyparse]
  B --> C[得到 glusterfs_graph_t<br/>xlator 链表 + 父子关系]
  C --> D[glusterfs_graph_prepare<br/>ctx / itable / 校验选项]
  D --> E{xlator_validate_rec<br/>dynload + options 校验}
  E -->|失败| X[返回错误 / 打日志]
  E -->|成功| F[glusterfs_graph_init<br/>依次 init 各 xlator]
  F --> G[挂接或切换 active graph<br/>brick mux 时 link 子图]
  G --> H[进入事件循环<br/>处理 FUSE/RPC I/O]
```

---

## 2. `libglusterfs`：单次 FOP 的 WIND / UNWIND

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 34}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  S[入口：上层 WIND<br/>如 fuse 或 server actor] --> N[分配 call_stack<br/>压入 call_frame]
  N --> T[THIS 指向当前 xlator<br/>调用 fops 成员]
  T --> Q{本层是否下发子层}
  Q -->|是| W[STACK_WIND<br/>新建子 frame]
  W --> C[子 xlator fops]
  C --> Q
  Q -->|否 / 叶子或发 RPC| L[执行实际 IO 或<br/>rpc_clnt_submit 等]
  L --> R[异步完成]
  R --> U[STACK_UNWIND / default_cb<br/>调用父 frame 的 ret]
  U --> P{还有父 frame}
  P -->|是| U
  P -->|否| F[FRAME_DESTROY<br/>STACK_DESTROY 回收]
```

---

## 3. `libglusterfs`：典型 `lookup` 与 inode（概念）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  L1[lookup FOP 进入] --> L2{inode_table<br/>缓存命中}
  L2 -->|命中| L3[刷新/补全 stat<br/>inode_ctx]
  L2 -->|未命中| L4[WIND 至子层 lookup]
  L4 --> L5[底层返回 stat+gfid]
  L5 --> L6[inode_link / insert<br/>绑定 dentry]
  L6 --> L3
  L3 --> L7[UNWIND 带 inode_t<br/>给上层]
```

---

## 4. `libglusterfs`：事件循环与 I/O 就绪（概念）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 22, 'rankSpacing': 38}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  E1[epoll_wait / io_uring<br/>gf-io 后端] --> E2{fd 就绪类型}
  E2 -->|FUSE| E3[fuse 读回调<br/>进入 graph 顶]
  E2 -->|socket| E4[rpc_transport_pollin<br/>→ rpcsvc / rpc_clnt]
  E2 -->|管道/定时器| E5[timer / 内部唤醒]
  E3 --> E6[继续 WIND 链]
  E4 --> E6
  E5 --> E6
```

---

## 5. `rpc/xdr`：编解码在调用链中的位置

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 24, 'rankSpacing': 40}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart LR
  subgraph 发送端
    M[C 结构体 + iovec] --> X1[XDR 编码<br/>libgfxdr]
    X1 --> B[字节流 buffer]
  end
  subgraph 传输
    B --> T[socket.la writev]
  end
  subgraph 接收端
    T --> R[读缓冲]
    R --> X2[XDR 解码<br/>xdr_to_*]
    X2 --> M2[C 结构体 / actor 参数]
  end
```

---

## 6. `rpc-lib`：服务端 `rpcsvc` 处理一帧 RPC

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 32}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  R0[RPC_TRANSPORT_MSG_RECEIVED] --> R1[rpcsvc_handle_rpc_call]
  R1 --> R2[rpcsvc_request_create<br/>xdr_to_rpc_call]
  R2 --> R3{解码与 RPC 版本}
  R3 -->|失败| RE[置错 GARBAGE_ARGS 等<br/>err_reply]
  R3 -->|成功| R4[rpcsvc_authenticate]
  R4 --> R5{鉴权}
  R5 -->|拒绝| RE
  R5 -->|通过| R6[rpcsvc_program_actor<br/>查 prog/proc 表]
  R6 --> R7{actor 是否存在}
  R7 -->|否| RE
  R7 -->|是| R8[入队或同步执行 actor_fn]
  R8 --> R9[编码 RPC 回复<br/>writev 回客户端]
```

---

## 7. `rpc-lib`：客户端 `rpc_clnt` 发送与回调

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 32}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  C0[protocol/client 等<br/>rpc_clnt_submit] --> C1[序列化 proc + 参数 XDR]
  C1 --> C2[挂 saved_frame<br/>关联 call_frame cookie]
  C2 --> C3[socket writev 发出]
  C3 --> C4[等待 MSG_RECEIVED]
  C4 --> C5[XDR 解码 reply]
  C5 --> C6{op_ret / 错误}
  C6 --> C7[调用注册 cbk<br/>UNWIND 上层 FOP]
  C6 -->|断连重试| C8[重连逻辑<br/>可能重发]
```

---

## 8. `rpc-transport/socket`：传输层事件

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 22, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  T0[注册 poll 回调<br/>rpc_transport_t] --> T1{事件}
  T1 -->|ACCEPT| T2[新建连接 trans]
  T1 -->|MSG_RECEIVED| T3[组 iovec<br/>通知 mydata: rpcsvc/rpc_clnt]
  T1 -->|DISCONNECT| T4[清理连接<br/>上层 disconnect 回调]
  T1 -->|MSG_SENT| T5[释放写缓冲 / 续写]
  T3 --> T6[进入 rpcsvc 或<br/>clnt 收包状态机]
```

---

## 9. `glusterfsd`：进程启动主流程（概括）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  P0[main / 解析 argp] --> P1[初始化 ctx<br/>日志 / 事件线程池]
  P1 --> P2[获取 volfile<br/>本地或 glusterd 拉取]
  P2 --> P3[glusterfs_graph_construct<br/>+ prepare + init]
  P3 --> P4{角色}
  P4 -->|客户端| P5[注册 FUSE 或 gfapi]
  P4 -->|brick| P6[mgmt 监听<br/>等待 glusterd 指令]
  P5 --> P7[event_loop 直至退出]
  P6 --> P7
```

---

## 10. `cli`：命令行到 glusterd

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  U0[用户输入 gluster …] --> U1[cli-cmd-parser 分派]
  U1 --> U2[构造 dict 参数<br/>如 hostname / volname]
  U2 --> U3[proc = GLUSTER_CLI_*]
  U3 --> U4[rpc_clnt_submit<br/>到本机或对端 24007]
  U4 --> U5[同步等待或回调<br/>cli_output 打印结果]
```

---

## 11. `xlators/mount/fuse`：内核读请求进入 graph

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  F0[/dev/fuse 可读] --> F1[fuse 库回调<br/>如 fuse_read]
  F1 --> F2[查 inode / fd 绑定<br/>Gluster 侧对象]
  F2 --> F3[STACK_WIND readv<br/>子 xlator]
  F3 --> F4[异步返回后<br/>copy_to_user 等]
```

---

## 12. `xlators/protocol/client`：FOP → 网络 RPC

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 32}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  PC0[收到父层 WIND<br/>如 readv] --> PC1[填请求结构<br/>gfid / offset / size]
  PC1 --> PC2[client_submit_vec<br/>或等价路径]
  PC2 --> PC3[rpc_clnt_submit<br/>+ XDR 编码]
  PC3 --> PC4[等待 reply]
  PC4 --> PC5[解码 iov<br/>填 rsp 结构]
  PC5 --> PC6[STACK_UNWIND<br/>op_ret / buffer]
```

---

## 13. `xlators/protocol/server`：网络 RPC → FOP

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 32}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  PS0[rpcsvc actor<br/>glusterfs3 FOP] --> PS1[解码参数到<br/>本地结构体]
  PS1 --> PS2[分配 call_frame<br/>绑定 trans 与 xlator]
  PS2 --> PS3[STACK_WIND 到<br/>FIRST_CHILD]
  PS3 --> PS4[子树 posix 等执行]
  PS4 --> PS5[UNWIND 回到 server]
  PS5 --> PS6[XDR 编码 reply<br/>rpcsvc_submit_generic]
```

---

## 14. `xlators/cluster/dht`：`lookup` 选子卷（简化）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  D0[lookup 入] --> D1[根据路径/gfid<br/>算 hash 或布局]
  D1 --> D2[选择目标 subvol<br/>索引]
  D2 --> D3[WIND lookup 到<br/>单个子卷]
  D3 --> D4{找到}
  D4 -->|是| D5[UNWIND 带 inode 信息]
  D4 -->|否 / linkfile| D6[重试或修复路径<br/>rebalance 相关]
```

---

## 15. `xlators/cluster/afr`：读路径多副本（简化）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 32}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  A0[readv 入] --> A1[选择 readable 子卷<br/>quorum / 锁状态]
  A1 --> A2[WIND 到一个或多个子卷]
  A2 --> A3{副本数据一致}
  A3 -->|是| A4[取其一 UNWIND]
  A3 -->|否| A5[按策略修复或<br/>返回错误 / 自愈标记]
```

---

## 16. `xlators/cluster/ec`：条带读（简化）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  E0[readv 入] --> E1[按条带算法拆 offset/len]
  E1 --> E2[并行或顺序 WIND<br/>多个子卷片段]
  E2 --> E3[纠删解码 / 合并 iov]
  E3 --> E4[UNWIND 完整用户缓冲]
```

---

## 17. `xlators/storage/posix`：写落到本地文件系统

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 32}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  Po0[writev 入] --> Po1[gfid → 真实路径<br/>或 fd 表项]
  Po1 --> Po2[posix_pwritev 等<br/>sys_writev / pwrite]
  Po2 --> Po3{成功}
  Po3 -->|是| Po4[更新 xattr/时间戳<br/>必要时]
  Po3 -->|否| Po5[UNWIND 错误码]
  Po4 --> Po6[UNWIND 成功计数]
```

---

## 18. `xlators/performance/write-behind`：合并写（概念）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 22, 'rankSpacing': 38}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  W0[writev 入] --> W1{可合并窗口}
  W1 -->|是| W2[缓冲 / 延迟 WIND]
  W1 -->|否或刷盘条件| W3[WIND 子层 writev]
  W2 --> W3
  W3 --> W4[UNWIND 聚合结果]
```

---

## 19. `xlators/mgmt/glusterd`：`peer probe` 双端（概括）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 30}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  subgraph 发起节点_CLI
    B0[gluster peer probe H] --> B1[CLI RPC GLUSTER_CLI_PROBE]
    B1 --> B2[glusterd_handler<br/>__glusterd_handle_cli_probe]
    B2 --> B3[glusterd_probe_begin<br/>friend_add / 建连]
  end
  subgraph 对端_glusterd
    C0[收到 gd1_mgmt_probe_req] --> C1[__glusterd_handle_probe_query]
    C1 --> C2{UUID 冲突 / 已属他集群}
    C2 -->|拒绝| C3[返回 op_errno]
    C2 -->|接受| C4[friend_add<br/>friend_sm / op_sm]
  end
  B3 <-->|管理 RPC| C0
```

---

## 20. `xlators/mgmt/glusterd`：卷操作状态机（高度简化）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  V0[收到 CLI volume op] --> V1[注入 op 事件<br/>glusterd_op_sm]
  V1 --> V2[锁 / 事务阶段<br/>各 peer 协商]
  V2 --> V3[volgen 写 volfile<br/>glusterd-volgen]
  V3 --> V4[通知或 fork glusterfsd<br/>proc-mgmt / svc-mgmt]
  V4 --> V5[提交 store<br/>glusterd-store]
  V5 --> V6[CLI 返回成功/失败]
```

---

## 21. `api/libgfapi`：应用打开卷（概括）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 22, 'rankSpacing': 38}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  G0[glfs_new 分配 glfs_t] --> G1[解析 volname / 服务器]
  G1 --> G2[构造与挂载侧类似 graph<br/>api xlator + protocol/client]
  G2 --> G3[glfs_init 激活 graph]
  G3 --> G4[glfs_open / read<br/>走 FOP 与 client RPC]
```

---

## 22. `geo-replication`：主从同步（概念）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 36}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  G0[主卷 changelog 或<br/>rsync/变更流] --> G1[gsyncd / syncdaemon]
  G1 --> G2[解析变更记录]
  G2 --> G3[在从卷执行等价操作<br/>gfapi 或挂载]
  G3 --> G4[校验游标 / 断点续传]
```

---

## 23. `events/glustereventsd`：事件上报（概念）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 22, 'rankSpacing': 38}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  E0[Gluster 进程或钩子<br/>产生事件 JSON] --> E1[glustereventsd 接收<br/>HTTP/插件管道]
  E1 --> E2[handlers 路由]
  E2 --> E3[Webhook / 文件 / 自定义插件]
```

---

## 24. 端到端：客户端 `readv` 到 brick 再返回

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 16, 'rankSpacing': 28}, 'themeVariables': {'fontSize': '13px'}}}%%
flowchart TB
  subgraph 客户端进程
    R1[FUSE read 回调] --> R2[performance 可选]
    R2 --> R3[cluster 选路]
    R3 --> R4[protocol/client<br/>RPC 编码]
  end
  R4 --> NET[TCP socket]
  NET --> R5[protocol/server<br/>actor]
  subgraph 存储节点进程
    R5 --> R6[cluster + features]
    R6 --> R7[posix pread]
  end
  R7 --> DISK[(brick 目录)]
  DISK --> R7
  R7 --> R6
  R6 --> R5
  R5 --> NET
  NET --> R4
  R4 --> R3
  R3 --> R2
  R2 --> R1
```

---

## 25. `tests/`：回归测试执行流（概括）

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 22, 'rankSpacing': 38}, 'themeVariables': {'fontSize': '14px'}}}%%
flowchart TD
  T0[run-tests.sh / prove] --> T1[启动临时 glusterd<br/>构造卷]
  T1 --> T2[执行 .t 内命令<br/>bash + assert]
  T2 --> T3[比对输出与<br/>退出码]
  T3 --> T4[teardown 杀进程<br/>清目录]
```

---

### 使用说明

- 在 **VS Code / Cursor** 中安装 Markdown 预览增强或对 Mermaid 原生支持的预览即可渲染。  
- 图较多时若单页卡顿，可按章节拆分阅读。  
- 若某张图在旧版 Mermaid 中报错，可去掉行首的 `%%{init:…}%%` 再试。
