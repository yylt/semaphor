# NUMA Role

This Ansible role detects NUMA (Non-Uniform Memory Access) topology on Linux systems, caches the information, and calculates NUMA configuration variables for use in systemd services.

## Features

1. **NUMA Detection**: Automatically detects NUMA nodes, CPU assignments, and memory availability
2. **Information Caching**: Caches NUMA topology to `/etc/ansible_numa_cache.json` for reuse
3. **Smart Allocation**: Calculates optimal NUMA node based on memory requirements
4. **Boundary Handling**: Only enables NUMA binding when node count meets threshold (default: 2)
5. **Variable Export**: Provides `numa_cpuaffinity` and `numa_policy` for systemd service configuration

## Design Philosophy

This role **does not** create or manage systemd services. Instead, it:
- Detects NUMA topology
- Calculates optimal NUMA configuration
- Exports variables (`numa_cpuaffinity`, `numa_policy`) for use in your own service templates

Each service role should manage its own systemd service files and use the NUMA variables when appropriate.

## Requirements

- Linux system with NUMA support
- `numactl` package (automatically installed if missing)
- systemd for service management

## Role Structure

```
roles/numa/
├── tasks/
│   ├── main.yml           # NUMA detection and caching
│   └── calculate.yml      # Calculate NUMA configuration
├── defaults/
│   └── main.yml           # Default variables
└── handlers/
    └── main.yml           # Systemd reload handler
```

## Usage

### 1. Detect and Cache NUMA Information

First, run the role to detect and cache NUMA topology:

```yaml
- hosts: servers
  roles:
    - numa
```

This will:
- Install `numactl` if not present
- Detect NUMA node count, CPU assignments, and memory
- Cache information to `/etc/ansible_numa_cache.json`

### 2. Calculate NUMA Configuration in Your Service Role

In your service role, include the calculate task:

```yaml
- name: Calculate NUMA configuration
  include_tasks: roles/numa/tasks/calculate.yml
  vars:
    required_memory_mb: 8192  # Optional: for node selection
    numa_node_threshold: 2     # Optional: minimum nodes to enable NUMA (default: 2)
```

This sets the following variables:
- `numa_config_enabled`: Boolean, true if NUMA binding should be used
- `numa_cpuaffinity`: CPU list for systemd CPUAffinity (e.g., "0-15,32-47")
- `numa_policy`: NUMA policy for systemd NUMAPolicy (e.g., "bind")
- `selected_numa_node`: Selected NUMA node ID

### 3. Use NUMA Variables in Your Service Template

In your systemd service template (e.g., `roles/myservice/templates/myservice.service.j2`):

```jinja2
[Unit]
Description=My Service
After=network.target

[Service]
Type=simple
User=myapp
ExecStart=/opt/myapp/bin/myapp

# NUMA Configuration (only if enabled)
{% if numa_config_enabled | default(false) %}
CPUAffinity={{ numa_cpuaffinity }}
NUMAPolicy={{ numa_policy }}
{% endif %}

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## Complete Example

### Example Service Role Structure

```
roles/myservice/
├── tasks/
│   └── main.yml
├── templates/
│   └── myservice.service.j2
└── defaults/
    └── main.yml
```

### Example Service Role Tasks

```yaml
---
# roles/myservice/tasks/main.yml

- name: Ensure NUMA information is available
  include_role:
    name: numa

- name: Calculate NUMA configuration for this service
  include_tasks: roles/numa/tasks/calculate.yml
  vars:
    required_memory_mb: 8192

- name: Deploy service with NUMA configuration
  template:
    src: myservice.service.j2
    dest: /etc/systemd/system/myservice.service
    owner: root
    group: root
    mode: '0644'
  notify: reload systemd

- name: Enable and start service
  systemd:
    name: myservice
    enabled: yes
    state: started
    daemon_reload: yes
```

### Example Service Template

```jinja2
# roles/myservice/templates/myservice.service.j2
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp

ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/config.yml

{% if numa_config_enabled | default(false) %}
# NUMA binding enabled ({{ numa_node_count }} nodes detected)
CPUAffinity={{ numa_cpuaffinity }}
NUMAPolicy={{ numa_policy }}
{% endif %}

Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## Variables

