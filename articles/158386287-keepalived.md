# KeepAlived 高可用详解与实战

## 文章导读
Keepalived 是一款基于 **VRRP 协议**实现高可用的开源工具，不仅能通过 VIP 漂移解决单点故障，还能为 LVS（IPVS）提供负载均衡高可用，并支持自定义脚本做业务级健康检测。本文从核心理论、五种实验场景、排错方法到知识体系全面讲解，适合需要构建高可用集群的读者。

## 一、核心理论

### 1. 高可用集群基础
- **目标**：解决单点故障（SPoF），提升系统可用性（SLA）
- **公式**：可用性 `A = MTBF / (MTBF + MTTR)`，核心是降低平均修复时间（MTTR）
- **实现机制**：通过冗余机制（主/备 active/passive 或双主 active/active）和**心跳检测（Heartbeat）**实现故障转移

### 2. VRRP 协议（Virtual Router Redundancy Protocol）
Keepalived 的核心实现协议，用于解决静态网关单点风险。

**关键术语**
- **VRID**：虚拟路由器标识（0-255），同组必须一致
- **VIP**：虚拟 IP，对外服务的地址
- **VMAC**：虚拟 MAC（格式：`00-00-5e-00-01-VRID`）
- **Priority**：优先级（1-254），值越大越优先成为 Master

**工作模式**
- **抢占式（Preempt）**：默认模式。高优先级节点恢复后立即抢回 Master 角色（可能导致网络抖动）
- **非抢占式（Nopreempt）**：高优先级节点恢复后不抢回 Master，维持现状。配置要求：所有节点 `state` 必须设为 `BACKUP`
- **延迟抢占**：高优先级恢复后，等待指定时间（`preempt_delay`）再抢回

**通信方式**
- **多播（Multicast）**：默认方式，使用组播地址（如 `224.0.0.18`）
- **单播（Unicast）**：需配置 `unicast_src_ip` 和 `unicast_peer`，减少网络拥塞。注意：启用单播时不能开启 `vrrp_strict` 模式

### 3. Keepalived 架构与功能
**核心组件**
- **VRRP Stack**：心跳通告
- **Checkers**：健康检测
- **IPVS Wrapper**：生成负载均衡规则
- **System Call**：脚本调用

**主要功能**
- 基于 VRRP 实现 VIP 漂移
- 为 LVS（IPVS）集群提供高可用
- 对后端 Real Server（RS）进行健康状态检测
- 支持自定义脚本监控任意服务（如 Nginx、HAProxy）

## 二、核心实验与部署

### 1. 环境准备
- **时间同步**：所有节点必须同步（ntp/chrony）
- **网络互通**：关闭防火墙/SELinux，确保节点间通信
- **安装**：`dnf install keepalived -y`
- **配置文件**：`/etc/keepalived/keepalived.conf`
  - `Global_defs`：全局配置（邮件通知、router_id）
  - `VRRP_instance`：定义虚拟路由器（state、interface、priority、vip）
  - `Virtual_server`：定义 LVS 集群（可选）

### 2. 实验场景详解

**场景 A：主/备（Master/Slave）单主架构**
- Master：`state MASTER`，`priority 100`
- Backup：`state BACKUP`，`priority 80`
- 两者 `virtual_router_id` 和 `auth_pass` 必须一致
- **测试**：停止 Master 的 keepalived 服务，观察 Backup 是否获取 VIP（`ip addr`）；重启 Master，观察是否发生抢占

**场景 B：非抢占模式（Nopreempt）**
目的：防止 VIP 频繁漂移。
- 所有节点 `state` 均设为 `BACKUP`
- 添加 `nopreempt` 参数
- 注释掉 `vrrp_strict`（否则可能无法启动）
- **现象**：原 Master 故障恢复后，即使优先级高，也不会抢回 VIP，直到当前 Master 也故障

