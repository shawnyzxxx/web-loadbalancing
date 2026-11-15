# RHCSE--ansible2--剧本

## 文章信息

- 原文发布于：2026-01-16 22:19:30
- 标签：服务器、运维、linux、ansible
- 原阅读量：833

## 文章导读

本文是 RHCSE 体系下 Ansible 部分的进阶内容，围绕 **Ansible Playbook（剧本）** 展开。Playbook 是 Ansible 的核心配置文件，用 YAML 编写，用来把一系列有序的自动化任务组织起来，描述“要在哪些远程主机上执行哪些操作”。本文从 YAML 基本规则讲起，逐步覆盖 Playbook 的常用结构、`ansible-playbook` 的常用选项、执行流程与并发控制、语法检查，再到「无 Role」与「含 Role」两种 Playbook 的组织方式，最后给出两个完整实战案例：远程批量部署 Nginx 并修改端口、纯剧本方式部署 LNMP + Discuz。适合已经掌握 Ansible 入门与模块使用、希望用 Playbook 落地批量运维的 Linux 运维与 RHCSE 备考读者。

## 一、Ansible Playbook 概述

Ansible Playbook（剧本）是 Ansible 核心配置文件，采用 YAML 格式编写，用于定义一系列有序的自动化任务集合，描述“要在哪些远程主机上执行哪些操作”。它把多个模块、变量、处理器按顺序编排，一次执行即可对多台受控节点完成批量配置，是 Ansible 从单条命令走向工程化自动化的关键。

## 二、YAML 基本规则

YAML 是一种易读的序列化格式，是 Playbook 的核心语法，核心规则如下：

- **缩进规则**：仅允许空格缩进（禁止 Tab），通常 2 个半角空格为 1 级，缩进层级代表逻辑层级。
- **全角空格陷阱**：全角空格是中文输入法下的空格（肉眼看和半角空格一样，但解析器不认），必须替换为半角空格（英文输入法下的空格）。
- **大小写敏感**：变量名、模块名、键名（如 `hosts` / `Hosts`）均区分大小写。
- **注释规则**：单行注释用 `#`（`#` 后内容会被忽略），YAML 没有原生多行注释，需要时每行单独加 `#`。
- **字符串规则**：字符串可加引号（单引号 / 双引号）或不加；含特殊字符（`:`、`$`、空格）时必须加引号，例如 `name: "web-server:8080"`。
- **核心标识**：
  - 列表元素以 `-` 开头（短横线 + 空格），如 `- nginx`；
  - 键值对使用 `key: value`（冒号后必须加空格），如 `name: nginx`。
- **文件命名**：Playbook 文件后缀没有强制要求，只要内容是合法 YAML 即可执行；一般用 `.yaml` 或 `.yml` 结尾，前者更规范，后者是简写，两者都常用。

## 三、Playbook 常用 YAML 结构

Playbook 以 **Play** 为基本单元，一个 Play 就是一个部署单元，常见结构如下：

```text
Playbook（YAML文件）
├─ Play 1（一个部署单元，以 - 开头）
│  ├─ 基础配置（hosts、remote_user、gather_facts）
│  ├─ 变量（vars）
│  ├─ 处理器（handlers）
│  └─ 任务（tasks）→ 每个任务由「模块+参数」组成
│      ├─ 模块+参数
│      └─ 模块+参数
├─ Play 2（另一个部署单元）
│  └─ ...（同 Play 1 的结构）
└─ ...
```

### 3.1 单 Play 结构（极简版）

```yaml
- hosts: webservers  # 目标主机组
  remote_user: root  # 远程执行用户
  tasks:  # 任务列表
    - name: 安装 Nginx
      yum:
        name: nginx
        state: present
    - name: 启动 Nginx
      service:
        name: nginx
        state: started
```

### 3.2 多 Play 结构（多主机组执行不同任务）

```yaml
- hosts: webservers
  vars:
    web_name: nginx
  tasks:
    - name: 安装 Nginx
      yum:
        name: "{{ web_name }}"
        state: present
- hosts: dbservers
  tasks:
    - name: 安装 MySQL
      yum:
        name: mysql-server
        state: present
```

### 3.3 带变量 / 处理器的嵌套结构

下面以「部署 MySQL 服务」为例，编写带变量与处理器的嵌套结构 Playbook，保持核心逻辑一致，同时补充关键注释便于理解。

