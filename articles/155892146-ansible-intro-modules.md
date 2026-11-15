# RHCSE Ansible 入门与模块详解

## 文章导读
Ansible 是一款基于 Python、基于 SSH 的自动化运维工具，无需代理即可批量管理多台主机。本文从 **Ansible 的功能与特性、架构原理** 讲起，覆盖**环境准备、inventory 主机清单、免密登录（密码/密钥）、ansible.cfg 配置、模块帮助手册 ansible-doc、命令行选项与提权、指令执行流程、Collection 体系、模块分类、幂等性原理**，并深入讲解 **command 模块** 与 **Playbook 调用模块**，适合希望系统掌握 Ansible 自动化的运维与开发人员。

## 一、快速入门与功能

**Ansible 的核心功能：**
- **批量执行远程命令**：可以对远程的多台主机同时进行命令的执行
- **批量安装和配置软件服务**：可以对远程的多台主机以自动化方式配置和管理各种服务
- **编排高级的企业级复杂 IT 架构任务**：Ansible 的 Playbook 和 Role 可以轻松实现大型的 IT 复杂架构
- **提供自动化运维工具的开发 API**：有很多运维工具（如 JumpServer）就是基于 Ansible 实现自动化管理功能

**Ansible 特性：**
- **模块化**：调用特定的模块完成特定任务，支持自定义模块，可使用任何编程语言写模块
- **基于 Python 语言实现**
- **部署简单**：基于 Python 和 SSH（默认已安装），agentless，无需代理，不依赖 PKI（无需 SSL），安全，基于 OpenSSH
- **幂等性**：一个任务执行 1 遍和执行 n 遍效果一样，不因重复执行带来意外情况（此特性非绝对支持 Playbook 编排任务），YAML 格式，编排任务，支持丰富的数据结构
- **较强大的多层解决方案 Role**

## 二、Ansible 架构

**架构拓扑**
```
控制节点 (Ansible 控制机)  <-->  受控节点 (目标服务器)
     |                              |
     └─ 执行 Ansible 命令 ──────────┘

控制节点命令 → SSH 连接 → 受控节点执行 → 返回结果
  ↓            ↓            ↓
ansible命令   网络通信    实际安装配置
```

**部署流程**
1. **环境准备（前置步骤）**
   - 控制节点安装 Ansible
   - 控制节点配置 SSH 免密登录到受控节点
   - 控制节点编写 inventory 文件（`/etc/ansible/hosts`）定义受控节点
2. **执行部署（核心步骤）**
   - 选择纯剧本或剧本+角色方式
   - 执行 `ansible-playbook` 命令部署

**整体架构分层**

**1. 左侧：输入 / 数据源**
- **USERS（用户）**：Ansible 的操作发起者，可从「公有/私有云」「CMDB（配置管理数据库）」获取资产信息，也可直接操作 Ansible
- **CMDB**：配置管理数据库，存储企业所有 IT 资产（服务器、网络设备等）的信息，会将资产数据同步到 Ansible 的「Inventory（主机清单）」
- **ANSIBLE PLAYBOOK**：用户编写的自动化任务编排文件（YAML 格式），是 Ansible 执行复杂任务的核心输入

**2. 中间：Ansible 自动化引擎（核心）**
- **INVENTORY（主机清单）**：存储受控节点（服务器、网络设备）的信息（IP、分组、变量等），数据来源可以是用户手动维护、CMDB 同步、云平台自动拉取
- **API**：Ansible 提供的编程接口，用于和其他系统（如运维平台、CI/CD 工具）集成，实现自动化流程的对接
- **MODULES（模块）**：Ansible 的「功能单元」，是执行具体任务的最小工具（如 yum 模块装软件、service 模块启停服务），Ansible 通过调用模块完成对受控节点的操作
- **PLUGINS（插件）**：扩展 Ansible 功能的组件（如连接插件控制 SSH 连接、回调插件自定义输出格式），是 Ansible 灵活性的关键

**3. 右侧：管理对象**
- **HOSTS（受控主机）**：服务器、虚拟机等计算节点，是 Ansible 的核心管理目标
- **NETWORKING（网络设备）**：交换机、路由器等网络硬件，Ansible 也支持通过专用模块管理这类设备

