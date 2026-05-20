# dnf_repo_sync

Ansible role to synchronize (mirror) DNF/YUM repositories from upstream mirrors to a local directory. 

This is useful for:
- Creating local package mirrors for faster installs and updates
- Air-gapped or restricted network environments
- Reducing bandwidth usage and improving reliability
- Serving repos via HTTP/HTTPS for other servers in your infrastructure

## Requirements

- Ansible 2.15+
- Target OS: RHEL 8/9, Rocky Linux 8/9, AlmaLinux 8/9, CentOS Stream, Fedora (dnf based)
- Packages: dnf-utils (provides reposync), createrepo

## Role Variables

See `defaults/main.yml` for all configurable variables.

Key variables:

- `dnf_repo_sync_base_path`: Base directory for mirrored repos (default: `/var/www/html/repos`)
- `dnf_repo_sync_repos`: List of repositories to sync (name, repoid or baseurl, arch, releasever)
- `dnf_repo_sync_serve_repos`: Whether to install and configure web server (nginx/httpd) to serve the mirror
- `dnf_repo_sync_web_server`: `nginx` or `httpd`

## Usage

```yaml
- hosts: repo_mirrors
  become: true
  roles:
    - dnf_repo_sync
```

Or with custom vars in playbook/group_vars.

After run, repos will be available under `{{ dnf_repo_sync_base_path }}/<repo-name>/`

You can then point client /etc/yum.repos.d/ files to `http://your-mirror/repos/...`

## Tags

- `dnf`
- `repo`
- `sync`
- `mirror`

## Notes / TODO

- Currently basic implementation. Extend with more repo definitions, GPG key handling, delta metadata, and scheduled sync via cron or systemd timer.
- Supports both repoid (requires enabled repo on target) and direct baseurl.
- For production, consider using Pulp, Katello, or Nexus for advanced repo management.
- This role is part of ansible-server-management collection of roles for daily ops.

## License

MIT