```yaml
# 针对所有被管理主机部署 MySQL 服务
- hosts: all
  # 1. 定义 Play 级变量（可复用，修改时仅需改此处）
  vars:
    mysql_pkg: mariadb-server  # CentOS 默认 MySQL 兼容包
    mysql_port: 3306
    mysql_conf_path: /etc/my.cnf.d/mysql-server.cnf
    mysql_service: mariadb

  # 2. 定义处理器（仅被 notify 触发，用于配置变更后重启服务）
  handlers:
    - name: 重启MySQL服务并设置开机自启
      ansible.builtin.service:
        name: "{{ mysql_service }}"  # 引用变量
        state: restarted            # 重启服务
        enabled: true               # 开机自启

  # 3. 定义核心任务列表
  tasks:
    # 任务1：安装 MySQL 软件包（引用变量）
    - name: 安装 {{ mysql_pkg }} 软件包
      ansible.builtin.yum:
        name: "{{ mysql_pkg }}"
        state: present  # 确保安装（幂等：已装则不操作）

    # 任务2：启动 MySQL 服务（首次部署先启动）
    - name: 启动 {{ mysql_service }} 服务
      ansible.builtin.service:
        name: "{{ mysql_service }}"
        state: started

    # 任务3：复制自定义 MySQL 配置文件（修改配置后触发处理器）
    - name: 复制自定义MySQL配置文件（指定端口{{ mysql_port }}）
      ansible.builtin.copy:
        src: ./mysql-server.cnf  # 本地配置文件路径
        dest: "{{ mysql_conf_path }}"  # 远程目标路径（引用变量）
        mode: '0644'  # 配置文件权限
        owner: root   # 属主
        group: root   # 属组
      # 仅当配置文件发生变更时，触发处理器重启 MySQL
      notify: 重启MySQL服务并设置开机自启
```

## 四、ansible-playbook 常用选项

`ansible-playbook` 是执行 Playbook 的命令，常用选项如下：

| 选项 | 简写 | 核心说明 |
| --- | --- | --- |
| `--check` | `-C` | 模拟执行（干跑）：仅展示要执行的操作，不实际修改目标主机系统 |
| `--inventory <路径>` | `-i` | 指定自定义 Inventory（主机清单）文件，默认路径 `/etc/ansible/hosts` |
| `--user <用户名>` | `-u` | 指定远程执行任务的用户（如 `root`、`ansible`） |
| `--become` | `-b` | 提权执行（等效 sudo），普通用户执行系统操作（如装软件）时必备 |
| `--become-user <用户>` | 无 | 指定提权后的目标用户（默认 `root`），如 `--become-user=nginx` |
| `--verbose` | `-v` | 详细输出执行日志：`-v` 基础详情，`-vv` 中等，`-vvv` 最详细（排错专用） |
| `--ask-pass` | `-k` | 交互式提示输入远程主机 SSH 密码（避免明文写在 Inventory 中） |
| `--ask-become-pass` | `-K` | 交互式提示输入提权密码（如 sudo 密码） |
| `--limit <主机/组>` | 无 | 限制仅执行指定主机 / 主机组的任务，如 `--limit 192.168.1.100` |

## 五、Playbook 执行流程与并发控制

Playbook 的执行流程可以概括为：按 Play 内任务顺序串行推进，而每个任务在目标主机组内并发执行。

```text
任务1: [host1, host2, host3, ... hostN] ← 同时执行
等待任务1所有主机成功，然后开始执行任务2
任务2: [host1, host2, host3, ... hostN] ← 同时执行
```

> **结论**：如果主机数量过多、并发太大，等待时间会明显变长，因此用 `--limit <主机/组>` 限制执行范围，减少并发数、快速验证。

```bash
# 仅在 host1 和 host2 上执行 Playbook（减少并发数，快速验证）
ansible-playbook task.yml -i /hosts --limit "host1,host2"
```

如果 Inventory 中已有子组，可以直接限制组。先在 Inventory 中添加子组：

```ini
[web_servers_subset]
host1
host3
```

然后执行：

```bash
ansible-playbook task.yml -i /hosts --limit web_servers_subset
```

## 六、Playbook 语法检查

作用：检查 YAML 语法与 Playbook 结构（如字段缺失、缩进错误），不执行任务，输出报错位置和原因。

```bash
ansible-playbook -C -v playbook.yml
```

## 七、无 Role 的 Playbook

