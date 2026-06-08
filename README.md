# Ansible Disk Space Management

Automated disk space verification and cleanup for Linux systems using Ansible. This project provides safe, idempotent playbooks to monitor disk usage and clean up temporary files, old logs, package caches, and systemd journals.

## Features

- ✅ **Safe by Default**: Dry-run mode enabled by default
- 🔍 **Verification First**: Check disk space before cleanup
- 🎯 **Targeted Cleanup**: Clean /tmp, /var/log, package cache, and journals
- 🐧 **Multi-OS Support**: Ubuntu, Debian, CentOS, RHEL
- 📊 **Detailed Reporting**: Comprehensive before/after statistics
- 🔧 **Highly Configurable**: Extensive variables for customization
- ♻️ **Idempotent**: Safe to run multiple times
- 🏷️ **Tag Support**: Run specific cleanup operations

## Requirements

- **Ansible**: 2.9 or higher
- **Python**: 3.6 or higher on control node
- **Target Systems**: Ubuntu 18.04+, Debian 10+, CentOS 7+, RHEL 7+
- **Privileges**: sudo/root access on target systems
- **Collections**: 
  - `ansible.builtin` (included with Ansible)
  - `community.general` (for archive module)

### Installing Required Collections

```bash
ansible-galaxy collection install community.general
```

## Quick Start

### 1. Clone or Download

```bash
git clone <repository-url>
cd ansible_disk_space
```

### 2. Configure Inventory

Edit [`inventory/hosts.yml`](inventory/hosts.yml:1) with your target hosts:

```yaml
all:
  children:
    production:
      hosts:
        server01.example.com:
          ansible_host: 192.168.1.10
```

### 3. Verify Disk Space

```bash
# Check disk space on all hosts
ansible-playbook playbooks/verify_disk_space.yml

# Check specific hosts
ansible-playbook playbooks/verify_disk_space.yml -l webservers
```

### 4. Run Cleanup (Dry-Run)

```bash
# Dry-run mode (default - shows what would be cleaned)
ansible-playbook playbooks/remediate_disk_space.yml
```

### 5. Run Actual Cleanup

```bash
# CAUTION: This will delete files!
ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=false"
```

## Project Structure

```
ansible_disk_space/
├── README.md                          # This file
├── ansible.cfg                        # Ansible configuration
├── inventory/
│   ├── hosts.yml                      # Inventory file
│   └── group_vars/
│       └── all.yml                    # Global variables
├── playbooks/
│   ├── verify_disk_space.yml          # Verification playbook
│   └── remediate_disk_space.yml       # Cleanup playbook
├── roles/
│   ├── disk_verification/             # Disk space verification role
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   └── README.md
│   └── filesystem_cleanup/            # Cleanup role
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── cleanup_tmp.yml
│       │   ├── cleanup_logs.yml
│       │   ├── cleanup_package_cache.yml
│       │   └── cleanup_journals.yml
│       ├── defaults/
│       │   └── main.yml
│       └── README.md
└── tests/
    └── test_playbook.yml              # Test playbook
```

## Configuration

### Global Variables

Edit [`inventory/group_vars/all.yml`](inventory/group_vars/all.yml:1) to customize behavior:

```yaml
# Disk space thresholds
disk_space_warning_threshold: 80      # Warning at 80% usage
disk_space_critical_threshold: 90     # Critical at 90% usage

# Cleanup behavior (DRY-RUN by default for safety!)
cleanup_dry_run: true                 # Set to false for actual cleanup

# Enable/disable cleanup operations
cleanup_tmp_enabled: true
cleanup_logs_enabled: true
cleanup_package_cache_enabled: true
cleanup_journals_enabled: true

# Age thresholds
tmp_file_age_days: 7                  # Delete /tmp files older than 7 days
log_file_age_days: 30                 # Delete logs older than 30 days
log_compress_age_days: 7              # Compress logs older than 7 days

# Journal settings
journal_max_size: "500M"              # Maximum journal size
journal_max_time: "30d"               # Maximum journal age
```

### Runtime Overrides

Override variables at runtime using `-e`:

```bash
# Change thresholds
ansible-playbook playbooks/verify_disk_space.yml \
  -e "disk_space_warning_threshold=85" \
  -e "disk_space_critical_threshold=95"

# Disable dry-run for actual cleanup
ansible-playbook playbooks/remediate_disk_space.yml \
  -e "cleanup_dry_run=false"

# Disable specific cleanup operations
ansible-playbook playbooks/remediate_disk_space.yml \
  -e "cleanup_journals_enabled=false"

# Adjust age thresholds
ansible-playbook playbooks/remediate_disk_space.yml \
  -e "tmp_file_age_days=14" \
  -e "log_file_age_days=60"
```

