# Quick Start Guide

Get started with Ansible Disk Space Management in 5 minutes!

## Prerequisites

- Ansible 2.9+ installed
- SSH access to target systems
- sudo privileges on target systems

## Installation

### 1. Install Required Collections

```bash
ansible-galaxy collection install community.general
```

### 2. Configure Inventory

Edit [`inventory/hosts.yml`](inventory/hosts.yml:1) with your servers:

```yaml
all:
  children:
    production:
      hosts:
        server01.example.com:
          ansible_host: 192.168.1.10
```

### 3. Test Connectivity

```bash
ansible all -m ping
```

## Basic Usage

### Check Disk Space

```bash
# Check all hosts
ansible-playbook playbooks/verify_disk_space.yml

# Check specific group
ansible-playbook playbooks/verify_disk_space.yml -l webservers
```

### Clean Up Disk Space

```bash
# Step 1: Dry-run (safe - shows what would be cleaned)
ansible-playbook playbooks/remediate_disk_space.yml

# Step 2: Review the output, then run actual cleanup
ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=false"
```

## Common Commands

### Verification

```bash
# Basic check
ansible-playbook playbooks/verify_disk_space.yml

# Check with custom thresholds
ansible-playbook playbooks/verify_disk_space.yml \
  -e "disk_space_warning_threshold=85" \
  -e "disk_space_critical_threshold=95"

# Fail if critical
ansible-playbook playbooks/verify_disk_space.yml -e "fail_on_critical=true"
```

### Cleanup

```bash
# Dry-run (default)
ansible-playbook playbooks/remediate_disk_space.yml

# Actual cleanup
ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=false"

# Clean specific targets only
ansible-playbook playbooks/remediate_disk_space.yml --tags tmp
ansible-playbook playbooks/remediate_disk_space.yml --tags logs,cache

# Target specific hosts
ansible-playbook playbooks/remediate_disk_space.yml \
  -l server01.example.com \
  -e "cleanup_dry_run=false"
```

## Configuration

### Quick Configuration Changes

Edit [`inventory/group_vars/all.yml`](inventory/group_vars/all.yml:1):

```yaml
# Adjust thresholds
disk_space_warning_threshold: 80
disk_space_critical_threshold: 90

# Adjust age thresholds
tmp_file_age_days: 7        # Delete /tmp files older than 7 days
log_file_age_days: 30       # Delete logs older than 30 days

# Disable specific cleanup operations
cleanup_journals_enabled: false
```

### Runtime Overrides

```bash
# Override any variable at runtime
ansible-playbook playbooks/remediate_disk_space.yml \
  -e "tmp_file_age_days=14" \
  -e "log_file_age_days=60" \
  -e "cleanup_dry_run=false"
```

## Safety Features

✅ **Dry-run by default** - Must explicitly enable actual cleanup  
✅ **File age protection** - Never deletes recent files  
✅ **Exclude patterns** - System-critical files protected  
✅ **Confirmation prompt** - Interactive confirmation before cleanup  
✅ **Pre/post verification** - Disk space checked before and after  

## Typical Workflow

1. **Verify disk space**:
   ```bash
   ansible-playbook playbooks/verify_disk_space.yml
   ```

2. **If cleanup needed, run dry-run**:
   ```bash
   ansible-playbook playbooks/remediate_disk_space.yml
   ```

3. **Review output, then run actual cleanup**:
   ```bash
   ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=false"
   ```

4. **Verify results**:
   ```bash
   ansible-playbook playbooks/verify_disk_space.yml
   ```

## Troubleshooting

### "Permission denied" errors
```bash
# Ensure sudo access
ansible all -m command -a "sudo whoami" --become
```

### "No hosts matched"
```bash
# Check inventory
ansible all --list-hosts
```

### Cleanup not freeing space
- Check if `cleanup_dry_run=false` is set
- Verify files are older than age thresholds
- Review exclude patterns in configuration

## Next Steps

- Read the full [README.md](README.md:1) for detailed documentation
- Review [roles/disk_verification/README.md](roles/disk_verification/README.md:1) for verification options
- Review [roles/filesystem_cleanup/README.md](roles/filesystem_cleanup/README.md:1) for cleanup options
- Run [tests/test_playbook.yml](tests/test_playbook.yml:1) to validate functionality

## Getting Help

- Check the [README.md](README.md:1) troubleshooting section
- Review [CONTRIBUTING.md](CONTRIBUTING.md:1) for development guidelines
- Open an issue for bugs or feature requests

## Quick Reference Card

| Task | Command |
|------|---------|
| Check disk space | `ansible-playbook playbooks/verify_disk_space.yml` |
| Dry-run cleanup | `ansible-playbook playbooks/remediate_disk_space.yml` |
| Actual cleanup | `ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=false"` |
| Clean /tmp only | `ansible-playbook playbooks/remediate_disk_space.yml --tags tmp` |
| Syntax check | `ansible-playbook playbooks/*.yml --syntax-check` |
| Run tests | `ansible-playbook tests/test_playbook.yml` |

---

**Remember**: Always test in dry-run mode first! 🛡️