不引入 Role 时，可以直接把变量、处理器、任务都写在同一个 Playbook 中。示例文件为 `site-simple.yml`：

```yaml
# site-simple.yml
---
- hosts: webservers
  vars:
    pkg_nginx: nginx
    nginx_port: 80
  handlers:
    - name: 重启Nginx
      service:
        name: nginx
        state: restarted
  tasks:
    - name: 安装{{ pkg_nginx }}
      yum:
        name: "{{ pkg_nginx }}"
        state: present
    - name: 复制配置文件
      copy:
        src: ./nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: 重启Nginx
    - name: 启动Nginx
      service:
        name: nginx
        state: started
```

## 八、含 Role 的 Playbook

Role 是 Ansible 组织复杂 Playbook 的方式，把变量、任务等按角色拆分到目录中，结构更清晰。目录结构示例：

```text
/ansible/
├── roles/
│   ├── web/  # web 角色目录（变量仅属于 web 角色）
│   │   ├── vars/
│   │   │   └── main.yml  # web 角色的变量文件（Ansible 自动识别）
│   │   └── tasks/
│   │       └── main.yml  # web 角色的任务文件
│   └── security/  # security 角色目录（变量仅属于 security 角色）
│       ├── vars/
│       │   └── main.yml  # security 角色的变量文件（Ansible 自动识别）
│       └── tasks/
│           └── main.yml  # security 角色的任务文件
└── site.yml  # 主 Playbook
```

创建对应目录：

```bash
mkdir -p /ansible/roles/web/vars
mkdir -p /ansible/roles/web/tasks
mkdir -p /ansible/roles/security/vars
mkdir -p /ansible/roles/security/tasks
mkdir -p /ansible/roles/security/{vars,tasks}
```

### 8.1 清单文件

仅定义 `webservers` 主机组（学生替换为自己的受控节点 IP）：

```ini
[newserver]
192.168.223.151
192.168.223.152
```

### 8.2 web 角色 - 变量：roles/web/vars/main.yml

web 角色仅保留核心变量，减少记忆成本：

```yaml
# web角色仅保留核心变量（减少记忆成本）
nginx_pkg: nginx    # Nginx软件包名
nginx_service: nginx # Nginx服务名
```

### 8.3 web 角色 - 任务：roles/web/tasks/main.yml

web 角色核心任务为安装 + 启动 Nginx，无复杂配置，聚焦角色逻辑：

```yaml
# web角色核心任务：安装+启动Nginx（无复杂配置，聚焦角色逻辑）
---
- name: 安装Nginx软件包
  ansible.builtin.yum:
    name: "{{ nginx_pkg }}"
    state: present  # 确保安装（幂等，多次执行不重复安装）

- name: 启动并开机自启Nginx服务
  ansible.builtin.service:
    name: "{{ nginx_service }}"
    state: started
    enabled: true
```

### 8.4 security 角色 - 任务：roles/security/tasks/main.yml

security 角色核心任务为防火墙放行 80 端口 + SELinux 宽松模式：

```yaml
# security角色核心任务：防火墙放行80端口 + SELinux宽松模式
- name: 放行防火墙80端口（http服务）
  ansible.builtin.firewalld:
    service: http  # 直接用预设的http服务（对应80端口，比写port更简单）
    permanent: true # 永久生效
    immediate: true # 立即生效（无需重启防火墙）
    state: enabled

- name: 设置SELinux为宽松模式（临时生效，避免重启）
  ansible.builtin.selinux:
    policy: targeted
    state: permissive  # 宽松模式：不阻止操作，仅记录日志
```

### 8.5 主 Playbook：site.yml

主 Playbook 只做两件事：指定目标主机 + 引入两个角色：

```yaml
# 主Playbook仅做2件事：指定目标主机 + 引入两个角色
---
- hosts: webservers  # 关联清单中的webservers主机组
  remote_user: root  # 远程执行用户
  gather_facts: true # 自动收集Facts（firewalld/selinux模块依赖）

  # 按顺序引入角色：先做安全配置，再部署Web（逻辑合理）
  # 提前写好仓库挂载好，或者增加一个cnagku角色
  roles:
    - security  # 引入security角色（执行防火墙+SELinux配置）
    - web       # 引入web角色（执行Nginx部署）
```

### 8.6 执行与验证

```bash
ansible-playbook -i /hosts /ansible/site.yml
```