## 三、实验：安装 Ansible
```bash
[root@rhce ~]# wget https://mirrors.aliyun.com/epel/epel-release-latest-9.noarch.rpm
[root@rhce ~]# rpm -ivh epel-release-latest-9.noarch.rpm
[root@rhce ~]# yum install ansible -y
```

## 四、inventory 主机清单与连接建立

`/etc/ansible/hosts` 是 Ansible 管理受控节点的**主机清单文件**，支持多种写法以适配不同场景。

### 1. 密码登录

**写法1：**
```bash
[root@rhce ~]# vi /etc/ansible/hosts
192.168.223.152  ansible_ssh_user=root  ansible_ssh_port=22  ansible_ssh_pass=redhat

[root@rhce /]# vim /etc/ansible/ansible.cfg
[defaults]
host_key_checking=False

[root@rhce /]# ansible 192.168.223.152 -m ping
192.168.223.152 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
> `ansible_facts` 是 Ansible 中的「事实集合」字段，专门存储从受控节点收集到的系统级基础信息（如操作系统版本、IP、Python 解释器路径、内存等），称为「Ansible Facts」；`discovered_interpreter_python` 是 Ansible 自动探测到的可用 Python 解释器路径。

**Ansible 输出颜色与状态对应：**

| 颜色 | 核心状态 | 含义 |
|---|---|---|
| 绿色 | SUCCESS | 操作成功，且没有产生任何状态变化（如目标已处于目标状态） |
| 黄色 | CHANGED | 操作成功，且产生了实际状态变化（如之前未挂载，现在成功挂载） |
| 红色 | FAILED | 操作失败（会明确显示错误信息，如权限不足、设备不存在） |

**写法2：分组写法**
```bash
[root@rhce /]# vim /etc/ansible/hosts
192.168.223.152  ansible_ssh_pass=redhat

[server]
192.168.223.152  ansible_ssh_pass=redhat
192.168.223.151  ansible_ssh_pass=redhat

[root@rhce /]# vim /etc/ansible/ansible.cfg
[defaults]
host_key_checking=False

[root@rhce /]# ansible server -m ping
```
> 执行后会分别对 `[server]` 组内的两台主机返回 SUCCESS 结果。

**写法3：自定义 ansible.cfg**
```bash
[root@master ~]# ansible-config init --disabled > /ansible.cfg
# 把 /etc/ansible/ansible.cfg 里的配置删除，配置 /ansible.cfg，切换到当前目录后，ansible 就不读取 /etc/ansible.cfg，只读取 /ansible.cfg 并生效
[root@master ~]# vim /ansible.cfg
[defaults]
host_key_checking=False

[root@master ~]# cd /
[root@rhce /]# vim /etc/ansible/hosts
[server]
192.168.223.15[1:2]  ansible_ssh_pass=redhat

[root@rhce /]# ansible server -m ping
```

**ansible.cfg 的加载顺序：**
1. 环境变量 `ANSIBLE_CONFIG` 指定的路径
2. 当前工作目录下的 `ansible.cfg`（项目目录下的配置）
3. 用户家目录的 `.ansible.cfg`
4. 系统全局配置 `/etc/ansible/ansible.cfg`

**写法4：指定 inventory 文件**
```bash
[root@rhce /]# vim /hosts
[newserver]
192.168.223.15[1:2]  ansible_ssh_user=redhat ansible_ssh_pass=redhat

[root@rhce /]# vim /ansible.cfg
[defaults]
host_key_checking=False

[root@rhce /]# cd /
[root@rhce /]# ansible newserver -i /hosts -m ping
```

**清单来源的加载优先级（从高到低）：**
1. 执行命令时用了 `-i /tmp/test-hosts`，Ansible 只会读取该指定文件
2. 没指定命令行 `-i`，但环境变量 `ANSIBLE_INVENTORY` 配置了路径，则用环境变量指定的 hosts
3. 以上都没配置，才读 `ansible.cfg` 中 `inventory` 项指定的 hosts
4. 只有前 3 项都没配置时，才用系统默认的 `/etc/ansible/hosts`

### 2. 密钥登录（免密）
```bash
[root@rhce /]# cat /hosts
[newserver]
192.168.223.15[1:2]  ansible_ssh_private_key_file=/root/.ssh/id_rsa