### Detection (main.yml)

No input variables required. Outputs:
- `numa_node_count`: Number of NUMA nodes
- `numa_nodes_info`: Dictionary with node details (CPUs, memory)

### Calculation (calculate.yml)

**Optional Input:**
- `required_memory_mb`: Memory requirement in MB (for node selection)
- `numa_node_threshold`: Minimum NUMA nodes to enable binding (default: 2)

**Outputs:**
- `numa_config_enabled`: Boolean, whether NUMA binding should be used
- `numa_cpuaffinity`: CPU list for systemd (e.g., "0-15,32-47")
- `numa_policy`: NUMA policy for systemd (e.g., "bind")
- `selected_numa_node`: Selected NUMA node ID
- `selected_numa_cpus`: CPU list for the selected node
- `selected_numa_memory_mb`: Available memory on selected node

### Default Variables

**roles/numa/defaults/main.yml:**
- `numa_cache_file`: `/etc/ansible_numa_cache.json` - Cache location
- `numa_node_threshold`: `2` - Minimum nodes to enable NUMA binding

## How It Works

### NUMA Detection

The role uses `numactl --hardware` to detect:
- Number of NUMA nodes
- CPU assignments per node
- Memory available per node

Example cached data:
```yaml
numa_node_count: 2
numa_nodes_info:
  '0':
    node_id: '0'
    cpus: '0-15,32-47'
    memory_mb: 65536
  '1':
    node_id: '1'
    cpus: '16-31,48-63'
    memory_mb: 65536
```

### Boundary Conditions

**NUMA binding is DISABLED when:**
- NUMA node count < `numa_node_threshold` (default: 2)
- In this case: `numa_config_enabled` = false, `numa_cpuaffinity` = "", `numa_policy` = ""

**NUMA binding is ENABLED when:**
- NUMA node count >= `numa_node_threshold`
- In this case: Variables are set with appropriate values

### NUMA Node Selection Algorithm

1. Loads cached NUMA information
2. Checks if node count meets threshold
3. If `required_memory_mb` is specified:
   - Finds NUMA nodes with sufficient memory
   - Selects first suitable node
   - Falls back to node 0 if none suitable
4. If `required_memory_mb` is not specified:
   - Defaults to node 0
5. Sets `numa_cpuaffinity` to the node's CPU list
6. Sets `numa_policy` to "bind"

### Systemd Integration

The generated variables are used in systemd service files:

```ini
[Service]
CPUAffinity=0-15,32-47
NUMAPolicy=bind
```

This configuration:
- Binds process to specific CPUs on the NUMA node
- Uses `NUMAPolicy=bind` to allocate memory from the local node
- Reduces memory access latency for performance-critical services

## Benefits

- **Performance**: Reduces memory access latency by binding to local NUMA node
- **Predictability**: Prevents memory allocation across NUMA boundaries
- **Flexibility**: Each service manages its own systemd configuration
- **Smart Defaults**: Automatically disables NUMA binding on single-node systems
- **Separation of Concerns**: NUMA role only calculates, services apply configuration

## Notes

- NUMA binding is only enabled when node count >= threshold (default: 2)
- Single-node systems automatically skip NUMA configuration
- NUMA cache is stored in `/etc/ansible_numa_cache.json`
- Each service role should include its own systemd service template
- Use `numa_config_enabled` to conditionally apply NUMA settings

## Troubleshooting

**Check NUMA topology:**
```bash
numactl --hardware
cat /etc/ansible_numa_cache.json
```

**Verify service NUMA binding:**
```bash
systemctl cat myservice  # Check CPUAffinity and NUMAPolicy
cat /proc/$(systemctl show -p MainPID --value myservice)/numa_maps
```

**Check if NUMA is enabled for a service:**
```bash
systemctl show myservice | grep -E "(CPUAffinity|NUMAPolicy)"
```

**Debug NUMA calculation:**
```yaml
- include_tasks: roles/numa/tasks/calculate.yml
  vars:
    required_memory_mb: 8192

- debug:
    msg: |
      NUMA enabled: {{ numa_config_enabled }}
      CPUAffinity: {{ numa_cpuaffinity }}
      NUMAPolicy: {{ numa_policy }}
```
