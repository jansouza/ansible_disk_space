# Filesystem Cleanup Role

This role performs safe, automated cleanup of disk space on Linux systems by removing old temporary files, compressing/removing old logs, cleaning package caches, and trimming systemd journals.

## Purpose

- Free up disk space on systems approaching capacity
- Clean temporary files from /tmp
- Compress and remove old log files
- Clear package manager caches
- Trim systemd journal logs
- Support dry-run mode for safety

## Requirements

- Ansible 2.9+
- Target systems: Ubuntu, Debian, CentOS, RHEL
- Privileges: sudo/root access required
- Collections: `community.general` (for archive module)

### Installing Required Collections

```bash
ansible-galaxy collection install community.general
```

## Role Variables

### Defaults

All variables are defined in [`defaults/main.yml`](defaults/main.yml:1):

```yaml
# SAFETY: Dry-run mode is ENABLED by default
cleanup_dry_run: true

# Cleanup operation toggles
cleanup_tmp_enabled: true
cleanup_logs_enabled: true
cleanup_package_cache_enabled: true
cleanup_journals_enabled: true

# /tmp cleanup settings
tmp_file_age_days: 7
tmp_exclude_patterns:
  - ".X*"
  - ".ICE-unix"
  - "systemd-*"
  - ".X11-unix"
  - ".font-unix"
  - "ssh-*"

# /var/log cleanup settings
log_file_age_days: 30
log_compress_age_days: 7
log_exclude_patterns:
  - "*.gz"
  - "*.bz2"
  - "*.xz"
  - "lastlog"
  - "wtmp"
  - "btmp"

# Package cache settings
package_cache_keep_latest: 1
package_autoremove_enabled: true

# Journal settings
journal_max_size: "500M"
journal_max_time: "30d"
journal_verify_before_cleanup: true

# Reporting settings
show_detailed_reports: true
calculate_space_freed: true
```

### Variable Descriptions

#### Safety Controls
- **cleanup_dry_run**: When `true`, shows what would be deleted without making changes (default: `true`)

#### Operation Toggles
- **cleanup_tmp_enabled**: Enable /tmp cleanup
- **cleanup_logs_enabled**: Enable /var/log cleanup
- **cleanup_package_cache_enabled**: Enable package cache cleanup
- **cleanup_journals_enabled**: Enable systemd journal cleanup

#### /tmp Settings
- **tmp_file_age_days**: Delete files older than this many days (default: 7)
- **tmp_exclude_patterns**: Patterns to exclude from deletion (protects system files)
- **tmp_exclude_paths**: Full paths to exclude

#### /var/log Settings
- **log_file_age_days**: Delete logs older than this many days (default: 30)
- **log_compress_age_days**: Compress logs older than this many days (default: 7)
- **log_exclude_patterns**: Patterns to exclude from cleanup
- **log_file_patterns**: Patterns to target for cleanup

#### Package Cache Settings
- **package_cache_keep_latest**: Number of package versions to keep (0 = clean all)
- **package_autoremove_enabled**: Remove unused package dependencies

#### Journal Settings
- **journal_max_size**: Maximum total size for journals (default: "500M")
- **journal_max_time**: Maximum age for journal entries (default: "30d")
- **journal_verify_before_cleanup**: Verify journal integrity before cleanup

## Facts Set by This Role

After running, this role sets the following facts:

### cleanup_stats
Detailed statistics for each cleanup operation:
```yaml
cleanup_stats:
  tmp:
    files_removed: 150
    dirs_removed: 10
    space_freed_mb: 245.5
    dry_run: false
  logs:
    files_compressed: 25
    files_deleted: 10
    space_freed_mb: 180.2
    dry_run: false
  package_cache:
    package_manager: "apt"
    cache_size_before: "1.2G"
    cache_size_after: "150M"
    dry_run: false
  journals:
    size_before: "800M"
    size_after: "450M"
    max_size: "500M"
    max_time: "30d"
    dry_run: false
```