[root@rhce /]# cat /ansible.cfg
[defaults]
remote_user = root
timeout = 3
host_key_checking=False

[root@rhce /]# ssh-keygen -t rsa
# 生成公钥 /root/.ssh/id_rsa.pub
```

**用 authorized_key 模块推送公钥（命令行）：**
```bash
[root@rhce /]# ansible newserver -m authorized_key -a "user=root state=present key='{{ lookup('file', '/root/.ssh/id_rsa.pub') }}'" -e "ansible_ssh_pass=redhat" -i /hosts
```
> **说明：**
1. 命令行中 `key=` 后的公钥内容：若用 `lookup` 函数，需保证引号嵌套正确（外层双引号，内层单引号）；若直接粘贴公钥字符串，需去掉 `{{ lookup(...) }}`，直接写公钥。
2. **权限问题**：`authorized_key` 模块会自动将 `authorized_keys` 文件权限设为 600（符合 SSH 要求），无需手动修改。
3. **免密验证**：推送完成后，执行 `ansible newserver -m ping` 验证免密是否生效（无需再传 `-e ansible_ssh_pass`）。

**用 YAML 格式重写 authorized_key 模块：**
```bash
[root@rhce /]# cat /auth.yaml
---
- hosts: newserver      # 目标主机组（对应清单里的 [newserver]）
  gather_facts: no       # 关闭事实收集（首次推送公钥时，受控节点可能未配置 Python 依赖）
  vars:
    ansible_ssh_pass: "redhat"
    ansible_host_key_checking: False
  tasks:
    - name: 给root用户添加SSH公钥（Ansible管理）
      authorized_key:
        user: root                       # 指定目标用户（必填）
        state: present                   # 确保公钥存在（添加）
        key: "{{ lookup('file', '/root/.ssh/id_rsa.pub') }}"   # 读取本地公钥

[root@rhce /]# ansible-playbook /auth.yaml -i /hosts
```

## 五、模块帮助手册 ansible-doc

Ansible 自带了模块搜索、列表、详情查询的命令，靠关键词就能快速定位：

| 场景 | 命令示例 | 优点 |
|---|---|---|
| 记得关键词（key/ssh） | `ansible-doc -l | grep -i key` | 最快，直接定位 |
| 完全忘关键词 | `ansible-doc -l | less` | 按功能翻找，无记忆成本 |
| 找到后验证用法 | `ansible-doc authorized_key` | 看参数/示例，避免用错 |

```bash
# 查看指定模块的完整帮助（比如 file 模块）
ansible-doc file

