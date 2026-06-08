# Disk Verification Role

This role checks disk space usage on target systems and reports on filesystems that need attention. It's a read-only operation that makes no changes to the system.

## Purpose

- Monitor disk space usage across filesystems
- Identify filesystems approaching capacity limits
- Generate actionable reports with usage statistics
- Set facts for downstream roles to use

## Requirements

- Ansible 2.9+
- Target systems: Linux (any distribution)
- Privileges: Read access to filesystem information (typically requires sudo)

## Role Variables

### Defaults

All variables are defined in [`defaults/main.yml`](defaults/main.yml:1):

```yaml
# Threshold for warning status (percentage of disk usage)
disk_space_warning_threshold: 80

# Threshold for critical status (percentage of disk usage)
disk_space_critical_threshold: 90

# Filesystems to monitor (empty list = monitor all)
monitored_filesystems: []

# Whether to fail the playbook when critical threshold is exceeded
fail_on_critical: false

# Whether to display detailed filesystem information
show_detailed_info: true

# Minimum size threshold for filesystems to monitor (in GB)
min_filesystem_size_gb: 1
```

### Variable Descriptions

- **disk_space_warning_threshold**: Percentage at which a filesystem is considered at warning level (default: 80%)
- **disk_space_critical_threshold**: Percentage at which a filesystem is considered critical (default: 90%)
- **monitored_filesystems**: List of specific mountpoints to monitor. If empty, all filesystems are checked
- **fail_on_critical**: If true, the playbook will fail when any filesystem exceeds the critical threshold
- **show_detailed_info**: Display detailed information for each filesystem
- **min_filesystem_size_gb**: Skip filesystems smaller than this size (helps filter out tiny virtual filesystems)

## Facts Set by This Role

After running, this role sets the following facts that can be used by other roles or playbooks:

### disk_space_status
Overall status: `"ok"`, `"warning"`, or `"critical"`

### filesystems_info
List of dictionaries containing information about each filesystem:
```yaml
- filesystem: "/dev/sda1"
  size_bytes: 107374182400
  used_bytes: 85899345920
  available_bytes: 21474836480
  use_percent: 80
  mountpoint: "/"
```

### filesystems_warning
List of filesystems at warning level (between warning and critical thresholds)

### filesystems_critical
List of filesystems at critical level (at or above critical threshold)

### filesystems_needing_cleanup
Combined list of all filesystems needing attention (warning + critical)

### disk_verification_results
Complete verification results including:
```yaml
disk_verification_results:
  status: "warning"
  timestamp: "2024-01-15T10:30:00Z"
  filesystems_checked: 5
  filesystems_warning: 2
  filesystems_critical: 0
  filesystems_needing_cleanup: [...]
  thresholds:
    warning: 80
    critical: 90
```

## Dependencies

None. This role is self-contained.

## Example Playbook

### Basic Usage

```yaml
---
- name: Check disk space
  hosts: all
  become: true
  
  roles:
    - disk_verification
```

### With Custom Thresholds

```yaml
---
- name: Check disk space with custom thresholds
  hosts: webservers
  become: true
  
  roles:
    - role: disk_verification
      vars:
        disk_space_warning_threshold: 75
        disk_space_critical_threshold: 85
        fail_on_critical: true
```

### Monitor Specific Filesystems

```yaml
---
- name: Check specific filesystems
  hosts: databases
  become: true
  
  roles:
    - role: disk_verification
      vars:
        monitored_filesystems:
          - /
          - /var
          - /data
```

### Use Results in Subsequent Tasks

```yaml
---
- name: Check disk space and act on results
  hosts: all
  become: true
  
  tasks:
    - name: Run disk verification
      include_role:
        name: disk_verification
    
    - name: Send alert if critical
      debug:
        msg: "ALERT: {{ inventory_hostname }} has critical disk space!"
      when: disk_space_status == 'critical'
    
    - name: List filesystems needing cleanup
      debug:
        msg: "{{ item.mountpoint }} is at {{ item.use_percent }}%"
      loop: "{{ filesystems_needing_cleanup }}"
```

## Tags

This role supports the following tags:

- `verification`: Run verification tasks
- `facts`: Gather filesystem facts
- `reporting`: Display reports

### Tag Usage Examples

```bash
# Run only verification tasks
ansible-playbook playbook.yml --tags verification

# Skip reporting
ansible-playbook playbook.yml --skip-tags reporting
```

## Output Examples

### When All Filesystems Are OK

```
TASK [disk_verification : Display disk space summary]
ok: [server01] => {
    "msg": [
        "=== Disk Space Verification Summary ===",
        "Overall Status: OK",
        "Total Filesystems Checked: 5",
        "Filesystems at Warning Level (80%+): 0",
        "Filesystems at Critical Level (90%+): 0"
    ]
}
```

### When Filesystems Need Attention

```
TASK [disk_verification : Display disk space summary]
ok: [server01] => {
    "msg": [
        "=== Disk Space Verification Summary ===",
        "Overall Status: WARNING",
        "Total Filesystems Checked: 5",
        "Filesystems at Warning Level (80%+): 2",
        "Filesystems at Critical Level (90%+): 0"
    ]
}

TASK [disk_verification : Display filesystems needing cleanup]
ok: [server01] => (item={'mountpoint': '/var', 'use_percent': 82}) => {
    "msg": [
        "=== Filesystems Requiring Attention ===",
        "/var: 82% used (15.5 GB available)"
    ]
}
```

## Integration with Cleanup Role

This role is designed to work seamlessly with the `filesystem_cleanup` role:

```yaml
---
- name: Verify and cleanup if needed
  hosts: all
  become: true
  
  tasks:
    - name: Check disk space
      include_role:
        name: disk_verification
    
    - name: Run cleanup if needed
      include_role:
        name: filesystem_cleanup
      when: disk_space_status in ['warning', 'critical']
```

## Troubleshooting

### Issue: "df: command not found"

**Cause**: The `df` command is not available on the target system.

**Solution**: Install coreutils package:
```bash
# Debian/Ubuntu
apt-get install coreutils

# RHEL/CentOS
yum install coreutils
```

### Issue: Incorrect filesystem sizes reported

**Cause**: Different filesystem types report sizes differently.

**Solution**: The role uses `df -B1` for consistent byte-level reporting across all filesystem types.

### Issue: Too many small filesystems reported

**Cause**: Virtual filesystems (tmpfs, devtmpfs, etc.) are being included.

**Solution**: Increase `min_filesystem_size_gb` or specify `monitored_filesystems` explicitly:
```yaml
min_filesystem_size_gb: 5  # Skip filesystems smaller than 5GB
```

## Best Practices

1. **Run Regularly**: Schedule this role to run daily or weekly for proactive monitoring
2. **Set Appropriate Thresholds**: Adjust thresholds based on your environment and growth patterns
3. **Monitor Trends**: Track disk usage over time to predict when cleanup will be needed
4. **Use with Alerting**: Integrate with monitoring systems to alert on critical status
5. **Document Baselines**: Know your normal disk usage patterns for each system

## License

[Specify your license]

## Author

[Your name/organization]