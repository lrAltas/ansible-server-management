# dnf_repo_sync

**专为 CentOS 9 / Rocky 9 / RHEL 9 设计的本地 DNF 源镜像角色**（从0开始创建 + 增量升级 + 清除重建）。

## 核心特性（完全按你的需求实现）

- ✅ **字典形式配置源**：在 `defaults/main.yml` 或 `group_vars` 里用 dict 定义要同步的路径，你直接给 upstream_url，角色就同步该路径下的所有包 + metadata。
- ✅ **从0开始创建本地源** + **支持清除重建**（`dnf_repo_sync_clean: true` 或 `state: absent`）。
- ✅ **幂等性**：反复执行安全，已存在的仓库会增量同步新包。
- ✅ **全变量驱动**：本地路径、web server、防火墙、包列表全部可配置。
- ✅ **直接从 inventory 取主机**：使用你填写的 `ansible_host`。
- 使用 `reposync --repofrompath` 实现“给什么路径就同步什么路径下的内容”。

## 变量配置（全部在 defaults/main.yml，你直接编辑）

```yaml
# 1. 本地仓库根目录
dnf_repo_sync_base_path: /var/www/html/repos

# 2. 同步状态与清理
dnf_repo_sync_state: present          # present=创建/同步, absent=删除本地镜像
dnf_repo_sync_clean: false            # true 时先清空再重建（慎用）

# 3. 要同步的源（字典形式 - 你改这里）
dnf_repo_sync_sources:
  rocky9-baseos:
    upstream_url: "https://download.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/"
  rocky9-appstream:
    upstream_url: "https://download.rockylinux.org/pub/rocky/9/AppStream/x86_64/os/"
  # 添加任意你想要的路径
```

## 使用方法

1. 编辑 inventory/hosts，填入你的 CentOS 9 源服务器：
   ```ini
   [dnf_repo_servers]
   centos9-mirror ansible_host=10.0.0.50 ansible_user=root
   ```

2. 编辑 `roles/dnf_repo_sync/defaults/main.yml`（或新建 group_vars/all.yml），修改：
   - `dnf_repo_sync_sources` 里的 upstream_url（直接粘贴你想同步的完整路径）
   - `dnf_repo_sync_base_path` 等其他变量

3. 执行同步（首次或增量升级）：
   ```bash
   ansible-playbook playbooks/sync_repos.yml -l dnf_repo_servers
   ```

4. **全新重建**（清空后从0开始）：
   ```bash
   ansible-playbook playbooks/sync_repos.yml -l dnf_repo_servers -e "dnf_repo_sync_clean=true"
   ```

5. **删除本地镜像**：
   ```bash
   ansible-playbook playbooks/sync_repos.yml -l dnf_repo_servers -e "dnf_repo_sync_state=absent"
   ```

## 注意事项

- 第一次运行会下载大量包，根据你的网络和上游速度而定。
- 建议配合 `--newest-only`（已在默认 options 里）节省空间。
- Web 服务默认使用 nginx，可通过变量切换成 httpd。
- 如需更复杂的 nginx 配置（自定义 server_name、location 等），告诉我，我继续完善。
- 客户端可将 /etc/yum.repos.d/ 里的 baseurl 改成 `http://你的IP/repos/<key>/`

## 下一步可以继续完善的点（告诉我你要不要）
- 自动生成客户端 .repo 文件的 task
- 支持 rsync:// 协议（比 http 更快）
- 定时同步（systemd timer）
- 更精细的 nginx 配置模板
- 多架构支持（aarch64 等）

仓库已按你的要求重构完毕，可以直接使用！