# 只看模块的参数说明（精简版）
ansible-doc -s file
```

**帮助手册的组成（统一结构）：**
1. **基础信息区**：模块名称 + 简介、官方分类、版本信息
2. **参数说明区（核心，占比 90%）**：`=` 参数名 表示必选，`-` 参数名 表示可选
3. **示例区（EXAMPLES）**：手册末尾提供高频实操示例
4. **返回值区（RETURN）**：说明模块执行后返回的数据

**查看帮助手册的核心逻辑（按优先级）**
① 先看**必选参数**（`=` 开头的必须传）；② 再看**可选参数**的默认值；③ 看参数的**适用条件**；④ 看**参数别名**（如 `path` 可用 `dest` 替代）；⑤ 参考**示例区**。

> 打开 `ansible-doc file` 后进入交互模式，可用快捷键：`/参数名` 搜索、上下箭头滚动、`q` 退出。

## 六、命令行调用模块与提权

**命令行语法**
```bash
ansible [-a MODULE_ARGS] [-m MODULE_NAME] [-i INVENTORY]
ansible <主机组/单个IP> -m <模块> -a <模块参数>
```

**常用命令行选项：**

| 选项 | 说明 |
|---|---|
| `--version` | 显示版本 |
| `-m module` | 指定模块，默认为 command |
| `-v` | 显示详细过程，`-vv`/`-vvv` 显示更详细内容 |
| `--list-hosts`（可简写 `--list`） | 显示主机列表 |
| `-k, --ask-pass` | 提示输入 SSH 连接密码（默认使用密钥验证） |
| `-u, --user=REMOTE_USER` | 指定远程执行任务的用户，默认为 root |
| `-b, --become` | 提权执行，默认提权到 root 用户（能否提权看受控节点 `/etc/sudoers`） |
| `--become-user=USERNAME` | 指定 sudo 的 runas 用户，默认为 root |
| `-K, --ask-become-pass` | 提示输入 sudo 操作的口令 |

**范例：将普通用户提升权限**
```bash
# 先在受控节点 sudo 授权
[root@node1 ~]# vim /etc/sudoers
hehe  ALL=(ALL)  NOPASSWD: ALL
[root@node1 ~]# echo redhat | passwd hehe --stdin
```
> 注意：`-u hehe` 是受控节点的用户，且该用户需写入 `/etc/sudoers`。Ansible 要成功执行模块，需先确保能连接到受控节点。

```bash
# 以 hehe 用户连接，利用 sudo 提权执行 cat /etc/shadow
[root@rhce ~]# ansible 192.168.223.151 -m shell -a 'cat /etc/shadow' -u hehe -k -b
SSH password:
192.168.223.151 | CHANGED | rc=0 >>
root:$6$...::0:99999:7:::
```
> 若 sudoers 未配置 NOPASSWD，则需加 `-K` 输入 sudo 口令：`ansible 192.168.223.151 -m shell -a 'cat /etc/shadow' -u hehe -k -b -K`。

## 七、ansible 指令执行过程

**完整流程**
控制节点解析命令 → 读主机清单 → 建并发进程 → SSH 连目标主机 → 传模块脚本 → 提权/检查后执行 → 收集结果 + 清理临时文件 → 输出最终结果

**简化流程步骤：**

| 步骤 | 具体操作 | 关键实操补充 |
|---|---|---|
| 1. 加载配置文件 | 读取 Ansible 全局配置、模块默认参数、用户自定义配置 | 自定义 ansible.cfg 优先级高于全局配置；校验配置合法性 |
| 2. 加载模块文件 | 从内置模块目录读取指定模块的核心逻辑 | 校验模块参数（如 file 模块必传 path）；自定义模块放 `~/.ansible/modules/` 优先加载 |
| 3. 生成临时 py 文件并传输 | 将模块脚本打包为临时 Python 文件，通过 SSH 传输到远程临时目录（默认 `~/.ansible/tmp/`） | 临时文件命名含随机字符串；可用 `remote_tmp` 自定义；command/shell 无需生成 py 文件 |
| 4. 给文件添加执行权限 | 远程主机对临时文件执行 `chmod +x` | 仅给临时文件加权限；执行后立即清理 |
| 5. 执行并返回结果 | 远程运行临时脚本，将结果（状态/返回码/输出）传回控制节点 | 状态分 changed/ok/failed 三类；提权 `-b`、检查 `-C` 在此生效 |
| 6. 删除临时文件 | 确认结果接收后，触发远程删除临时脚本 | 执行失败可能残留临时文件，需手动清理；调试可设 `keep_remote_files=yes` |

## 八、Collection 体系与模块历史

**版本划分依据：**
- **早期版本**：Ansible 1.x ~ 2.0（2012-2016 年），功能基础、模块数量少，未引入 Collection 体系
- **新版本**：Ansible 2.10+（2020 年后）引入 **Collection（集合）** 体系，后续版本号调整为「Ansible 包版本」，对应核心引擎是 ansible-core 2.14 左右，支持 Collection、更多模块和企业级功能
> 之前用到的 `ansible.posix.authorized_key` 模块，就是调用了 `ansible.posix` 这个 Collection 里的模块，即 Collection 体系下的模块调用方式：**命名空间.集合名.模块名**。

**Collection 出现前的问题**
早期 Ansible（2.10 之前）的模块「零散存放」：官方模块放在核心代码里，第三方模块需手动下载、易丢难更新；不同模块版本无法单独管理；模块命名容易冲突。

**Collection 是什么？**
Collection 是一个「资源包」，每个 Collection 打包对应场景下的所有相关资源，有唯一身份标识：**命名空间.集合名**（如 `ansible.posix`、`community.docker`）：
- **命名空间**：区分归属方（ansible 官方、community 社区、cisco 厂商）
- **集合名**：区分功能场景（posix 为 POSIX 系统工具、docker 为 Docker 管理）

**举例**：`ansible.posix`（官方 POSIX 工具集合，含 authorized_key）、`community.docker`（社区 Docker 集合，含 docker_container）、`cisco.aci`（Cisco 厂商 ACI 网络设备集合）。

**背景：类 UNIX 系统与 POSIX**
UNIX 是 1969 年由贝尔实验室的肯·汤普森、丹尼斯·里奇开发的多用户、多任务分时操作系统，后版权被 AT&T 收回并闭源收费。此后两类人群推动了「类 UNIX 系统」的诞生：
- **学术界**（如加州大学伯克利分校）：基于早期 UNIX 开源分支开发出 **BSD 系统**，成为最早的类 UNIX 系统之一；
- **普通用户/开发者**：1991 年林纳斯·托瓦兹基于 Minix 启发开发出 **Linux 内核**，后与 GNU 工具组合形成各种 Linux 发行版（Ubuntu、SUSE、RHEL、CentOS）。

这些「功能、用法与 UNIX 高度相似，但不是官方 UNIX」的系统统称为**类 UNIX 系统**（含 Linux 系、BSD 系、商业类 UNIX）。为解决兼容问题，**IEEE 制定了 POSIX 标准**（可移植操作系统接口），规定类 UNIX 系统必须遵守的接口规则。只要类 UNIX 系统符合 POSIX 标准，开发者写的软件（如 Ansible）不用修改就能在所有符合 POSIX 的系统上运行——这也是 Ansible 能用同一个模块管理 Ubuntu 和 RHEL 的核心原因。

**模块数量发展**
2015 年底 270 多个模块，2016 年 540 个，2018 年 1 月 1378 个，2018 年 7 月 1852 个，2019 年 5 月（2.7.10）2080 个，2020 年 3 月 3387 个。早期模块以基础系统运维功能为主，分类简洁。

## 九、模块分类

**早期基础分类**

| 分类 | 模块 |
|---|---|
| 文件与权限管理 | file、copy、unarchive |
| 用户与组管理 | user、group、authorized_key |
| 软件包管理 | yum、apt、pip |
| 系统服务管理 | service |
| 命令与脚本执行 | command、shell、script |
| 网络与安全管理 | iptables、nmcli |
| 磁盘与存储管理 | mount、parted |
| 配置与定时任务管理 | lineinfile、cron |

**当前 Ansible 模块分类（核心内置 + 第三方 Collections）**

| 分类 | 模块 |
|---|---|
| 文件与权限管理 | file、copy、template、unarchive、get_url、synchronize |
| 用户与身份管理 | user、group、authorized_key、seuser |
| 软件包管理 | yum/dnf、apt、package、pip、gem、npm、go |
| 系统服务与进程管理 | systemd、service、supervisorctl、ps、kill |
| 命令与脚本执行 | command、shell、script、raw |
| 网络与安全管理 | nmcli、route、dnsmasq、firewalld、ufw、selinux、auditd |
| 磁盘与存储管理 | mount、parted、lvg、lvol、filesystem |
| 配置文件管理 | lineinfile、replace、ini_file、xml、json_file |
| 数据存储管理 | mysql_db、postgresql_db、mongodb、redis、nfs_server、gluster_volume |
| 定时任务管理 | cron、at |
| 云资源管理（Collections） | amazon.aws.ec2_instance、azure.azcollection.azure_vm、aliyun.alicloud.aliyun_ecs、openstack.cloud.server、vmware.vmware_rest.vcenter_vm |
| 容器与虚拟化管理 | community.docker.docker_container、containers.podman.podman_container、kubernetes.core.k8s、libvirt.libvirt.vm、vmware.vmware_rest.vcenter_vm |
| 源码与版本控制 | git、svn、hg |
| 工具与辅助模块 | wait_for、debug、pause、assert、set_fact |
| 消息与通知 | slack、email、telegram、wechat_work |

## 十、幂等性

**定义**：多次执行同一个 Ansible 操作，最终结果和执行一次完全一致，不会产生冗余变更、不会重复创建资源、不会报错，就像「开关灯」——按一次开灯，再按多次还是开灯状态。

**两个关键概念**

**1. 声明式模块（原生支持幂等）**
这类模块的核心是「声明要达到什么状态，而非要执行什么动作」，模块会自动对比「当前实际状态」和「目标状态」，仅当二者不一致时才执行操作，这是幂等性的本质。
- **典型模块**：file、package、service、mount、firewalld 等
- **状态类参数**：用于声明目标状态的参数（如 `state`、`enabled`），是实现幂等的关键载体

**2. 命令式模块（无原生幂等）**
这类模块的核心是「单纯执行指定动作」，不会判断目标状态，仅机械执行命令，因此无原生幂等性。
- **典型模块**：command、shell、raw 等
- **无状态类参数**：仅接收要执行的命令/动作作为参数，不支持声明目标状态

> **注意**：并不是模块的参数都单独具有幂等性。幂等性是「状态参数声明目标状态」+「模块内置的状态对比逻辑」共同实现的，而非某个参数单独实现。

**以 file 模块创建目录为例：**
```yaml
- name: 确保目录存在
  file:
    path: /tmp/test
    state: directory   # 状态类参数：声明目标状态是目录存在
