# Ansible Disk Space Management - Implementation Plan

## Project Overview

This project provides automated disk space verification and cleanup for Linux systems (Ubuntu, CentOS, RHEL) using Ansible. It follows a two-phase approach: verification first, then remediation if needed.

## Implementation Summary

| Phase | Name | Duration | Key Deliverables | Priority |
|-------|------|----------|------------------|----------|
| **1** | Project Foundation | 30 min | Directory structure, ansible.cfg, inventory files | High |
| **2** | Disk Verification Role | 45 min | disk_verification role with monitoring and reporting | High |
| **3** | Filesystem Cleanup Role | 90 min | filesystem_cleanup role with 4 cleanup targets | High |
| **4** | Playbooks | 30 min | verify_disk_space.yml and remediate_disk_space.yml | High |
| **5** | Documentation | 45 min | Main README, role READMEs, inline docs | Medium |
| **6** | Testing & Validation | 60 min | Test playbooks, validation checklist, molecule tests | Medium |
| | **Total** | **5 hours** | Production-ready Ansible disk space management solution | |

### Quick Reference

- **Total Implementation Time**: 5 hours
- **Roles**: 2 (disk_verification, filesystem_cleanup)
- **Playbooks**: 2 (verify, remediate)
- **Cleanup Targets**: 4 (/tmp, /var/log, package cache, systemd journals)
- **Supported OS**: Ubuntu, CentOS, RHEL
- **Safety Features**: Dry-run by default, file age protection, exclude patterns

## Project Structure

```
ansible_disk_space/
├── README.md
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── all.yml
├── playbooks/
│   ├── verify_disk_space.yml
│   └── remediate_disk_space.yml
├── roles/
│   ├── disk_verification/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   └── README.md
│   └── filesystem_cleanup/
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── cleanup_tmp.yml
│       │   ├── cleanup_logs.yml
│       │   ├── cleanup_package_cache.yml
│       │   └── cleanup_journals.yml
│       ├── defaults/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       └── README.md
└── tests/
    ├── test_playbook.yml
    └── molecule/
        └── default/
            ├── molecule.yml
            ├── converge.yml
            └── verify.yml
```

## Implementation Phases

### Phase 1: Project Foundation (30 minutes)

**Objective**: Set up the basic project structure and configuration files.

**Tasks**:
1. Create directory structure
2. Create [`ansible.cfg`](ansible.cfg:1) with best practices:
   - Inventory location
   - Host key checking disabled for testing
   - Retry files disabled
   - Callback plugins for better output
   - Fact caching configuration

3. Create inventory structure:
   - [`inventory/hosts.yml`](inventory/hosts.yml:1) with example hosts
   - [`inventory/group_vars/all.yml`](inventory/group_vars/all.yml:1) with global variables

**Deliverables**:
- Basic project skeleton
- Configuration files
- Inventory examples

---

### Phase 2: Disk Verification Role (45 minutes)

**Objective**: Implement the [`disk_verification`](roles/disk_verification/tasks/main.yml:1) role to check disk space usage.

**Role Purpose**: 
- Check disk space usage on target filesystems
- Report current usage statistics
- Determine if cleanup is needed based on thresholds
- Generate actionable reports

**Implementation Details**:

#### [`roles/disk_verification/defaults/main.yml`](roles/disk_verification/defaults/main.yml:1)
```yaml
# Threshold for warning (percentage)
disk_space_warning_threshold: 80

# Threshold for critical action (percentage)
disk_space_critical_threshold: 90

# Filesystems to monitor
monitored_filesystems:
  - /
  - /tmp
  - /var
  - /home

# Whether to fail on critical threshold
fail_on_critical: false
```

#### [`roles/disk_verification/tasks/main.yml`](roles/disk_verification/tasks/main.yml:1)
**Key Tasks**:
1. Gather disk space facts using [`ansible.builtin.setup`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html)
2. Parse filesystem usage with [`ansible.builtin.df`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html) command
3. Calculate usage percentages
4. Generate report with:
   - Current usage per filesystem
   - Available space
   - Warning/critical status
