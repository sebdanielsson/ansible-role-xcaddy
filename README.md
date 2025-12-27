# xcaddy (Caddy using xcaddy)

This role installs Caddy using xcaddy on Linux systems. It builds a custom Caddy binary with user-specified modules, deploys a Caddyfile configuration, and manages the Caddy systemd service.

## Features

- **Custom Caddy builds** with user-specified modules via xcaddy
- **Idempotent builds** - only rebuilds when Caddy version or modules change
- **Caddyfile configuration** - supports both raw content and Jinja2 templates
- **Systemd integration** with proper capabilities for binding to privileged ports
- **Config validation** before reloading to prevent broken deployments
- **Multi-distro support** - consistent Go-based installation across all supported distros

## Requirements

- systemd init system
- **Debian family** (Debian, Ubuntu, Raspbian) - uses Go package + `go install`
- **RedHat family** (Fedora, RHEL, CentOS, AlmaLinux, RockyLinux) - uses Go package + `go install`
- **Arch Linux family** (Arch Linux, Manjaro) - uses Go package + `go install`
- **Other distros** - uses Go package + `go install` (best effort)

## Role Variables

### State Control

| Variable | Default | Description |
| -------------- | -------- | ------------- |
| `xcaddy_state` | `present` | Role state: `present`, `latest`, or `absent` |
| `xcaddy_caddy_version` | `latest` | Caddy version to build (e.g., `v2.10.2` or `latest`) |

### Modules

| Variable | Default | Description |
| ---------------- | ------- | ------------------------------------------------- |
| `xcaddy_modules` | `[]` | List of Caddy module URLs, and optionally versions, to include in the build |

### Configuration

| Variable | Default | Description |
| ---------- | --------- | ------------- |
| `xcaddy_caddyfile_content` | `""` | Raw Caddyfile content (mutually exclusive with template) |
| `xcaddy_caddyfile_template` | `""` | Path to Caddyfile Jinja2 template (mutually exclusive with content) |
| `xcaddy_caddyfile_path` | `/etc/caddy/Caddyfile` | Path where Caddyfile will be deployed |
| `xcaddy_config_dir` | `/etc/caddy` | Caddy configuration directory |
| `xcaddy_data_dir` | `/var/lib/caddy` | Caddy data directory (certificates, etc.) |

### Service

| Variable | Default | Description |
| ---------- | --------- | ------------- |
| `xcaddy_service_enabled` | `true` | Enable Caddy service at boot |
| `xcaddy_service_started` | `true` | Start Caddy service after configuration |
| `xcaddy_user` | `caddy` | User to run Caddy service |
| `xcaddy_group` | `caddy` | Group for Caddy service |

### Paths

| Variable | Default | Description |
| ---------- | --------- | ------------- |
| `xcaddy_bin_path` | `/usr/local/bin/caddy` | Path to install Caddy binary |
| `xcaddy_bin` | `xcaddy` | Path to xcaddy binary (auto-configured) |
| `xcaddy_build_dir` | `/tmp/xcaddy-build` | Temporary build directory |
| `xcaddy_go_version` | `latest` | Go version to install. E.g., "1.25.5", or "latest" for auto-detection |
| `xcaddy_goproxy` | `$GOPROXY` or `https://proxy.golang.org,direct` | Go module proxy URL |

## Example Playbooks

### Basic Installation (No Modules)

```yaml
- hosts: webservers
  roles:
    - role: sebdanielsson.xcaddy
      vars:
        xcaddy_caddyfile_content: |
          :80 {
            respond "Hello, World!"
          }
```

### With Custom Modules (Cloudflare DNS)

```yaml
- hosts: webservers
  roles:
    - role: sebdanielsson.xcaddy
      vars:
        xcaddy_modules:
          - github.com/caddy-dns/cloudflare
        xcaddy_caddyfile_content: |
          example.com {
            tls {
              dns cloudflare {env.CF_API_TOKEN}
            }
            reverse_proxy localhost:8080
          }
```

### With Caddyfile Template

```yaml
- hosts: webservers
  roles:
    - role: sebdanielsson.xcaddy
      vars:
        xcaddy_modules:
          - github.com/caddy-dns/cloudflare
        xcaddy_caddyfile_template: "templates/Caddyfile.j2"
```

### Specific Caddy Version and Cloudflare module version

```yaml
- hosts: webservers
  roles:
    - role: sebdanielsson.xcaddy
      vars:
        xcaddy_caddy_version: "v2.10.2"
        xcaddy_modules:
          - github.com/caddy-dns/cloudflare@v0.2.2
        xcaddy_caddyfile_content: |
          example.com {
            tls {
              dns cloudflare {env.CF_API_TOKEN}
            }
            reverse_proxy localhost:8080
          }
```

### Uninstall Caddy

```yaml
- hosts: webservers
  roles:
    - role: sebdanielsson.xcaddy
      vars:
        xcaddy_state: absent
```

## Idempotency

The role implements intelligent rebuild detection:

1. **Version check**: Compares installed Caddy version against `xcaddy_caddy_version`
2. **Module check**: Compares installed module versions against `xcaddy_modules` list
3. **Rebuild trigger**: Only runs `xcaddy build` if version or modules differ or if unpinned modules are used

When `xcaddy_state: latest`, the role will always rebuild to fetch the latest Caddy version.

## Handlers

The role includes the following handlers:

- **Validate Caddy config**: Runs `caddy validate` before reload
- **Reload Caddy**: Reloads Caddy configuration (triggered by Caddyfile changes)
- **Restart Caddy**: Restarts Caddy service (triggered by binary changes)

## Setup development environment

### 1. Clone the Repository

```sh
git clone https://github.com/sebdanielsson/ansible-role-xcaddy.git
cd ansible-role-xcaddy
```

### 2. Create and Activate Virtual Environment

```sh
uv venv
source .venv/bin/activate
```

### 3. Install Python Dependencies

```sh
uv pip install -r requirements.txt --force-reinstall
```

### 4. Install Ansible Collections and Roles

```sh
ansible-galaxy install -r requirements.yml --force
```

### 5. Verify Installation

```sh
ansible --version
ansible-lint --version
yamllint --version
```

## License

MIT

## Author Information

Created by Sebastian Danielsson 2025  
GitHub Profile: <https://github.com/sebdanielsson>  
Website: <https://sebdanielsson.dev>