## 九、Role 变量文件的识别机制

扩展问题：把某个角色需要的变量单独放到文件里，`ansible-playbook` 如何识别哪个变量文件属于哪个角色？

> **结论**：当在 Playbook 的 `roles` 中直接以“字符串形式”引用角色（如 `- web`）时，Ansible 会默认将该角色名称作为目录名，去 `roles/` 目录下查找同名目录（如 `roles/web/`），这是最规范、最常用的方式，无需额外配置。

## 十、案例：远程批量安装 Nginx、修改端口并生效

该案例演示用纯 Playbook 批量部署 Nginx，修改监听端口，并让配置生效。

需要注意两个要点：

- **每个 Task 只能调用 1 个模块**。
- `description` 参数用于描述 yum 源的用途（比如“CentOS BaseOS Repository”），是 `yum_repository` 模块的强制参数；没有它 Ansible 会判定参数不完整，直接终止任务并抛出 `Parameter 'description' is required` 错误。
- `yum_repository` 指定的 `file` 名称（比如 `x`）和 `/etc/yum.repos.d/` 目录下已存在的 `.repo` 文件（如 `x.repo`）重名时，Ansible 的处理逻辑非常友好——不会直接覆盖整个文件，而是“精准修改 / 追加”，不用担心原有配置丢失。

```yaml
---
- name: 给web服务器部署nginx  # Play的名字（方便看日志）
  hosts: webservers  # 这个Play要操作的主机（对应你的Hosts列表）
  tasks:  # 这个Play里的Task列表
    - name: 挂载光盘
      mount:
        path: /mnt
        src: /dev/sr0
        state: mounted
        fstype: iso9660   # 光盘专属文件系统类型（必须加）
    - name: 配置仓库base
      yum_repository:
        file: x
        name: base
        description: base 
        state: present
        enabled: yes
        gpgcheck: no
        baseurl: /mnt/BaseOS
    - name: 配置仓库app
      yum_repository:
        file: x
        name: app
        description: app 
        state: present
        enabled: yes
        gpgcheck: no
        baseurl: /mnt/AppStream
    - name: 安装nginx
      yum:
        name: nginx
        state: present
    - name: 修改nginx配置文件
      lineinfile:
        path: /etc/nginx/nginx.conf  # 主流端口配置文件路径
        regexp: "^\\s*listen\\s+\\d+;?$"  # 匹配以listen开头、后跟数字的行（如listen 80;）
        line: "    listen 8080;"              # 替换为监听8080端口（缩进和原文件保持一致）
        backup: yes                           # 修改前自动备份原文件（建议开启，方便回滚）
        state: present
    - name: 修改Nginx默认欢迎页面为welcome test
      copy:
        content: "welcome test"  # 页面要显示的内容
        dest: /usr/share/nginx/html/index.html  # Nginx默认首页路径
        backup: yes  # 备份原首页文件（生成index.html.bak.xxxxxx）
        mode: '0644'  # 设置文件权限，确保Nginx能读取（必需）
    - name: 启动nginx
      service:
        name: nginx
        state: started
```

## 十一、实验2：纯剧本方式部署 LNMP + Discuz

下面给出一个扁平任务结构的 Playbook，用纯剧本方式部署 LNMP + Discuz 论坛，并通过 `find` 动态处理文件名不固定的压缩包。

```bash
[root@rhce ~]# cat discuz.yml
```