```
- `state: directory` 只是告诉模块「目标是让 `/tmp/test` 成为目录并存在」，它本身不具备幂等能力；
- **file 模块内置逻辑**会先检查 `/tmp/test` 的当前状态：目录不存在则创建（`changed: true`）；已存在则不操作（`ok: true`）；
- 幂等性由「state 参数声明目标状态」+「模块内置状态对比逻辑」共同实现。

**command 模块实现幂等的补充逻辑**
command 模块不原生支持幂等性，比如直接用 `command: mkdir /tmp/test`，第一次成功，第二次会报错「目录已存在」。可通过 `creates` 先检查状态再条件执行：
```bash
[root@rhce ~]# ansible 192.168.223.151 -m command -a 'mkdir /data'
192.168.223.151 | CHANGED | rc=0 >>
[root@rhce ~]# ansible 192.168.223.151 -m command -a 'mkdir /data'
192.168.223.151 | FAILED | rc=1 >>
mkdir: 无法创建目录 “/data”: 文件已存在

# creates 先判断 /data 是否存在，存在就不执行，不存在才执行
[root@rhce ~]# ansible 192.168.223.151 -m command -a 'mkdir /data creates=/data'
192.168.223.151 | SUCCESS | rc=0 >>
skipped, since /data exists
```

## 十一、command 模块详解

**参数格式与执行范围**：command 模块接收「命令名 + 空格分隔的参数列表」作为输入，指定命令会在所有选中的远程节点上执行。通过 `cmd` 参数或 `args` 关键字，无法同时执行多个指令。

**核心限制（无 Shell 解析）**：该模块执行的命令**不经过远程节点的 Shell 解释器**，因此以下内容均无法生效：环境变量（如 `$HOSTNAME`）；Shell 特殊操作符（`*` 通配符、`<`/`>` 重定向、`|` 管道、`;`/`&` 多命令拼接）。如需 Shell 特殊操作符，应替换为 `shell` 模块。
> 说明：能用对应模块解决的问题，尽量不用 command/shell，因为对应模块更安全，且具备幂等性。

**语法与示例：**
```yaml
# 'cmd' 是模块参数（推荐的规范写法）
- name: 仅当 /path/to/database 不存在时执行命令（使用 'cmd' 参数）
  ansible.builtin.command:
    cmd: /usr/bin/make_database.sh db_user db_name
    creates: /path/to/database