5. Set facts for downstream roles:
   - `disk_space_status`: "ok", "warning", or "critical"
   - `filesystems_needing_cleanup`: list of filesystems above threshold
6. Display summary using [`ansible.builtin.debug`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)

**Best Practices**:
- Use [`ansible.builtin.set_fact`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/set_fact_module.html) to store results
- Tag tasks appropriately: `verification`, `reporting`
- Use [`ansible.builtin.assert`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html) for validation
- Implement proper error handling

**Deliverables**:
- Complete disk_verification role
- Role README with usage examples
- Variable documentation

---

### Phase 3: Filesystem Cleanup Role (90 minutes)

**Objective**: Implement the [`filesystem_cleanup`](roles/filesystem_cleanup/tasks/main.yml:1) role with safe, idempotent cleanup operations.

**Role Purpose**:
- Clean temporary files from /tmp
- Rotate and compress logs in /var/log
- Clear package manager caches
- Trim systemd journal logs
- Support dry-run mode for safety

**Implementation Details**:

#### [`roles/filesystem_cleanup/defaults/main.yml`](roles/filesystem_cleanup/defaults/main.yml:1)
```yaml
# Dry run mode - only report what would be deleted
cleanup_dry_run: true

# Cleanup targets
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

# /var/log cleanup settings
log_file_age_days: 30
log_compress_age_days: 7
log_exclude_patterns:
  - "*.gz"
  - "lastlog"
  - "wtmp"

# Package cache settings
package_cache_keep_latest: 1

# Journal settings
journal_max_size: "500M"
journal_max_time: "30d"
```

#### Task Files Structure:

**[`roles/filesystem_cleanup/tasks/main.yml`](roles/filesystem_cleanup/tasks/main.yml:1)**:
- Include OS-specific variables
- Validate prerequisites
- Include cleanup task files based on enabled flags
- Generate cleanup summary report