```yaml
---
- name: 纯剧本部署 LNMP + Discuz 论坛（适配文件名不固定）
  hosts: 192.168.223.151
  gather_facts: no
  become: yes

  vars:
    # 基础配置
    mount_src: /dev/sr0
    mount_path: /mnt
    mount_fstype: iso9660
    web_root: /var/www/html/discuz
    nginx_conf_path: /etc/nginx/conf.d/discuz.conf
    discuz_url: "https://gitee.com/Discuz/DiscuzX/attach_files/2335009/download"
    # 数据库配置
    db_root_pwd: "Redhat123!"  # 学生可自行修改目标密码
    db_user: "root"
    db_host: "localhost"

  tasks:
    ###########################################################################
    # 步骤 0：挂载光盘（原生幂等）
    ###########################################################################
    - name: 挂载光盘到/mnt目录
      mount:
        src: "{{ mount_src }}"
        path: "{{ mount_path }}"
        fstype: "{{ mount_fstype }}"
        state: mounted

    ###########################################################################
    # 步骤 1：安装基础软件（原生幂等）
    ###########################################################################

    - name: 安装LNMP及基础依赖包
      package:
        name:
          - unzip
          - nginx
          - php
          - php-fpm
          - php-mysqlnd
          - mariadb-server
          - python3-PyMySQL
        state: present

    ###########################################################################
    # 步骤 2：启动基础服务（原生幂等）
    ###########################################################################
    - name: 启动并开机自启Nginx服务
      systemd:
        name: nginx
        state: started
        enabled: yes

    - name: 启动并开机自启PHP-FPM服务
      systemd:
        name: php-fpm
        state: started
        enabled: yes

    - name: 启动并开机自启MariaDB服务
      systemd:
        name: mariadb
        state: started
        enabled: yes

    ###########################################################################
    # 步骤 3：创建网站根目录（原生幂等）
    ###########################################################################
    - name: 创建Discuz网站根目录并设置权限
      file:
        path: "{{ web_root }}"
        state: directory
        owner: nginx
        group: nginx
        mode: 0755

    ###########################################################################
    # 步骤 4：下载+解压Discuz（核心适配：文件名不固定，全靠find动态处理）
    ###########################################################################
    # 第一步：检查是否已有任意Discuz开头的压缩包（不限制具体名称）
    - name: 检查{{ web_root }}是否有Discuz开头的压缩包
      find:
        path: "{{ web_root }}"
        patterns: "Discuz*.zip"  # 匹配所有Discuz开头的zip包
        file_type: file
      register: existing_discuz_zip

    # 第二步：仅当无任何Discuz压缩包时，才下载（避免重复下载/401）
    - name: 下载Discuz压缩包（文件名不固定，下载到目录即可）
      get_url:
        url: "{{ discuz_url }}"
        dest: "{{ web_root }}/"  # 指向目录，让文件按URL默认名保存
        validate_certs: no
        timeout: 60
      when: existing_discuz_zip.files | length == 0  # 无任何Discuz包时才下载

    # 第三步：再次find所有Discuz压缩包（适配下载后的随机名称）
    - name: 动态查找{{ web_root }}下所有Discuz开头的压缩包
      find:
        path: "{{ web_root }}"
        patterns: "Discuz*.zip"
        file_type: file
      register: discuz_zip_files

    # 第四步：检查是否已解压（通过Discuz核心目录判断，与文件名无关）
    - name: 检查Discuz是否已解压（通过upload目录判断）
      stat:
        path: "{{ web_root }}/upload"
      register: discuz_unzip_stat

    # 第五步：解压（仅找到压缩包且未解压时执行，适配任意Discuz开头的包）
    - name: 解压动态匹配到的Discuz压缩包
      unarchive:
        src: "{{ discuz_zip_files.files[0].path | default('') }}"  # 取第一个匹配的包，兜底赋空
        dest: "{{ web_root }}"
        remote_src: yes
      when:
        - discuz_zip_files.files | length > 0  # 至少找到1个Discuz包
        - not discuz_unzip_stat.stat.exists    # 未解压过
      ignore_errors: yes  # 兜底：压缩包损坏/格式问题不中断流程
#当 {{ web_root }}/upload 目录不存在时：discuz_unzip_stat.stat.exists 为 False，经 not 取反后为 True，条件满足，解压任务会执行；
#当 {{ web_root }}/upload 目录已存在时：discuz_unzip_stat.stat.exists 为 True，经 not 取反后为 False，条件不满足，解压任务会跳过。

    ###########################################################################
    # 步骤 5：配置Nginx虚拟主机（原生幂等）
    ###########################################################################
    - name: 写入Discuz的Nginx配置文件
      copy:
        content: |
          server {
              listen 80;
              root {{ web_root }}/upload;
              include /etc/nginx/default.d/php.conf;
          }
        dest: "{{ nginx_conf_path }}"
        backup: yes
      register: nginx_conf
      changed_when: nginx_conf.changed

    - name: 重新加载Nginx配置（仅配置变更时执行）
      systemd:
        name: nginx
        state: reloaded
      when: nginx_conf.changed

    ###########################################################################
    # 步骤 6：配置防火墙和SELinux（原生幂等）
    ###########################################################################
    - name: 永久开放80端口
      firewalld:
        port: 80/tcp
        permanent: yes
        state: enabled
        immediate: yes

    - name: 临时关闭SELinux
      command: setenforce 0
      ignore_errors: yes
      changed_when: no

    ###########################################################################
    # 步骤 7：修改Discuz权限（幂等：仅目录存在时执行）
    ###########################################################################
    - name: 检查Discuz upload目录是否存在
      stat:
        path: "{{ web_root }}/upload"
      register: discuz_upload_stat

    - name: 修改Discuz data目录权限为777
      file:
        path: "{{ web_root }}/upload/data"
        mode: 0777
        recurse: yes
        state: directory
      when: discuz_upload_stat.stat.exists

    - name: 修改Discuz config目录权限为777
      file:
        path: "{{ web_root }}/upload/config"
        mode: 0777
        recurse: yes
        state: directory
      when: discuz_upload_stat.stat.exists

    - name: 批量修改uc_client/uc_server目录权限为777
      file:
        path: "{{ web_root }}/upload/{{ item }}"
        mode: 0777
        recurse: yes
        state: directory
      loop:
        - uc_client
        - uc_server
      when: discuz_upload_stat.stat.exists

    ###########################################################################
    # 步骤 8：初始化数据库（原生幂等）
    ###########################################################################
    # 步骤1：判断root是否有密码（核心：用简单命令测试）
    - name: 测试root无密码能否登录
      ansible.builtin.shell: 'mysql -u{{ db_user }} -S /var/lib/mysql/mysql.sock -e "show databases;" 2>/dev/null'
      register: no_pwd_login
      ignore_errors: yes  # 有密码时登录失败，忽略错误不中断
      changed_when: no    # 仅判断状态，不标记“变更”

    # 步骤2：仅无密码时，设置root密码（幂等：设过就不重复设）
    - name: 初始化root密码（仅第一次执行）
      community.mysql.mysql_user:
        name: "{{ db_user }}"
        host: "localhost"  # 简化：只设localhost的密码，去掉冗余host
        password: "{{ db_root_pwd }}"
          #  check_implicit_admin: yes  # 无密码时允许登录的关键参数
        login_unix_socket: /var/lib/mysql/mysql.sock
        login_user: "{{ db_user }}"
        login_host: "{{ db_host }}"
        login_password: ""  # 无密码时填空
        state: present      # 幂等核心：密码已设则跳过
      when: no_pwd_login.rc == 0  # 只有无密码登录成功时才执行

    # 步骤3：创建测试库（幂等：已存在则跳过，适配有/无密码登录）
    - name: 创建luntan数据库（多次执行不报错）
      community.mysql.mysql_db:
        name: luntan
        state: present  # 幂等：确保库存在，不存在才创建
        encoding: utf8mb4
        # 动态适配密码：有密码用db_root_pwd，无密码填空
        login_unix_socket: /var/lib/mysql/mysql.sock
        login_user: "{{ db_user }}"
        login_host: "{{ db_host }}"
        login_password: "{{ db_root_pwd }}"
```

