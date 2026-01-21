# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Ansible automation repository for provisioning and configuring Linux systems (both Debian/Ubuntu and RedHat/CentOS/AlmaLinux). It manages system setup, package installation, service deployment, and development environment configuration across multiple hosts.

## Running Playbooks

### Main Playbooks

- **New host setup**: `ansible-playbook -i inventory/inventory.ini newhost.yml`
  - Applies `default` and `apps` roles to hosts in the `newhost` group

- **Service deployment**: `ansible-playbook -i inventory/inventory.ini service.yml`
  - Applies `default` and `service` roles to hosts in the `runner` group

- **Application installation**: `ansible-playbook -i inventory/inventory.ini apps.yml`
  - Applies `default` and `apps` roles to specified hosts

- **Gather facts**: `ansible-playbook -i inventory/inventory.ini facts.yml`
  - Collects and displays system facts for debugging

- **NUMA example**: `ansible-playbook -i inventory/inventory.ini numa_example.yml`
  - Demonstrates NUMA detection and service deployment with NUMA binding

### Inventory Management

- Inventory file: `inventory/inventory.ini`
- Group variables: `inventory/group_vars/all.yml`
- Host groups: `newhost`, `runner`
- All hosts use Python 3 interpreter (`ansible_python_interpreter=/usr/bin/python3`)

## Architecture

### Role Structure

The repository follows standard Ansible role structure with these key roles:

#### 1. **new** (Base System Configuration)
Located in `roles/new/`, this role handles initial system setup:
- **OS-specific package management**: Separate tasks for Debian (`debian.yml`) and RedHat (`redhat.yml`) families
- **Repository configuration**: Sets up mirrors (Aliyun for China) based on distribution and architecture
- **SSH hardening**: Disables strict host key checking, configures known_hosts
- **User environment**: Creates `.proxy`, `.gitconfig`, `.vimrc` for all bash users
- **Service management**: Disables unwanted services
- **Custom hosts**: Updates `/etc/hosts` with custom entries from `custom_hosts` variable

Key implementation details:
- Backs up existing YUM repos before reconfiguration (roles/new/tasks/redhat.yml:10-14)
- Dynamically sets APT repo paths based on architecture (aarch64 uses ubuntu-ports) (roles/new/tasks/debian.yml:6-9)
- SELinux is disabled on RedHat systems (roles/new/tasks/redhat.yml:59-73)

#### 2. **dns**
Located in `roles/dns/`, manages DNS and timezone:
- Configures NetworkManager to not manage DNS
- Sets custom DNS servers via `/etc/resolv.conf`
- Sets timezone to Asia/Shanghai

#### 3. **tools**
Located in `roles/tools/`, installs development tools and languages:
- **Language runtimes**: Downloads and installs Node.js and Go from mirrors (roles/tools/defaults/main.yml:4-11)
- **GitHub tools**: Downloads releases from GitHub (with proxy support) for tools like `crane`, `uv` (roles/tools/tasks/github.yml)
- **Environment setup**: Configures PATH and environment variables (roles/tools/tasks/env.yml)

Installation pattern:
1. Creates temporary directory with `mktemp`
2. Downloads from mirrors or GitHub releases
3. Extracts to `/usr/local/<tool_name>` with `--strip-components=1`
4. Finds and copies executables to `/usr/local/bin`
5. Cleans up temporary files

#### 4. **apps**
Located in `roles/apps/`, manages desktop applications:
- **AppImage deployment**: Extracts and installs AppImage files (roles/apps/tasks/appimage.yml)
  - Mounts AppImage with `--appimage-extract`
  - Extracts `.desktop` files and icons
  - Installs to `/usr/local/share/applications` and `/usr/local/share/icons`
- **APT repositories**: Adds third-party repos (Mozilla Firefox, VSCodium) with GPG keys (roles/apps/tasks/repo.yml)
- **UI packages**: Installs desktop applications like Firefox, GIMP, KDE Connect (controlled by `install_ui_packages` flag)

#### 5. **service**
Located in `roles/service/`, manages service updates:
- Downloads service binaries from GitHub releases
- Extracts archives and finds matching executables
- Copies to destination and restarts systemd services
- Example: EasyTier VPN service (roles/service/defaults/main.yml:3-6)

Update pattern (roles/service/tasks/service.yml):
1. Downloads archive from URL (supports architecture variables)
2. Extracts to temporary directory
3. Finds matching file by basename
4. Copies to destination with proper permissions
5. Restarts systemd service if specified
6. Cleans up temporary files