## Usage Examples

### Verification Playbook

```bash
# Basic verification
ansible-playbook playbooks/verify_disk_space.yml

# Check specific host group
ansible-playbook playbooks/verify_disk_space.yml -l webservers

# Check single host
ansible-playbook playbooks/verify_disk_space.yml -l web01.example.com

# Fail playbook if critical threshold exceeded
ansible-playbook playbooks/verify_disk_space.yml -e "fail_on_critical=true"

# Use custom thresholds
ansible-playbook playbooks/verify_disk_space.yml \
  -e "disk_space_warning_threshold=75" \
  -e "disk_space_critical_threshold=85"
```

### Remediation Playbook

```bash
# Dry-run (default - safe, shows what would be cleaned)
ansible-playbook playbooks/remediate_disk_space.yml

# Dry-run on specific hosts
ansible-playbook playbooks/remediate_disk_space.yml -l webservers

# Actual cleanup (CAUTION!)
ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=false"

# Cleanup specific hosts only
ansible-playbook playbooks/remediate_disk_space.yml \
  -l web01.example.com \
  -e "cleanup_dry_run=false"

# Run specific cleanup operations using tags
ansible-playbook playbooks/remediate_disk_space.yml --tags tmp
ansible-playbook playbooks/remediate_disk_space.yml --tags logs,cache
ansible-playbook playbooks/remediate_disk_space.yml --tags journals

# Skip verification (not recommended)
ansible-playbook playbooks/remediate_disk_space.yml --skip-tags verification

# Disable specific cleanup operations
ansible-playbook playbooks/remediate_disk_space.yml \
  -e "cleanup_journals_enabled=false" \
  -e "cleanup_dry_run=false"
```

### Using Ansible Check Mode

```bash
# Ansible check mode (no changes, even with cleanup_dry_run=false)
ansible-playbook playbooks/remediate_disk_space.yml --check
```

## Cleanup Targets

### 1. /tmp Directory
- **What**: Removes files older than `tmp_file_age_days` (default: 7 days)
- **Excludes**: System-critical files (.X11-unix, .ICE-unix, systemd-*, etc.)
- **Safety**: Age-based deletion, exclude patterns

### 2. /var/log Directory
- **What**: 
  - Compresses logs older than `log_compress_age_days` (default: 7 days)
  - Deletes logs older than `log_file_age_days` (default: 30 days)
- **Excludes**: Already compressed files, system logs (wtmp, btmp, lastlog)
- **Safety**: Age-based operations, exclude patterns

### 3. Package Cache
- **Ubuntu/Debian**: 
  - `apt-get clean` (removes all cached packages)
  - `apt-get autoclean` (removes obsolete packages)
  - `apt-get autoremove` (removes unused dependencies)
- **CentOS/RHEL 7**: 
  - `yum clean all`
  - `yum autoremove`
- **CentOS/RHEL 8+**: 
  - `dnf clean all`
  - `dnf autoremove`
- **Safety**: Package caches can be rebuilt, autoremove only removes unused packages

### 4. Systemd Journals
- **What**: Trims journal logs using `journalctl --vacuum-size` and `--vacuum-time`
- **Limits**: 
  - Max size: `journal_max_size` (default: 500M)
  - Max age: `journal_max_time` (default: 30d)
- **Safety**: Journals are preserved within limits, integrity check optional

## Safety Features

### 1. Dry-Run by Default
- `cleanup_dry_run: true` in defaults
- Must explicitly set to `false` for actual cleanup
- Shows what would be deleted without making changes

### 2. File Age Protection
- Never deletes files newer than configured age
- Default: 7 days for /tmp, 30 days for logs
- Configurable per cleanup target

### 3. Exclude Patterns
- System-critical files excluded by default
- Configurable exclusion lists
- Pattern matching for flexibility

### 4. Pre/Post Verification
- Disk space checked before cleanup
- Re-verified after cleanup
- Before/after comparison in reports

### 5. Confirmation Prompt
- Interactive prompt when running actual cleanup
- Can be bypassed with `--extra-vars` or in automation

### 6. Idempotency
- Safe to run multiple times
- No unintended side effects
- Consistent results

## Tags

Use tags to run specific parts of playbooks:

### Verification Playbook Tags
- `verification`: Run verification tasks
- `reporting`: Display reports only

### Remediation Playbook Tags
- `verification`: Pre/post verification
- `cleanup`: All cleanup operations
- `tmp`: /tmp cleanup only
- `logs`: /var/log cleanup only
- `cache`: Package cache cleanup only
- `journals`: Systemd journal cleanup only
- `reporting`: Display reports