- name: 切换工作目录到 somedir/ 并以 db_owner 用户身份执行命令
  ansible.builtin.command: /usr/bin/make_database.sh db_user db_name
  become: yes            # 开启提权
  become_user: db_owner   # 切换到 db_owner 用户执行
  args:
    chdir: somedir/       # 执行命令前切换工作目录
    creates: /path/to/database

# 'argv' 以列表形式传递命令——将命令字段留空，argv 自动处理空格等特殊字符
- name: 使用 'argv' 以列表形式传递命令
  ansible.builtin.command:
    argv:
      - /usr/bin/make_database.sh
      - Username with whitespace
      - dbname with whitespace
    creates: /path/to/database
```
> `args` 是 Ansible 用于优化「命令执行类任务」的通用关键字，传递命令的附加执行选项（如切换目录、条件跳过），写法比复合 Shell 命令更规范；`argv` 是 command/shell 模块的专属参数，以列表形式传递命令及参数，避免特殊字符解析错误。

**实例：区分指令和参数**
- **指令**：`touch`、`ls` 等命令本身
- **参数**：command 模块的 `chdir`、`removes`、`creates` 等
```bash
# creates 先判断 /datafile 是否存在，存在不执行，不存在才执行
[root@rhce ~]# ansible 192.168.223.151 -m command -a 'touch /datafile creates=/datafile'
192.168.223.151 | CHANGED | rc=0 >>