#### 6. **numa** (NUMA Configuration Provider)
Located in `roles/numa/`, provides NUMA topology detection and configuration variables for service roles:
- **NUMA detection**: Detects NUMA nodes, CPU assignments, and memory using `numactl` (roles/numa/tasks/main.yml)
- **Information caching**: Caches topology to `/etc/ansible_numa_cache.json` for reuse
- **Smart allocation**: Calculates optimal NUMA node based on memory requirements (roles/numa/tasks/calculate.yml)
- **Variable export**: Provides `numa_cpuaffinity` and `numa_policy` for use in service templates

Design philosophy:
- **Separation of concerns**: NUMA role only detects and calculates, does NOT manage services
- **Boundary handling**: Automatically disables NUMA binding when node count < threshold (default: 2)
- **Flexible integration**: Each service role uses NUMA variables in its own systemd templates

NUMA workflow:
1. Run role to detect and cache NUMA topology (installs `numactl` if needed)
2. In service role, include `calculate.yml` with optional `required_memory_mb` parameter
3. Use exported variables (`numa_cpuaffinity`, `numa_policy`) in service template
4. Service template conditionally applies NUMA settings based on `numa_config_enabled`

Key features:
- Selects NUMA node with sufficient memory for the workload
- Exports CPUAffinity and NUMAPolicy values for systemd
- Automatically skips NUMA binding on single-node systems
- No service management - each role handles its own services
- Reduces memory access latency for performance-critical services

Example usage pattern (see roles/numa/README.md):
```yaml
# In service role tasks:
- include_tasks: roles/numa/tasks/calculate.yml
  vars:
    required_memory_mb: 8192

# In service template:
{% if numa_config_enabled | default(false) %}
CPUAffinity={{ numa_cpuaffinity }}
NUMAPolicy={{ numa_policy }}
{% endif %}
```

### Key Variables

**Global defaults** (roles/default/defaults/main.yml):
- `tool_dir`: `/usr/local/bin` - where executables are installed
- `github_proxy_url`: `https://ghfast.top/` - proxy for GitHub downloads

**Apps role** (roles/apps/defaults/main.yml):
- `signkey_dir`: `/etc/apt/keyrings` - APT signing keys
- `applications_dir`: `/usr/local/share/applications` - desktop entries
- `icons_dir`: `/usr/local/share/icons` - application icons
- `apt_repos`: List of third-party repositories (Mozilla, VSCodium)
- `apt_ui_packages`: Desktop applications to install

**Tools role** (roles/tools/defaults/main.yml):
- `lang_tools`: Language runtimes with download URLs (Node.js, Go)
- `github_tools`: GitHub releases to download (crane, uv)

**Service role** (roles/service/defaults/main.yml):
- `update_services`: List of services to update with source URLs and destinations

**NUMA role** (roles/numa/defaults/main.yml):
- `numa_cache_file`: `/etc/ansible_numa_cache.json` - NUMA topology cache location
- `numa_node_threshold`: `2` - Minimum NUMA nodes to enable binding (default: 2)

### Architecture Patterns

1. **OS Family Abstraction**: Tasks branch on `ansible_os_family` (Debian vs RedHat) and `ansible_distribution` (Ubuntu vs Debian vs AlmaLinux vs CentOS)

2. **Architecture Support**: Uses `ansible_architecture` to select correct binaries (aarch64/arm64 vs x86_64/amd64)

3. **Idempotency**: Uses `command -v` checks before installing tools, `creates` parameter for file operations

4. **Temporary File Management**: Creates unique temp directories with `mktemp -d /tmp/ansible-XXXXXX`, always cleans up

5. **Mirror/Proxy Support**: Uses Chinese mirrors (Aliyun, npmmirror) and GitHub proxy for faster downloads

6. **Template-Driven Configuration**: Uses Jinja2 templates for repo files, resolv.conf, etc.

## Testing

No automated tests are present. Test changes by:
1. Running playbooks against test hosts in the `newhost` group
2. Using `facts.yml` to verify system state
3. Checking service status with `systemctl status <service>`

## Important Notes

- All hosts are accessed as root user (`ansible_ssh_user=root`)
- Proxy settings are configured in user home directories (`.proxy` file)
- GitHub downloads use proxy at `https://ghfast.top/`
- Language tools are installed to `/usr/local/<tool_name>` not system package managers
- SELinux is disabled on RedHat systems
- NetworkManager DNS management is disabled when present