### 11.1 执行剧本与验证

1. 执行命令：`ansible-playbook discuz-lnmp-playbook.yml`
2. 验证：浏览器访问 `http://受控节点IP`，进入 Discuz 安装向导，输入剧本中定义的数据库信息（库名 `discuz_db`、用户 `discuz_user`、密码 `Discuz@123`），完成安装。

## 十二、常见问题：MariaDB root 密码任务执行失败

执行修改 MariaDB root 密码的任务时可能失败，问题与解决如下。

### 12.1 错误原因

Ansible 的 `mysql_user` / `mysql_db` 模块依赖 Python 的 MySQL 数据库驱动包，但受控节点上没有安装对应的包，或者 Python 解释器版本不匹配，导致找不到驱动。

### 12.2 解决方案

1. **解释错误核心**：
   - 错误提示明确说明需要 MySQL Python 模块：Python 2.7 需要 `PyMySQL` / `MySQL-python`，Python 3 需要 `mysqlclient` / `PyMySQL`；
   - 受控节点要么没装这些包，要么 Ansible 使用的 Python 解释器与已安装的驱动版本不匹配。
2. **分步骤解决**：
   - **步骤 1**：在受控节点安装 Python MySQL 驱动（区分 CentOS 版本，使用 `yum` / `dnf`）；
   - **步骤 2**：（可选）指定 Ansible 的 Python 解释器，避免版本不匹配；
   - **步骤 3**：重新执行任务。