# removes 先判断 /datafile 是否存在，存在才执行，不存在不执行
[root@rhce ~]# ansible 192.168.223.151 -m command -a 'touch /datafile removes=/datafile'
192.168.223.151 | CHANGED | rc=0 >>
[root@rhce ~]# ansible 192.168.223.151 -m command -a 'rm -rf /datafile removes=/datafile'
192.168.223.151 | CHANGED | rc=0 >>
```
> 之后再次执行 `ls -l /datafile` 会因文件已删除而 FAILED，验证 `removes` 条件生效。

## 十二、扩展：Playbook 调用模块

```bash
[root@rhce ~]# cat com.yaml
---
- name: 测试提权
  hosts: 192.168.223.151
  remote_user: redhat
  gather_facts: no
  tasks:
    - name: 查看shadow文件，以redhat身份执行
      command: cat /etc/shadow
      become: yes
      become_user: root
      args:
        creates: /data2

[root@rhce ~]# ansible-playbook -i /hosts com.yaml -K
BECOME password:
```

**整体格式说明（YAML 规范）：**
1. 所有缩进必须使用**半角空格**，禁止全角空格/制表符（Tab），默认 2 个空格为 1 个缩进层级
2. 键值对格式为「`key: value`」，冒号后必须跟 1 个半角空格
3. YAML 列表项以「`-`」开头，`-` 后必须跟 1 个半角空格
4. 顶层无缩进，每向下一个层级，增加 2 个半角空格

**扩展1：Ansible 会先解析模块参数**
要求：在 newserver 主机上，切换到 `/etc` 目录后读取 `redhat-release` 文件内容（指定 inventory 文件 `/hosts`，提权执行）。
```bash
ansible newserver -m command -a 'chdir=/etc cat redhat-release' -i /hosts
```
> 为什么 `cat redhat-release chdir=/etc` 也能读到 `/etc` 目录下的文件？因为 `chdir=/etc` 是 command 模块的参数而非命令的一部分，Ansible 会先解析该参数，在执行 `cat redhat-release` 前自动切换到 `/etc` 目录。

**扩展2：验证「不支持 Shell 特殊语法」（管道/&&/重定向）**
尝试用 command 模块执行 `cat /etc/passwd | grep root`（过滤 root 用户），观察结果并分析——由于 command 模块不经过 Shell，管道无法生效，应改用 shell 模块。

## 十三、小结
本文系统梳理了 Ansible 的**功能特性与架构原理**，从**环境准备、inventory 清单、免密登录、ansible.cfg 配置**，到**模块帮助手册、命令行选项与提权、指令执行流程、Collection 体系、模块分类、幂等性原理**，并深入讲解了 **command 模块** 与 **Playbook 调用模块**。掌握这些内容，就具备了编写和运行 Ansible 自动化任务、管理大规模主机集群的扎实基础。