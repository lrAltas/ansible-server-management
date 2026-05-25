---
# dnf_repo_sync

**CentOS 9 / Rocky 9 / RHEL 9 本地 DNF 镜像仓库自动化管理角色**

该角色用于在指定服务器上建立、维护并定时更新 DNF/YUM 本地镜像仓库，适用于内网环境、带宽优化及离线部署场景。

---

## 📝 快速配置说明书（必读）

**你只需要修改一个文件** 即可完成部署：`roles/dnf_repo_sync/vars/main.yml`

### 必须修改的变量（核心）

| 变量 | 说明 | 示例值 |
|---------|---------|----------|
| `dnf_repo_sync_sources` | **最重要**！你要同步的上游仓库列表（字典形式） | 见下方详细示例 |

**`dnf_repo_sync_sources` 配置示例** (请直接替换为你的地址)：

```yaml
dnf_repo_sync_sources:
  centos-stream-9-crb:
    upstream_url: "https://mirrors.huaweicloud.com/centos-stream/9-stream/CRB/x86_64/os/"
    local_path: "{{ dnf_repo_sync_base_path }}/centos-stream/9/CRB/x86_64/os"
```

### 强烈建议修改的变量

| 变量 | 说明 | 建议值 |
|---------|---------|----------|
| `dnf_repo_sync_auto_update` | 是否启用每天自动同步 | `true` (生产环境强烈建议) |
| `dnf_repo_sync_cron_hour` | 定时任务执行时间（小时） | `"3"` (凌晨 3 点，避开高峰) |
| `dnf_repo_sync_base_path` | 本地镜像根目录 | `/usr/share/nginx/html/repos` |

### 可选修改的变量（高级）

| 变量 | 说明 | 默认值 |
|---------|---------|----------|
| `dnf_repo_sync_disable_system_repos` | 是否暂时禁用系统 yum 源（强烈建议开启） | `true` |
| `dnf_repo_sync_clean` | 是否清空后重建 | `false` |

---

## ⚠️ 执行顺序说明与建议

**2026-05-25 更新**

为了确保软件包安装顺利，本角色采用了以下执行顺序：

1. 检查配置
2. **安装软件包** (nginx, dnf-utils, createrepo 等)
3. 创建目录
4. **暂时禁用系统 yum 源**
5. 执行同步
6. 配署定时任务、Web 服务等
7. **恢复系统 yum 源** + 完成提示

**强烈建议**：

在运行剧本前，**手动在镜像机器上预先安装好必要的软件包**：

```bash
sudo dnf install -y nginx dnf-utils createrepo firewalld
```

这样可以避免任何源问题，让剧本运行更稳定。

---

## ⚠️ 重要：暂时禁用系统 yum 源（新增功能）

**2026-05-25 新增功能**

如果你的镜像机器上已经安装了其他 yum 源（例如 `dnf repolist` 能看到 baseos/appstream/crb 等），`reposync` 很可能会使用这些系统源，导致同步错误的内容。

本角色默认会自动执行以下操作（可以关闭）：

1. **备份** `/etc/yum.repos.d/` 下所有 `.repo` 文件
2. **暂时移除** 这些文件（让同步环境完全干净）
3. **同步完成后自动恢复** 原来的 repo 文件

**输出会显示清晰提示**：

```
⚠️ 正在暂时禁用系统所有 yum 源...
已备份 X 个 repo 文件到 /etc/yum.repos.d/backup-...
已恢复系统 repo 文件
```

如果你想关闭此功能，在 `vars/main.yml` 中设置：

```yaml
dnf_repo_sync_disable_system_repos: false
```

---

## 功能特性

- 支持任意上游路径镜像（直接给完整 URL 即可）
- 可自定义本地存放位置，目录结构清晰、易于管理
- 支持从零开始重建和增量同步
- 内置定时任务（cron），每天自动更新
- 可选部署 Nginx/HTTPD 提供 HTTP 访问
- 完整的防火墙与 SELinux 配置
- 高度可配置，所有参数均可在 `vars/main.yml` 中一键修改

## 目录结构建议（强烈建议使用）

```
/var/www/html/repos/
├── rocky/
│   ├── 9/
│       ├── BaseOS/
│       │   └── x86_64/
│       │       └── os/
│       └── AppStream/
│           └── x86_64/
│               └── os/
├── centos/
│   └── 9-stream/
│       └── BaseOS/
│           └── x86_64/
│               └── os/
└── epel/
    └── 9/
        └── Everything/
            └── x86_64/
```

这种结构方便按发行版、架构、组件进行分类管理。

## 快速开始

### 1. 配置主机

在 `inventory/hosts` 中添加你的镜像服务器：

```ini
[dnf_repo_servers]
centos9-mirror ansible_host=192.168.1.50 ansible_user=root
```

### 2. 修改配置文件

打开 `roles/dnf_repo_sync/vars/main.yml`，主要修改以下内容：

1. `dnf_repo_sync_sources` — 填入你的上游地址
2. `dnf_repo_sync_auto_update: true` — 开启自动更新
3. `dnf_repo_sync_cron_hour` — 调整执行时间

### 3. 执行同步

```bash
ansible-playbook playbooks/sync_repos.yml -l dnf_repo_servers
```

### 4. 检查定时任务

```bash
crontab -l | grep update-dnf-repos
```

## 变量详解

主要配置均在 `vars/main.yml` 中，并附带详细注释。

## 定时自动更新机制

启用后，角色会在目标机器上生成以下文件：

- `/usr/local/bin/update-dnf-repos.sh` — 自动生成的同步脚本
- `/var/log/dnf-repo-sync.log` — 同步日志
- cron 任务（默认每天凌晨 3 点）

## 客户端配置示例

在其他服务器上创建 `/etc/yum.repos.d/rocky-local.repo`：

```ini
[rocky-9-baseos-local]
name=Rocky 9 BaseOS Local
baseurl=http://your-mirror-ip/repos/rocky/9/BaseOS/x86_64/os/
enabled=1
gpgcheck=0

[rocky-9-appstream-local]
name=Rocky 9 AppStream Local
baseurl=http://your-mirror-ip/repos/rocky/9/AppStream/x86_64/os/
enabled=1
gpgcheck=0
```

## 高级用法

### 从零重建整个镜像

```bash
ansible-playbook playbooks/sync_repos.yml -l dnf_repo_servers -e "dnf_repo_sync_clean=true"
```

### 删除本地镜像

```bash
ansible-playbook playbooks/sync_repos.yml -l dnf_repo_servers -e "dnf_repo_sync_state=absent"
```

## 注意事项

- 首次运行可能需要几十分钟到几小时，请耐心等待
- 建议生产环境开启 `auto_update`
- 日志文件位于 `/var/log/dnf-repo-sync.log`

## 授权许可

MIT License