### Examples

```bash
# Run only /tmp cleanup
ansible-playbook playbooks/remediate_disk_space.yml --tags tmp

# Run logs and cache cleanup
ansible-playbook playbooks/remediate_disk_space.yml --tags logs,cache

# Skip verification (not recommended)
ansible-playbook playbooks/remediate_disk_space.yml --skip-tags verification

# Run only reporting
ansible-playbook playbooks/verify_disk_space.yml --tags reporting
```

## Testing

### Syntax Check

```bash
# Check playbook syntax
ansible-playbook playbooks/verify_disk_space.yml --syntax-check
ansible-playbook playbooks/remediate_disk_space.yml --syntax-check
```

### Lint Check

```bash
# Install ansible-lint
pip install ansible-lint

# Lint playbooks
ansible-lint playbooks/*.yml

# Lint roles
ansible-lint roles/*/tasks/*.yml
```

### Dry-Run Testing

```bash
# Test with Ansible check mode
ansible-playbook playbooks/remediate_disk_space.yml --check

# Test with dry-run mode (default)
ansible-playbook playbooks/remediate_disk_space.yml

# Test on a single host first
ansible-playbook playbooks/remediate_disk_space.yml -l test-server
```

### Test Playbook

Run the included test playbook:

```bash
ansible-playbook tests/test_playbook.yml
```

## Troubleshooting

### Issue: "Permission denied" errors

**Solution**: Ensure the Ansible user has sudo privileges:

```bash
# Test sudo access
ansible all -m command -a "sudo whoami" --become

# Add user to sudoers if needed
echo "ansible ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ansible
```

### Issue: "Module not found" errors

**Solution**: Install required collections:

```bash
ansible-galaxy collection install community.general
```

### Issue: Cleanup not freeing expected space

**Possible causes**:
1. Files are newer than age thresholds
2. Files match exclude patterns
3. Dry-run mode is enabled
4. Large files in other locations

**Solution**: 
- Review age thresholds in configuration
- Check exclude patterns
- Verify `cleanup_dry_run=false` for actual cleanup
- Investigate other directories: `du -h --max-depth=1 / | sort -hr`

### Issue: "No hosts matched" error

**Solution**: Check inventory configuration:

```bash
# List all hosts
ansible all --list-hosts

# Test connectivity
ansible all -m ping
```

### Issue: Playbook hangs at confirmation prompt

**Solution**: 
- Press ENTER to continue or CTRL+C then 'A' to abort
- For automation, use `--extra-vars` to bypass prompt
- Or set `ANSIBLE_TIMEOUT` environment variable

## Best Practices

### 1. Test First
- Always run verification before cleanup
- Use dry-run mode first
- Test on non-production systems

### 2. Start Conservative
- Use default age thresholds initially
- Monitor results before adjusting
- Gradually increase cleanup frequency

### 3. Schedule Regular Runs
- Run verification weekly
- Run cleanup monthly (or as needed)
- Use cron or systemd timers

### 4. Monitor Results
- Review cleanup reports
- Track space freed over time
- Adjust thresholds based on patterns

### 5. Backup Important Data
- Backup before first run
- Document what was cleaned
- Keep cleanup logs

### 6. Use Version Control
- Track inventory changes
- Document variable modifications
- Review changes before applying

## Scheduling with Cron

Example crontab entries:

```bash
# Verify disk space daily at 2 AM
0 2 * * * cd /path/to/ansible_disk_space && ansible-playbook playbooks/verify_disk_space.yml

# Cleanup monthly on the 1st at 3 AM
0 3 1 * * cd /path/to/ansible_disk_space && ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=false"
```

## OS-Specific Notes

### Ubuntu/Debian
- Package manager: `apt`
- Cache location: `/var/cache/apt/archives/`
- Supports all cleanup operations

### CentOS/RHEL 7
- Package manager: `yum`
- Cache location: `/var/cache/yum/`
- Supports all cleanup operations

### CentOS/RHEL 8+
- Package manager: `dnf`
- Cache location: `/var/cache/dnf/`
- Supports all cleanup operations

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Test your changes thoroughly
4. Submit a pull request

## License

[Specify your license here]

## Support

For issues, questions, or contributions:
- Open an issue in the repository
- Review existing documentation
- Check troubleshooting section

## Changelog

### Version 1.0.0
- Initial release
- Disk verification role
- Filesystem cleanup role
- Support for Ubuntu, Debian, CentOS, RHEL
- Dry-run mode by default
- Comprehensive reporting

## Acknowledgments

Built with Ansible best practices and community feedback.