### filesystem_cleanup_results
Complete cleanup results:
```yaml
filesystem_cleanup_results:
  timestamp: "2024-01-15T10:30:00Z"
  dry_run: false
  operations_performed:
    tmp: true
    logs: true
    package_cache: true
    journals: true
  statistics: { ... }
  total_space_freed_mb: 425.7
```

## Dependencies

None. This role is self-contained.

## Example Playbook

### Basic Usage (Dry-Run)

```yaml
---
- name: Cleanup disk space (dry-run)
  hosts: all
  become: true
  
  roles:
    - filesystem_cleanup
```

### Actual Cleanup

```yaml
---
- name: Cleanup disk space (actual)
  hosts: all
  become: true
  
  roles:
    - role: filesystem_cleanup
      vars:
        cleanup_dry_run: false
```

### Selective Cleanup

```yaml
---
- name: Clean only /tmp and logs
  hosts: webservers
  become: true
  
  roles:
    - role: filesystem_cleanup
      vars:
        cleanup_dry_run: false
        cleanup_package_cache_enabled: false
        cleanup_journals_enabled: false
```

### Custom Age Thresholds

```yaml
---
- name: Aggressive cleanup
  hosts: all
  become: true
  
  roles:
    - role: filesystem_cleanup
      vars:
        cleanup_dry_run: false
        tmp_file_age_days: 3
        log_file_age_days: 14
        log_compress_age_days: 3
```

### Integration with Verification Role

```yaml
---
- name: Verify and cleanup if needed
  hosts: all
  become: true
  
  tasks:
    - name: Check disk space
      include_role:
        name: disk_verification
    
    - name: Run cleanup if critical
      include_role:
        name: filesystem_cleanup
      vars:
        cleanup_dry_run: false
      when: disk_space_status == 'critical'
```

## Tags

This role supports the following tags:

- `cleanup`: All cleanup operations
- `tmp`: /tmp cleanup only
- `logs`: /var/log cleanup only
- `cache`: Package cache cleanup only
- `journals`: Systemd journal cleanup only
- `reporting`: Display reports

### Tag Usage Examples

```bash
# Run only /tmp cleanup
ansible-playbook playbook.yml --tags tmp

# Run logs and cache cleanup
ansible-playbook playbook.yml --tags logs,cache

# Skip journal cleanup
ansible-playbook playbook.yml --skip-tags journals
```

## Cleanup Operations Details

### 1. /tmp Cleanup

**What it does:**
- Finds files older than `tmp_file_age_days`
- Excludes system-critical patterns
- Removes old files and empty directories

**Safety features:**
- Age-based deletion only
- Exclude patterns for system files
- Dry-run mode support

**Example output:**
```
=== /tmp Cleanup Summary (DRY-RUN) ===
Files to be deleted: 150
Directories to be deleted: 10
Space to be freed: 245.50 MB
Age threshold: 7 days
```

### 2. /var/log Cleanup

**What it does:**
- Compresses logs older than `log_compress_age_days`
- Deletes logs older than `log_file_age_days`
- Excludes already compressed files and system logs

**Safety features:**
- Two-stage process (compress, then delete)
- Exclude patterns for critical logs
- Age-based operations

**Example output:**
```
=== /var/log Cleanup Results ===
Log files compressed: 25
Log files deleted: 10
Space freed: 180.20 MB
```

### 3. Package Cache Cleanup

**What it does:**
- **Ubuntu/Debian**: Runs `apt-get clean`, `autoclean`, and `autoremove`
- **CentOS/RHEL 7**: Runs `yum clean all` and `autoremove`
- **CentOS/RHEL 8+**: Runs `dnf clean all` and `autoremove`

**Safety features:**
- Package caches can be rebuilt
- Autoremove only removes unused packages
- OS-specific handling

**Example output:**
```
=== APT Cache Cleanup Results ===
Cache size before: 1.2G
Cache size after: 150M
```

### 4. Systemd Journal Cleanup

