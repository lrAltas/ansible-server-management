# dnf_repo_sync

**CentOS 9 / Rocky 9 / RHEL 9 本地 DNF 镜像仓库自动化管理角色**

该角色用于在指定服务器上建立、维护并定时更新 DNF/YUM 本地镜像仓库，适用于内网环境、带宽优化及离线部署场景。

## 功能特性

- 支持任意上游路径镜像（直接给完整 URL 即可）
- 可自定义本地存放位置，目录结构清晰、易于管理
- 支持从零开始重建和增量同步
- 内置定时任务（cron），每天自动更新
- 可选部署 Nginx/HTTPD 提供 HTTP 访问
- 完整的防火墙与 SELinux 配置
- 高度可配置，所有参数均可在 vars/main.yml 中一键修改

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

### 2. 配置要同步的源

编辑 `roles/dnf_repo_sync/vars/main.yml` 中的 `dnf_repo_sync_sources` 字典：

```yaml
dnf_repo_sync_sources:
  rocky-9-baseos:
    upstream_url: "https://download.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/"
    local_path: "{{ dnf_repo_sync_base_path }}/rocky/9/BaseOS/x86_64/os"   # 可选

  rocky-9-appstream:
    upstream_url: "https://download.rockylinux.org/pub/rocky/9/AppStream/x86_64/os/"
```

### 3. 执行同步

```bash
ansible-playbook playbooks/sync_repos.yml -l dnf_repo_servers
```

### 4. 启用定时自动更新（强烈建议）

在 `vars/main.yml` 中设置：

```yaml
dnf_repo_sync_auto_update: true
dnf_repo_sync_cron_hour: "3"      # 每天凌晨 3 点
```

然后再次执行上面的 playbook 即可。

## 变量详解

主要配置均在 `vars/main.yml` 中，并附带详细注释。

| 变量 | 说明 | 默认值 |
|---------|---------|----------|
| `dnf_repo_sync_base_path` | 本地镜像根目录 | `/var/www/html/repos` |
| `dnf_repo_sync_sources` | 要同步的源列表（字典） | `{}` |
| `dnf_repo_sync_auto_update` | 是否启用定时任务 | `false` |
| `dnf_repo_sync_cron_hour` | 定时任务执行时间（小时） | `"3"` |
| `dnf_repo_sync_clean` | 是否清空后重建 | `false` |
| `dnf_repo_sync_reposync_options` | reposync 参数 | `--download-metadata --downloadcomps --delete --newest-only` |

## 定时自动更新机制

启用后，角色会在目标机器上生成以下文件：

- `/usr/local/bin/update-dnf-repos.sh` — 自动生成的同步脚本
- `/var/log/dnf-repo-sync.log` — 同步日志
- cron 任务（默认每天凌晨 3 点）

可以通过 `crontab -l` 查看定时任务。

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
- 如果上游支持 rsync 协议，可以手动替换为 rsync 命令以提高速度
- 日志文件位于 `/var/log/dnf-repo-sync.log`

## 授权许可

MIT License
