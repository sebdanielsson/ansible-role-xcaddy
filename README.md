# xcaddy (Caddy using xcaddy)

This role installs Caddy using xcaddy on Linux systems. It builds a custom Caddy binary with user-specified modules, deploys a Caddyfile configuration, and manages the Caddy systemd service.

## Features

- **Custom Caddy builds** with user-specified modules via xcaddy
- **Idempotent builds** - only rebuilds when Caddy version or modules change
- **Caddyfile configuration** - supports both raw content and Jinja2 templates
- **Systemd integration** with proper capabilities for binding to privileged ports
- **Optional `EnvironmentFile`** to pass environment variables to the Caddy process
- **Fully idempotent** - converges cleanly with no spurious reloads or restarts
- **Config validation** before reloading to prevent broken deployments
- **Multi-distro support** - consistent Go-based installation across all supported distros

## Requirements

- A systemd init system
- `ansible-core` >= 2.17.14

Installation is deliberately distro-agnostic — there are no per-distro package paths. The role
resolves the latest stable Go from the [go.dev release API](https://go.dev/dl/?mode=json) (or the
version pinned in `xcaddy_go_version`), installs the toolchain to `/usr/local/go`, then uses it to
`go install` xcaddy and build Caddy. Any systemd distro on `amd64` or `arm64` that can fetch and
extract a tarball should work.

CI runs the Molecule suite against Debian 13, Ubuntu 24.04, Fedora 43, and RHEL UBI 10 on every PR.
Arch Linux and Manjaro are expected to work through the same generic path but aren't covered by the
test matrix.

## Role Variables

### State Control

| Variable               | Default   | Description                                          |
| ---------------------- | --------- | ---------------------------------------------------- |
| `xcaddy_state`         | `present` | Role state: `present`, `latest`, or `absent`         |
| `xcaddy_caddy_version` | `latest`  | Caddy version to build (e.g., `v2.10.2` or `latest`) |

### Modules

| Variable         | Default | Description                                                                 |
| ---------------- | ------- | --------------------------------------------------------------------------- |
| `xcaddy_modules` | `[]`    | List of Caddy module URLs, and optionally versions, to include in the build |

### Configuration

| Variable                    | Default                | Description                                                         |
| --------------------------- | ---------------------- | ------------------------------------------------------------------- |
| `xcaddy_caddyfile_content`  | `""`                   | Raw Caddyfile content (mutually exclusive with template)            |
| `xcaddy_caddyfile_template` | `""`                   | Path to Caddyfile Jinja2 template (mutually exclusive with content) |
| `xcaddy_caddyfile_path`     | `/etc/caddy/Caddyfile` | Path where Caddyfile will be deployed                               |
| `xcaddy_config_dir`         | `/etc/caddy`           | Caddy configuration directory                                       |
| `xcaddy_data_dir`           | `/var/lib/caddy`       | Caddy data directory (certificates, etc.)                           |

### Service

| Variable                   | Default | Description                                                                                                  |
| -------------------------- | ------- | ------------------------------------------------------------------------------------------------------------ |
| `xcaddy_service_enabled`   | `true`  | Enable Caddy service at boot                                                                                 |
| `xcaddy_service_started`   | `true`  | Start Caddy service after configuration                                                                      |
| `xcaddy_user`              | `caddy` | User to run Caddy service                                                                                    |
| `xcaddy_group`             | `caddy` | Group for Caddy service                                                                                      |
| `xcaddy_environment_file`  | `""`    | Optional path for the systemd `EnvironmentFile=` directive (omitted when empty)                              |
| `xcaddy_environment`       | `{}`    | Optional dict of environment variables; rendered to `xcaddy_environment_file` when both are set              |

### Paths

| Variable            | Default                                         | Description                                                           |
| ------------------- | ----------------------------------------------- | --------------------------------------------------------------------- |
| `xcaddy_bin_path`   | `/usr/local/bin/caddy`                          | Path to install Caddy binary                                          |
| `xcaddy_bin`        | `xcaddy`                                        | Path to xcaddy binary (auto-configured)                               |
| `xcaddy_build_dir`  | `/tmp/xcaddy-build`                             | Temporary build directory                                             |
| `xcaddy_go_version` | `latest`                                        | Go version to install. E.g., "1.25.5", or "latest" for auto-detection |
| `xcaddy_goproxy`    | `$GOPROXY` or `https://proxy.golang.org,direct` | Go module proxy URL                                                   |

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

### With an Environment File (DNS provider credentials)

Pass secrets to Caddy via a systemd `EnvironmentFile` instead of embedding them
in the Caddyfile. The role renders `xcaddy_environment` to the file when both
variables are set; omit `xcaddy_environment` to manage the file out of band.

```yaml
- hosts: webservers
  roles:
    - role: sebdanielsson.xcaddy
      vars:
        xcaddy_modules:
          - github.com/caddy-dns/cloudflare
        xcaddy_environment_file: /etc/caddy/caddy.env
        xcaddy_environment:
          CF_API_TOKEN: "{{ vault_cf_api_token }}"
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

When `xcaddy_caddy_version: latest`, the role resolves the latest upstream Caddy GitHub release tag and rebuilds only when the installed binary is behind that release.

When `xcaddy_state: latest`, the role will always rebuild to refresh the binary even if the installed version already matches the latest release.

## Handlers

The role includes the following handlers:

- **Validate Caddy config**: Runs `caddy validate` before reload
- **Reload Caddy**: Reloads Caddy configuration (triggered by Caddyfile changes)
- **Reload systemd daemon**: Runs `systemctl daemon-reload` (triggered only when the unit file changes)
- **Restart Caddy**: Restarts Caddy service (triggered by binary, unit file, or environment file changes)

Handlers only fire when the resource they watch actually changes, so a converged
host re-runs with zero changes and no service disruption.

## Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for how to set
up a development environment, run the linters and formatters, and execute the
Molecule test suite locally.

## Releases

Releases are automated with [Release Please](https://github.com/googleapis/release-please) and
published to
[Ansible Galaxy](https://galaxy.ansible.com/ui/standalone/roles/sebdanielsson/xcaddy/). The version
and [`CHANGELOG.md`](CHANGELOG.md) are derived from conventional commits — see
[CONTRIBUTING.md](CONTRIBUTING.md#releases).

## License

MIT

## Author Information

Created by Sebastian Danielsson 2025  
GitHub Profile: <https://github.com/sebdanielsson>  
Website: <https://sebdanielsson.dev>