KA1 配置示例：
```
[root@KA1 ~]# vim /etc/keepalived/keepalived.conf
vrrp_instance WEB_VIP {
    state BACKUP       # 非抢占模式互为 backup
    interface eth0
    virtual_router_id 51
    nopreempt          # 启动非抢占模式
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.25.254.100/24 dev eth0 label eth0:0
    }
}
```

KA2 配置（`priority 80`，其余相同）：
```
[root@KA2 ~]# vim /etc/keepalived/keepalived.conf
vrrp_instance WEB_VIP {
    state BACKUP
    interface eth0
    virtual_router_id 51
    nopreempt
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.25.254.100/24 dev eth0 label eth0:0
    }
}
```

测试：
```
[root@KA1 ~]# systemctl start keepalived.service
[root@KA2 ~]# systemctl start keepalived.service
[root@KA1 ~]# ifconfig
# 可见 eth0:0 上持有 VIP 172.25.254.100
```
停止 KA1 后，KA2 接管 VIP：
```
[root@KA1 ~]# systemctl stop keepalived.service
[root@KA2 ~]# ifconfig
# KA2 的 eth0:0 上出现 172.25.254.100
```
> 由于启用了 nopreempt，KA1 重启后不会抢占回 VIP。

**场景 C：双主架构（Master/Master）**
目的：充分利用两台服务器资源，互为备份。
- 定义两个 VRRP 实例（如 `VI_1` 和 `VI_2`），对应两个 VIP（VIP1、VIP2）
- 节点 1：`VI_1` 为 MASTER（高优先级），`VI_2` 为 BACKUP（低优先级）
- 节点 2：`VI_1` 为 BACKUP（低优先级），`VI_2` 为 MASTER（高优先级）
- **结果**：节点 1 主导 VIP1，节点 2 主导 VIP2，任意一台故障，另一台接管所有 VIP

**场景 D：结合 LVS（IPVS）实现负载均衡高可用**
- 配置结构：在 `virtual_server` 块中定义 VIP 和端口，以及后端 `real_server`
- **健康检测**：
  - `TCP_CHECK`：检测端口连通性
  - `HTTP_GET`：检测 URL 返回状态码（如 200）
- **工作流程**：Keepalived 监控 RS 状态，自动从 IPVS 表中剔除故障节点；若所有 RS 故障，流量转至 `sorry_server`
- **LVS-DR 模式准备**：后端 RS 需在 `lo` 接口绑定 VIP，并调整 ARP 参数（`arp_ignore=1`，`arp_announce=2`）

**场景 E：自定义脚本监控（VRRP Script）**
目的：监控非标准服务（如 HAProxy、Nginx 进程），根据结果动态调整优先级。
- **配置步骤**：
  1. 定义脚本（`vrrp_script`）：设置脚本路径、执行间隔（`interval`）、权重变化（`weight`，通常设为负数）、失败/成功次数（`fall`/`rise`）
  2. 调用脚本（`track_script`）：在 `vrrp_instance` 中引用定义的脚本名
- **逻辑**：脚本返回非 0（失败）→ 节点优先级降低（Priority + weight）→ 若低于备用节点，则触发 VIP 漂移

### 3. 调试与排错

**日志查看**
- 修改 `/etc/sysconfig/keepalived` 添加 `-D -S 6` 开启详细日志
- 配置 rsyslog 将 `local6` 设施输出到 `/var/log/keepalived.log`

**抓包分析**
使用 `tcpdump` 监听 VRRP 组播或单播包，观察 Advertisement 报文中的优先级和状态：
```bash
tcpdump -i eth0 -nn host 224.0.0.18    # 多播
```

**配置检查**
```bash
keepalived -t -f /etc/keepalived/keepalived.conf    # 检查语法
```

## 三、知识体系重构

### 1. 核心定位：双引擎驱动
Keepalived 集成两大核心引擎，分别解决「可用性」与「性能」问题：

