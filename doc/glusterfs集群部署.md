# GlusterFS 集群搭建分析

本文从**运维步骤**与**源码中的控制面逻辑**两方面说明如何搭建 GlusterFS 集群，并与本仓库中的 `glusterfs-10.5` 实现对应。部署细节以你使用的发行版包与 [Gluster 官方文档](https://docs.gluster.org) 为准；此处给出通用流程与代码级因果链。

---

## 1. 集群模型（概念）

- **节点（Node）**：每台运行 **glusterd** 的机器；glusterd 负责本地卷元数据、与其它节点的 RPC、以及拉起 **glusterfsd**（brick 进程）。
- **信任池（Trusted Storage Pool）**：通过 `gluster peer probe` 把各节点的 glusterd 拉进同一逻辑集群；之后才能跨节点声明 brick。
- **卷（Volume）**：在信任池内把多个 **brick**（通常是某节点上的目录）组合成 distribute / replicate / dispersed 等拓扑；客户端通过 **glusterfs**（FUSE）或 **gfapi** 等访问。

---

## 2. 搭建前准备

### 2.1 时间与解析

- 各节点时间同步（NTP/chrony），避免证书、日志与一致性相关行为异常。
- 节点间建议用**稳定可解析**的主机名或 IP；`gluster peer probe` 使用的名字应能被其它节点解析或路由到 glusterd 监听地址。

### 2.2 网络与防火墙

glusterd、NFS、事件服务及 brick 端口在源码附带 firewalld 描述中有归纳，见 `glusterfs-10.5/extras/firewalld/glusterfs.xml`：

| 用途 | TCP 端口（示例） |
|------|-------------------|
| glusterd | 24007（默认监听端口在源码中定义为 `GF_DEFAULT_BASE_PORT`，见下节） |
| glusterd RDMA 管理（若使用） | 24008 |
| glustereventsd | 55555 |
| 内置 Gluster NFS | 38465–38469 |
| brick / 辅助端口 | 49152–60999 |

实际环境请按是否启用 NFS、RDMA、events 等裁剪放行规则。

### 2.3 软件安装

在**每个**节点安装 Gluster 服务端组件（包名因发行版而异，如 `glusterfs-server`），并启用 **glusterd** 服务（systemd：`systemctl enable --now glusterd`）。从源码安装流程见 `glusterfs-10.5/INSTALL`。

---

## 3. 默认端口与工作目录（与源码一致）

- **glusterd 默认端口**：`24007`，定义在 `libglusterfs/src/glusterfs/globals.h` 中的 `GF_DEFAULT_BASE_PORT`；RPC 默认监听与此一致（如 `rpc/rpc-lib/src/rpcsvc.h` 中 `RPCSVC_DEFAULT_LISTEN_PORT`）。
- **glusterd 工作目录（Linux）**：`GLUSTERD_DEFAULT_WORKDIR` 为 `DATADIR "/lib/glusterd"`（`DATADIR` 通常为 `/var/lib`，即常见路径 `/var/lib/glusterd`），见 `libglusterfs/src/glusterfs/glusterfs.h`。集群 UUID、peer 信息等均落盘于此树内（具体文件名由 glusterd 版本决定）。

---

## 4. 运维步骤：从空节点到可挂载卷

以下命令在**某一节点**上作为“操作入口”执行即可（CLI 会连本机 glusterd，由 glusterd 与其它节点协调）。

### 4.1 建立信任池

在**已启动 glusterd** 的节点 A 上，逐个加入其它节点：

```bash
gluster peer probe <node-b-主机名或IP>
gluster peer probe <node-c-主机名或IP>
gluster peer status
```

要点：

- **不要**对**本机**执行 probe（源码中会识别为 localhost 并返回 `GF_PROBE_LOCALHOST`）。
- 若某节点**已属于另一集群**（本机已有其它 peer），对端可能返回 `GF_PROBE_ANOTHER_CLUSTER`（见下节源码逻辑）。
- 所有待加入节点上的 glusterd 必须已运行且网络可达。

### 4.2 创建卷并启动

示例：三节点各一个 brick 的分布式卷（仅作语法示例，生产需按副本/仲裁等要求调整）：

```bash
gluster volume create <volname> transport tcp \
  node-a:/data/brick1 \
  node-b:/data/brick1 \
  node-c:/data/brick1
gluster volume start <volname>
gluster volume info <volname>
```

brick 路径必须是**已存在**的目录（或按版本要求使用 `force` 等选项），且注意 SELinux/AppArmor 与权限。

### 4.3 客户端挂载

在客户端安装 `glusterfs-client`（或等价包），例如：

```bash
mount -t glusterfs node-a:/<volname> /mnt/gluster
```

高可用场景下常在挂载 URI 中配置多备份服务器或使用其它挂载封装脚本（发行版可能提供 `mount.glusterfs`）。

---

## 5. 源码侧：`peer probe` 如何工作

理解集群搭建有助于排错（probe 挂住、UUID 冲突、分属两集群等）。

### 5.1 CLI 层

`gluster peer probe <host>` 在 CLI 中解析主机名并构造字典，通过管理 RPC 调用 `GLUSTER_CLI_PROBE` 过程，见 `cli/src/cli-cmd-peer.c`：`dict_set_str(..., "hostname", ...)` 后走 `cli_rpc_prog->proctable[GLUSTER_CLI_PROBE]`。

### 5.2 glusterd 接收 CLI probe

`xlators/mgmt/glusterd/src/glusterd-handler.c` 中 `__glusterd_handle_cli_probe`：

- 记录对端 `hostname`/`port`。
- 若配置了 `transport.socket.bind-address`，会校验 probe 地址与绑定地址关系；否则会检查是否为本地地址（避免 probe localhost）。
- 若该主机已是 peer，则返回“已是朋友”（`GF_PROBE_FRIEND`）。
- 否则调用 `glusterd_probe_begin`，必要时 `glusterd_friend_add`，并驱动 **`glusterd_friend_sm()`** 与 **`glusterd_op_sm()`** 状态机。

### 5.3 对端处理 inbound probe

同一文件中的 `__glusterd_handle_probe_query` 处理来自其它节点的 **gd1_mgmt_probe_req**：

- 解码 XDR 请求；默认端口逻辑为：若请求带 `port` 则用其值，否则使用 `GF_DEFAULT_BASE_PORT`（24007）。
- **UUID 校验**：若对端 UUID 与**本机 MY_UUID** 相同，返回 `GF_PROBE_SAME_UUID`（常见原因：错误克隆了 `/var/lib/glusterd`、镜像模板未清理 glusterd 状态）。
- 若本机已有 peer 列表且无法匹配当前 probe 上下文，可能返回 `GF_PROBE_ANOTHER_CLUSTER`（两池互斥，防止误并入）。
- 否则 `glusterd_friend_add` 等，并同样触发 `glusterd_friend_sm()` / `glusterd_op_sm()`。

### 5.4 自动化参考

`extras/devel-tools/devel-vagrant/ansible/roles/cluster/tasks/main.yml` 中示例即用 Ansible 在各节点执行 `gluster peer probe {{ item }}`，与手工步骤一致。

---

## 6. 常见故障与排查方向

| 现象 | 可能原因 | 建议 |
|------|-----------|------|
| probe 超时/连接失败 | 防火墙、glusterd 未监听、路由/NAT、绑定地址 | 检查 24007 连通、`ss -lntp`、bind-address 配置 |
| same UUID / probe 失败 | 多节点复用同一 glusterd 数据目录 | 重新初始化各节点 `glusterd` 工作目录或重新生成镜像 |
| another cluster | 目标节点已在其它信任池 | 先在该节点 `peer detach` / 清池或重建环境 |
| volume create 报 brick 不可用 | 目录不存在、权限、未 peer、主机名解析不一致 | 统一用与 `peer status` 一致的名字声明 brick |

日志：`journalctl -u glusterd` 与 `/var/log/glusterfs/` 下各组件日志（具体路径依安装而定）。

---

## 7. 小结

| 步骤 | 作用 |
|------|------|
| 各节点安装并启动 glusterd | 提供管理 RPC 与后续 brick 进程编排 |
| `gluster peer probe` | 在 glusterd 间建立信任池（CLI → `glusterd-handler.c` → friend/op 状态机） |
| `gluster volume create/start` | 生成 volfile、分配端口范围、启动 glusterfsd |
| 客户端 mount | 走 FUSE/挂载辅助，与数据面 translator 图交互 |

更深入的卷类型（副本数、arbiter、dispersed）、扩容、地理复制等属于运维专题，建议结合 [docs.gluster.org](https://docs.gluster.org) 与 `glusterfs-10.5/doc/` 下开发者文档阅读。

---

*本文与 `glusterfs-10.5` 源码中的常量与处理路径对应；生产部署请以发行版安全基线与官方发行说明为准。*
