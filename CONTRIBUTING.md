# Contributing to Ansible Disk Space Management

Thank you for your interest in contributing to this project! This document provides guidelines and instructions for contributing.

## Code of Conduct

- Be respectful and inclusive
- Provide constructive feedback
- Focus on what is best for the community
- Show empathy towards other community members

## How to Contribute

### Reporting Bugs

If you find a bug, please create an issue with:

1. **Clear title**: Describe the issue concisely
2. **Description**: Detailed explanation of the problem
3. **Steps to reproduce**: How to recreate the issue
4. **Expected behavior**: What should happen
5. **Actual behavior**: What actually happens
6. **Environment details**:
   - Ansible version
   - Target OS and version
   - Python version
   - Relevant configuration

### Suggesting Enhancements

Enhancement suggestions are welcome! Please include:

1. **Use case**: Why is this enhancement needed?
2. **Proposed solution**: How should it work?
3. **Alternatives considered**: Other approaches you've thought about
4. **Additional context**: Any other relevant information

### Pull Requests

1. **Fork the repository**
2. **Create a feature branch**: `git checkout -b feature/your-feature-name`
3. **Make your changes**
4. **Test thoroughly**: Run tests and validate on multiple OS versions
5. **Commit with clear messages**: Follow commit message guidelines below
6. **Push to your fork**: `git push origin feature/your-feature-name`
7. **Create a Pull Request**: Provide a clear description of changes

## Development Guidelines

### Ansible Best Practices

- **Idempotency**: All tasks must be idempotent
- **Check mode support**: Tasks should support `--check` mode where possible
- **Clear task names**: Use descriptive names that explain what the task does
- **Tags**: Add appropriate tags to all tasks
- **Variables**: Use descriptive variable names with appropriate defaults
- **Documentation**: Update README files when adding features

### Code Style

- Use 2 spaces for indentation in YAML files
- Keep lines under 120 characters when possible
- Use meaningful variable and task names
- Add comments for complex logic
- Follow YAML best practices

### Testing Requirements

Before submitting a PR, ensure:

1. **Syntax check passes**:
   ```bash
   ansible-playbook playbooks/*.yml --syntax-check
   ```

2. **Lint check passes**:
   ```bash
   ansible-lint playbooks/*.yml roles/*/tasks/*.yml
   ```

3. **Test playbook passes**:
   ```bash
   ansible-playbook tests/test_playbook.yml
   ```

4. **Manual testing on supported OS**:
   - Ubuntu 20.04/22.04
   - CentOS 7/8
   - RHEL 8/9

### Commit Message Guidelines

Format: `<type>(<scope>): <subject>`

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

**Examples**:
```
feat(cleanup): add support for Docker image cleanup
fix(verification): correct disk usage calculation for XFS
docs(readme): update installation instructions
test(cleanup): add test for log compression
```

## Project Structure

```
ansible_disk_space/
├── roles/
│   ├── disk_verification/      # Disk space verification role
│   └── filesystem_cleanup/     # Cleanup operations role
├── playbooks/                  # Main playbooks
├── inventory/                  # Example inventory
├── tests/                      # Test playbooks
└── docs/                       # Additional documentation
```

## Adding New Features

### Adding a New Cleanup Target

1. Create a new task file in `roles/filesystem_cleanup/tasks/`
2. Follow the pattern of existing cleanup tasks:
   - Check if target exists
   - Calculate space to be freed
   - Display dry-run summary
   - Perform cleanup (if not dry-run)
   - Display results
   - Update cleanup statistics
3. Include the new task in `roles/filesystem_cleanup/tasks/main.yml`
4. Add configuration variables to `roles/filesystem_cleanup/defaults/main.yml`
5. Update documentation in role README
6. Add tests to `tests/test_playbook.yml`

### Adding OS Support

1. Test existing playbooks on the new OS
2. Add OS-specific handling where needed (especially in package cache cleanup)
3. Update documentation to list the new OS as supported
4. Add the OS to test matrix

## Documentation

When adding features or making changes:

1. Update the main [`README.md`](README.md:1)
2. Update role-specific READMEs
3. Update inline documentation in playbooks and tasks
4. Add examples for new features
5. Update variable documentation

## Testing

### Local Testing

```bash
# Syntax check
ansible-playbook playbooks/verify_disk_space.yml --syntax-check
ansible-playbook playbooks/remediate_disk_space.yml --syntax-check

# Lint check
ansible-lint playbooks/*.yml

# Run tests
ansible-playbook tests/test_playbook.yml

# Test on specific OS
ansible-playbook tests/test_playbook.yml -l ubuntu-test
```

### Integration Testing

Test the complete workflow:

1. Run verification playbook
2. Run remediation in dry-run mode
3. Run remediation in actual mode
4. Verify results

## Release Process

1. Update version in documentation
2. Update CHANGELOG.md
3. Create a git tag: `git tag -a v1.x.x -m "Version 1.x.x"`
4. Push tag: `git push origin v1.x.x`

## Questions?

If you have questions about contributing:

1. Check existing issues and pull requests
2. Review documentation
3. Create a new issue with your question

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.

## Thank You!

Your contributions help make this project better for everyone. Thank you for taking the time to contribute!