**What it does:**
- Vacuums journals by size using `journalctl --vacuum-size`
- Vacuums journals by time using `journalctl --vacuum-time`
- Optionally verifies journal integrity first

**Safety features:**
- Journals preserved within limits
- Integrity check before cleanup
- Graceful handling if journalctl unavailable

**Example output:**
```
=== Systemd Journal Cleanup Results ===
Journal size before: 800M
Journal size after: 450M
```

## Safety Considerations

### 1. Dry-Run by Default
The role defaults to dry-run mode (`cleanup_dry_run: true`). You must explicitly set it to `false` to perform actual cleanup.

### 2. File Age Protection
Files newer than the configured age thresholds are never deleted:
- /tmp: 7 days (default)
- /var/log: 30 days for deletion, 7 days for compression (default)

### 3. Exclude Patterns
System-critical files are excluded by default:
- X11 sockets
- systemd files
- SSH agent sockets
- System logs (wtmp, btmp, lastlog)

### 4. Confirmation Prompt
When running in actual cleanup mode, the role displays a warning prompt requiring user confirmation.

### 5. Idempotency
The role is idempotent - running it multiple times produces the same result without unintended side effects.

## OS-Specific Behavior

### Ubuntu/Debian
- Uses `apt` for package management
- Cache location: `/var/cache/apt/archives/`
- All cleanup operations supported

### CentOS/RHEL 7
- Uses `yum` for package management
- Cache location: `/var/cache/yum/`
- All cleanup operations supported

### CentOS/RHEL 8+
- Uses `dnf` for package management
- Cache location: `/var/cache/dnf/`
- All cleanup operations supported

## Troubleshooting

### Issue: "Permission denied" errors

**Cause**: Insufficient privileges to delete files.

**Solution**: Ensure the role runs with `become: true`:
```yaml
- role: filesystem_cleanup
  become: true
```

### Issue: Cleanup not freeing expected space

**Possible causes:**
1. Dry-run mode is enabled
2. Files are newer than age thresholds
3. Files match exclude patterns

**Solution:**
- Verify `cleanup_dry_run: false`
- Check age thresholds in configuration
- Review exclude patterns

### Issue: "journalctl: command not found"

**Cause**: systemd is not available on the system.

**Solution**: Disable journal cleanup:
```yaml
cleanup_journals_enabled: false
```

### Issue: Package cache cleanup fails

**Cause**: Package manager lock or network issues.

**Solution:**
- Ensure no other package operations are running
- Check network connectivity for autoremove operations
- Review package manager logs

## Best Practices

1. **Always Test First**: Run in dry-run mode before actual cleanup
2. **Start Conservative**: Use default age thresholds initially
3. **Monitor Results**: Review cleanup reports and adjust as needed
4. **Schedule Regularly**: Run cleanup monthly or when disk space is low
5. **Backup Important Data**: Ensure backups exist before first run
6. **Use with Verification**: Always run disk_verification role first
7. **Document Changes**: Keep track of what was cleaned and when

## Performance Considerations

- **Large /tmp directories**: May take several minutes to scan
- **Many log files**: Compression can be CPU-intensive
- **Package cache**: Network required for autoremove operations
- **Journal vacuum**: Generally fast, but depends on journal size

## Integration Examples

### With Monitoring

```yaml
---
- name: Cleanup and alert
  hosts: all
  become: true
  
  tasks:
    - include_role:
        name: filesystem_cleanup
      vars:
        cleanup_dry_run: false
    
    - name: Send notification
      debug:
        msg: "Cleanup freed {{ filesystem_cleanup_results.total_space_freed_mb }} MB"
```

### Conditional Cleanup

```yaml
---
- name: Cleanup only if needed
  hosts: all
  become: true
  
  tasks:
    - include_role:
        name: disk_verification
    
    - include_role:
        name: filesystem_cleanup
      vars:
        cleanup_dry_run: false
      when: 
        - disk_space_status in ['warning', 'critical']
```

## License

[Specify your license]

## Author

[Your name/organization]