**[`roles/filesystem_cleanup/tasks/cleanup_tmp.yml`](roles/filesystem_cleanup/tasks/cleanup_tmp.yml:1)**:
1. Find old files in /tmp using [`ansible.builtin.find`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/find_module.html)
2. Calculate space to be freed
3. Remove files (if not dry-run) using [`ansible.builtin.file`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
4. Report results

**[`roles/filesystem_cleanup/tasks/cleanup_logs.yml`](roles/filesystem_cleanup/tasks/cleanup_logs.yml:1)**:
1. Find old log files
2. Compress logs older than threshold using [`community.general.archive`](https://docs.ansible.com/ansible/latest/collections/community/general/archive_module.html)
3. Remove very old logs
4. Report space saved

**[`roles/filesystem_cleanup/tasks/cleanup_package_cache.yml`](roles/filesystem_cleanup/tasks/cleanup_package_cache.yml:1)**:
1. Detect OS family (Debian/RedHat)
2. For Ubuntu/Debian:
   - Run `apt-get clean` using [`ansible.builtin.apt`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
   - Run `apt-get autoremove`
3. For CentOS/RHEL:
   - Run `yum clean all` or `dnf clean all` using [`ansible.builtin.yum`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html)
4. Report cache size before/after

**[`roles/filesystem_cleanup/tasks/cleanup_journals.yml`](roles/filesystem_cleanup/tasks/cleanup_journals.yml:1)**:
1. Check current journal size using [`ansible.builtin.command`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html): `journalctl --disk-usage`
2. Vacuum journals by size: `journalctl --vacuum-size={{ journal_max_size }}`
3. Vacuum journals by time: `journalctl --vacuum-time={{ journal_max_time }}`
4. Report space freed

**Safety Considerations**:
- **Dry-run by default**: `cleanup_dry_run: true`
- **File age checks**: Never delete recent files
- **Exclude patterns**: Protect system-critical files
- **Backup recommendations**: Document in README
- **Idempotency**: All tasks must be safely re-runnable
- **Check mode support**: Use `check_mode: yes` where appropriate
- **Validation**: Verify paths exist before operations
- **Error handling**: Use `ignore_errors` with proper logging

**Best Practices**:
- Use [`ansible.builtin.stat`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/stat_module.html) to check before operations
- Implement proper tagging: `cleanup`, `tmp`, `logs`, `cache`, `journals`
- Use [`ansible.builtin.block`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/block_module.html) with rescue for error handling
- Register results for reporting
- Use `changed_when` to control change reporting

**Deliverables**:
- Complete filesystem_cleanup role with all task files
- Role README with detailed usage
- Variable documentation
- Safety guidelines

---

### Phase 4: Playbooks (30 minutes)

**Objective**: Create the main playbooks that orchestrate the roles.

#### [`playbooks/verify_disk_space.yml`](playbooks/verify_disk_space.yml:1)

```yaml
---
- name: Verify Disk Space Usage
  hosts: all
  become: true
  gather_facts: true
  
  roles:
    - role: disk_verification
      tags: ['verification']
  
  post_tasks:
    - name: Display verification summary
      ansible.builtin.debug:
        msg: |
          Disk Space Verification Complete
          Status: {{ disk_space_status }}
          Filesystems needing attention: {{ filesystems_needing_cleanup | default([]) | length }}
      tags: ['reporting']
```

**Features**:
- Read-only operations
- No system changes
- Generates actionable reports
- Can run frequently without risk

#### [`playbooks/remediate_disk_space.yml`](playbooks/remediate_disk_space.yml:1)

```yaml
---
- name: Remediate Disk Space Issues
  hosts: all
  become: true
  gather_facts: true
  
  pre_tasks:
    - name: Verify disk space first
      ansible.builtin.include_role:
        name: disk_verification
      tags: ['always']
    
    - name: Confirm remediation needed
      ansible.builtin.assert:
        that:
          - disk_space_status in ['warning', 'critical']
        fail_msg: "Disk space is OK, no remediation needed"
        success_msg: "Proceeding with cleanup"
      tags: ['always']
  
  roles:
    - role: filesystem_cleanup
      tags: ['cleanup']
  
  post_tasks:
    - name: Re-verify disk space
      ansible.builtin.include_role:
        name: disk_verification
      tags: ['verification']
    
    - name: Display remediation summary
      ansible.builtin.debug:
        msg: |
          Disk Space Remediation Complete
          Previous Status: {{ disk_space_status_before }}
          Current Status: {{ disk_space_status }}
          Space Freed: {{ total_space_freed | default('N/A') }}
      tags: ['reporting']
```

**Features**:
- Verification before cleanup
- Conditional execution based on thresholds
- Dry-run support via variables
- Post-cleanup verification
- Comprehensive reporting

**Best Practices**:
- Use `pre_tasks` for validation
- Use `post_tasks` for verification
- Implement proper tagging strategy
- Document variable overrides
- Include usage examples in comments

**Deliverables**:
- Both playbooks with inline documentation
- Example execution commands
- Variable override examples

---

### Phase 5: Documentation (45 minutes)

**Objective**: Create comprehensive documentation for users and maintainers.

#### Main [`README.md`](README.md:1)

**Sections**:
1. **Overview**: Project purpose and features
2. **Requirements**: 
   - Ansible version (2.9+)
   - Python version
   - Required collections
   - Target OS support matrix
3. **Quick Start**:
   - Installation steps
   - Basic usage examples
   - Common scenarios
4. **Project Structure**: Directory layout explanation
5. **Configuration**:
   - Inventory setup
   - Variable customization
   - Threshold configuration
6. **Usage**:
   - Verification workflow
   - Remediation workflow
   - Dry-run mode
   - Production execution
7. **Safety Features**:
   - Dry-run by default
   - File age protection
   - Exclude patterns
   - Backup recommendations
8. **Examples**:
   - Check disk space: `ansible-playbook playbooks/verify_disk_space.yml`
   - Dry-run cleanup: `ansible-playbook playbooks/remediate_disk_space.yml`
   - Actual cleanup: `ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=false"`
   - Specific hosts: `ansible-playbook playbooks/verify_disk_space.yml -l webservers`
   - Specific cleanup: `ansible-playbook playbooks/remediate_disk_space.yml --tags logs`
9. **Testing**: How to test the playbooks
10. **Troubleshooting**: Common issues and solutions
11. **Contributing**: Guidelines for contributions
12. **License**: Project license

#### Role READMEs

**[`roles/disk_verification/README.md`](roles/disk_verification/README.md:1)**:
- Role purpose
- Variables reference
- Dependencies
- Example playbook usage
- Tags available

**[`roles/filesystem_cleanup/README.md`](roles/filesystem_cleanup/README.md:1)**:
- Role purpose
- Cleanup targets explanation
- Variables reference with defaults
- Safety considerations
- OS-specific notes
- Dependencies
- Example playbook usage
- Tags available

**Deliverables**:
- Main README.md
- Role-specific READMEs
- Inline code documentation

---

### Phase 6: Testing & Validation (60 minutes)

**Objective**: Implement testing strategy and validation playbooks.

#### Testing Approach

**1. Syntax Validation**:
```bash
# Check playbook syntax
ansible-playbook playbooks/verify_disk_space.yml --syntax-check
ansible-playbook playbooks/remediate_disk_space.yml --syntax-check

# Lint playbooks
ansible-lint playbooks/*.yml
ansible-lint roles/*/tasks/*.yml
```

**2. Dry-Run Testing**:
```bash
# Check mode (no changes)
ansible-playbook playbooks/remediate_disk_space.yml --check

# Dry-run mode (reports what would be deleted)
ansible-playbook playbooks/remediate_disk_space.yml -e "cleanup_dry_run=true"
```

**3. Integration Testing**:

Create [`tests/test_playbook.yml`](tests/test_playbook.yml:1):
```yaml
---
- name: Test Disk Space Management
  hosts: test_servers
  become: true
  
  tasks:
    - name: Create test files in /tmp
      ansible.builtin.file:
        path: "/tmp/test_file_{{ item }}"
        state: touch
        modification_time: "{{ lookup('pipe', 'date -d \"10 days ago\" +%Y%m%d%H%M.%S') }}"
      loop: "{{ range(1, 6) | list }}"
    
    - name: Run verification
      ansible.builtin.include_role:
        name: disk_verification
    
    - name: Run cleanup in dry-run
      ansible.builtin.include_role:
        name: filesystem_cleanup
      vars:
        cleanup_dry_run: true
    
    - name: Verify test files still exist
      ansible.builtin.stat:
        path: "/tmp/test_file_{{ item }}"
      register: test_files
      loop: "{{ range(1, 6) | list }}"
    
    - name: Assert files not deleted in dry-run
      ansible.builtin.assert:
        that:
          - test_files.results | selectattr('stat.exists') | list | length == 5
    
    - name: Run actual cleanup
      ansible.builtin.include_role:
        name: filesystem_cleanup
      vars:
        cleanup_dry_run: false
    
    - name: Verify test files removed
      ansible.builtin.stat:
        path: "/tmp/test_file_{{ item }}"
      register: test_files_after
      loop: "{{ range(1, 6) | list }}"
    
    - name: Assert files deleted
      ansible.builtin.assert:
        that:
          - test_files_after.results | selectattr('stat.exists') | list | length == 0
```

**4. Molecule Testing** (Optional but recommended):

Create [`tests/molecule/default/molecule.yml`](tests/molecule/default/molecule.yml:1):
```yaml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu-instance
    image: ubuntu:22.04
    pre_build_image: true
  - name: centos-instance
    image: centos:8
    pre_build_image: true
provisioner:
  name: ansible
  playbooks:
    converge: converge.yml
    verify: verify.yml
verifier:
  name: ansible
```

**5. Manual Testing Checklist**:
- [ ] Verify playbook runs on Ubuntu 20.04/22.04
- [ ] Verify playbook runs on CentOS 7/8
- [ ] Verify playbook runs on RHEL 8/9
- [ ] Test dry-run mode doesn't delete files
- [ ] Test actual cleanup removes old files
- [ ] Test threshold-based execution
- [ ] Test tag-based selective cleanup
- [ ] Test with different variable overrides
- [ ] Verify idempotency (run twice, second run shows no changes)
- [ ] Test error handling (invalid paths, permissions)

**Validation Criteria**:
- All playbooks pass syntax check
- Ansible-lint shows no errors
- Dry-run mode reports correctly without changes
- Actual cleanup removes only intended files
- Playbooks are idempotent
- Works across all supported OS versions
- Proper error messages for failures
- Documentation matches implementation

**Deliverables**:
- Test playbook
- Molecule configuration (optional)
- Testing documentation
- Validation checklist

---

## Implementation Order

Follow this sequence for optimal development flow:

1. **Foundation** (Phase 1): 30 minutes
   - Create directory structure
   - Set up ansible.cfg
   - Create inventory files

2. **Verification** (Phase 2): 45 minutes
   - Implement disk_verification role
   - Test verification independently

3. **Cleanup** (Phase 3): 90 minutes
   - Implement filesystem_cleanup role
   - Implement each cleanup task file
   - Test in dry-run mode extensively

4. **Orchestration** (Phase 4): 30 minutes
   - Create verify_disk_space.yml playbook
   - Create remediate_disk_space.yml playbook
   - Test end-to-end workflow

5. **Documentation** (Phase 5): 45 minutes
   - Write main README
   - Write role READMEs
   - Add inline documentation

6. **Testing** (Phase 6): 60 minutes
   - Create test playbooks
   - Run validation tests
   - Fix any issues found

**Total Estimated Time**: 5 hours

---

## Ansible Best Practices Applied

### 1. **Role Design**
- Single responsibility per role
- Clear separation of concerns
- Reusable and modular
- Well-documented defaults

### 2. **Idempotency**
- All tasks can be run multiple times safely
- Use `changed_when` to control change reporting
- Proper state management

### 3. **Variables**
- Sensible defaults in `defaults/main.yml`
- Clear variable naming conventions
- Documentation for all variables
- Support for overrides at multiple levels

### 4. **Error Handling**
- Use `block`/`rescue`/`always` for complex operations
- Proper `failed_when` conditions
- Meaningful error messages
- Graceful degradation where appropriate

### 5. **Security**
- Dry-run by default for destructive operations
- File age checks to prevent accidental deletion
- Exclude patterns for system files
- Privilege escalation only where needed

### 6. **Performance**
- Efficient fact gathering
- Conditional task execution
- Proper use of `when` clauses
- Minimal external command usage

### 7. **Maintainability**
- Clear task names
- Logical file organization
- Comprehensive documentation
- Consistent formatting

### 8. **Testing**
- Syntax validation
- Lint checking
- Dry-run testing
- Integration testing
- Multi-OS testing

---

## Safety Considerations

### Critical Safety Features

1. **Dry-Run by Default**
   - `cleanup_dry_run: true` in defaults
   - Must explicitly set to `false` for actual cleanup
   - Reports what would be deleted without making changes

2. **File Age Protection**
   - Never delete files newer than configured age
   - Default: 7 days for /tmp, 30 days for logs
   - Configurable per cleanup target

3. **Exclude Patterns**
   - System-critical files excluded by default
   - User-configurable exclusion lists
   - Pattern matching for flexibility

4. **Pre-Cleanup Verification**
   - Verify disk space status before cleanup
   - Only proceed if thresholds exceeded
   - Option to skip verification if needed

5. **Post-Cleanup Verification**
   - Re-check disk space after cleanup
   - Report space freed
   - Validate cleanup success

6. **Backup Recommendations**
   - Document backup procedures in README
   - Recommend testing in non-production first
   - Suggest snapshot/backup before first run

7. **Audit Trail**
   - Log all cleanup operations
   - Report files deleted
   - Track space freed per operation

### Risk Mitigation

**Low Risk Operations**:
- Disk space verification (read-only)
- Package cache cleanup (can be rebuilt)
- Journal trimming (logs preserved)

**Medium Risk Operations**:
- /tmp cleanup (temporary files)
- Old log deletion (if logs not needed)

**High Risk Operations**:
- None in this implementation (by design)

**Mitigation Strategies**:
- Start with verification only
- Test in dry-run mode extensively
- Test on non-production systems first
- Use conservative age thresholds initially
- Implement gradual rollout
- Monitor results closely
- Have rollback plan (backups)

---

## Variable Reference

### Global Variables ([`inventory/group_vars/all.yml`](inventory/group_vars/all.yml:1))

```yaml
# Disk verification thresholds
disk_space_warning_threshold: 80
disk_space_critical_threshold: 90

# Cleanup behavior
cleanup_dry_run: true
cleanup_tmp_enabled: true
cleanup_logs_enabled: true
cleanup_package_cache_enabled: true
cleanup_journals_enabled: true

# Age thresholds
tmp_file_age_days: 7
log_file_age_days: 30
log_compress_age_days: 7

# Journal settings
journal_max_size: "500M"
journal_max_time: "30d"
```

### Runtime Overrides

```bash
# Disable dry-run for actual cleanup
-e "cleanup_dry_run=false"

# Adjust thresholds
-e "disk_space_critical_threshold=95"

# Disable specific cleanup
-e "cleanup_journals_enabled=false"

# Adjust age thresholds
-e "tmp_file_age_days=14"
```

---

## OS-Specific Considerations

### Ubuntu/Debian
- Package manager: `apt`
- Cache location: `/var/cache/apt/archives/`
- Log rotation: `logrotate` (usually pre-configured)
- Journal location: `/var/log/journal/`

### CentOS/RHEL 7
- Package manager: `yum`
- Cache location: `/var/cache/yum/`
- Log rotation: `logrotate`
- Journal location: `/var/log/journal/`

### CentOS/RHEL 8+
- Package manager: `dnf`
- Cache location: `/var/cache/dnf/`
- Log rotation: `logrotate`
- Journal location: `/var/log/journal/`

### Detection Strategy
Use [`ansible_os_family`](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html) and [`ansible_distribution_major_version`](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html) facts for OS-specific tasks.

---

## Success Criteria

The implementation will be considered successful when:

- [ ] All playbooks pass syntax validation
- [ ] Ansible-lint shows no errors or warnings
- [ ] Verification playbook runs successfully on all supported OS
- [ ] Remediation playbook works in dry-run mode
- [ ] Remediation playbook successfully cleans up space
- [ ] All cleanup operations are idempotent
- [ ] Dry-run mode doesn't make any changes
- [ ] File age protection works correctly
- [ ] Exclude patterns are respected
- [ ] Documentation is complete and accurate
- [ ] Test playbooks validate functionality
- [ ] README includes all necessary information
- [ ] Role READMEs document all variables and usage

---

## Future Enhancements

Consider these additions after initial implementation:

1. **Monitoring Integration**
   - Send metrics to monitoring systems
   - Alert on cleanup failures
   - Track space freed over time

2. **Advanced Cleanup Targets**
   - Docker images and containers
   - Old kernel versions
   - User home directories (with caution)
   - Application-specific caches

3. **Reporting**
   - HTML report generation
   - Email notifications
   - Slack/Teams integration

4. **Scheduling**
   - Cron job setup role
   - Systemd timer configuration
   - AWX/Tower job templates

5. **Recovery**
   - Backup before cleanup option
   - Restore capability
   - Cleanup history tracking

---

## Conclusion

This implementation plan provides a comprehensive, safe, and maintainable solution for automated disk space management using Ansible. By following the phased approach and adhering to best practices, you'll create a production-ready tool that can be safely deployed across your infrastructure.

The emphasis on safety (dry-run by default, file age protection, exclude patterns) ensures that the tool can be trusted in production environments, while the modular design allows for easy customization and extension.

**Next Steps**: 
1. Review and approve this plan
2. Begin implementation following the suggested order
3. Test thoroughly in non-production environment
4. Document any deviations or customizations
5. Deploy to production with monitoring