| 核心引擎 | 功能 | 关键技术 | 应用场景 |
|---|---|---|---|
| 高可用（HA） | 消除单点故障，确保服务连续性 | VRRP 协议 + VIP 漂移 | Nginx/MySQL/Redis 主备切换 |
| 负载均衡（LB） | 分发流量，提升系统吞吐能力 | LVS（IPVS）框架 | 四层 TCP/UDP 流量调度 |

> 注：虽然具备负载均衡能力，但生产环境中 Keepalived 更多作为 LVS 的高可用伴侣或独立服务的 HA 解决方案使用。

### 2. 理论基石：VRRP 协议机制
Keepalived 的高可用能力完全建立在 **VRRP** 之上。

**工作原理**
- **虚拟路由器组**：多台物理服务器组成一个逻辑上的「虚拟路由器」
- **对外表现**：整个组对外只暴露一个虚拟 IP（VIP）。客户端仅与 VIP 交互，对后端物理节点的切换无感知

**角色与选举**
- **Master（主节点）**：持有 VIP，响应 ARP 请求，实际承载业务流量；周期性发送 VRRP 通告报文（Advertisement）
- **Backup（备节点）**：监听 Master 的通告，处于待命状态；若超时未收到 Master 通告，则发起选举并接管 VIP

**选举规则**
1. 基于优先级（Priority）：数值越大越优先（范围 1-254）
2. 优先级相同时，比较接口 IP 地址大小

### 3. 进阶机制：智能健康检查
这是 Keepalived 区别于传统心跳检测的关键特性，实现了从「进程级」到「业务级」的监控。

**双层检测逻辑**
- **VRRP 层检测**：仅确认对端 Keepalived 进程是否存活（网络连通性）
- **业务层检测（Track Script）**：执行自定义脚本（如检查 Nginx 端口、MySQL 连接、URL 状态码）；若检测到本地业务异常，Keepalived 主动降低自身优先级（通常减去一个权重值）；优先级低于备节点 → 触发 VIP 漂移 → 流量自动切换至健康节点

**核心价值**：避免「Keepalived 进程正常但业务服务已挂」的脑裂假象，确保 VIP 始终指向健康的服务器。

### 4. 关键特性
- **轻量高效**：资源占用极低，专为高性能场景设计
- **标准协议**：基于成熟的 VRRP RFC 标准，兼容性好
- **高度灵活**：支持任意自定义脚本，可集成 Redis、MySQL、HAProxy 等任何 TCP/HTTP 服务
- **秒级切换**：故障检测与 VIP 漂移通常在秒级完成，业务中断时间极短
- **开源免费**：社区活跃，无授权费用，Linux 发行版默认源即可安装

### 5. 典型应用场景矩阵

| 场景分类 | 具体架构 | 说明 |
|---|---|---|
| Web 服务 HA | Nginx/Apache + Keepalived | 实现 Web 入口主备或双活，用户访问 VIP 不受单台宕机影响 |
| 数据库 HA | MySQL/PostgreSQL + Keepalived | 配合主从复制，通过脚本判断主库读写状态，实现 VIP 随主库漂移 |
| LVS 集群 HA | LVS Director + Keepalived | 经典组合，为 LVS 调度器提供高可用，防止调度器成为单点故障 |
| 中间件 HA | Redis/Memcached + Keepalived | 为有状态或无状态缓存服务提供虚拟入口，简化客户端配置 |
| 任意 TCP 服务 | 自定义服务 + 脚本检测 | 只要服务可通过脚本检测连通性，均可纳入 Keepalived 管理 |

## 四、小结
Keepalived 的核心价值在于**「以最小的成本实现最大的可靠性」**。它利用 VRRP 协议解决了「谁提供服务」（VIP 归属）的问题，利用健康检查脚本解决了「谁能提供服务」（业务健康度）的问题，是构建高可用集群不可或缺的基础组件。