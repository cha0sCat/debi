# GitHub Actions Workflow for Testing debi.sh

## Overview

This workflow tests the Debian Network Reinstall Script (`debi.sh`) using KVM virtualization in GitHub Actions. It validates that the script correctly prepares a system for Debian installation with various configurations.

## Test Matrix

The workflow tests multiple configurations:

1. **Debian 11 with USTC mirror** - Chinese mirror with static IPv4
2. **Debian 12 with USTC mirror** - Chinese mirror with static IPv4
3. **Debian 11 with default mirror** - Global mirror with DHCP
4. **Debian 12 with Cloudflare** - Cloudflare DNS with ethx naming

## How It Works

### 1. Environment Setup
- Enables KVM support on GitHub Actions runner
- Installs QEMU and related tools
- Downloads official Debian cloud images

### 2. VM Preparation
- Creates a copy-on-write disk image from the base cloud image
- Generates cloud-init configuration for initial VM setup
- Configures root SSH access for testing

### 3. VM Execution
- Starts QEMU VM with KVM acceleration
- Waits for SSH to become available
- Uploads `debi.sh` script to the VM

### 4. Script Testing
- Executes `debi.sh` with matrix-specific arguments
- Captures output for debugging

### 5. Validation
The workflow validates that the script:
- Successfully downloads Debian installer components to `/boot/debian-*/`
- Creates the required `linux` and `initrd.gz` files
- Updates GRUB configuration with Debian Installer entry
- Sets GRUB default to boot into the installer

### 6. Success Criteria

A test passes if:
- ✓ Script executes without critical errors
- ✓ Installer files exist in `/boot/debian-*/`
- ✓ GRUB configuration contains "Debian Installer" entry
- ✓ System is ready for reboot into installer

## Running the Tests

### Automatic Execution
Tests run automatically on:
- Push to `master` or `main` branch
- Pull requests targeting `master` or `main`

### Manual Execution
You can manually trigger the workflow:
1. Go to Actions tab in GitHub
2. Select "Test Debian Installation Script"
3. Click "Run workflow"

## Test Artifacts

Each test run uploads:
- `debi-output.log` - Complete output from script execution
- `serial.log` - VM serial console log

Access artifacts from the workflow run summary page.

## Interpreting Results

### ✅ Success
- All validation checks pass
- Green checkmark in GitHub Actions UI
- System ready for installer reboot

### ❌ Failure
Common failure scenarios and debugging:

1. **SSH connection timeout**
   - Check serial console log
   - Verify cloud-init configuration

2. **Installer files not downloaded**
   - Review debi-output.log
   - Check network connectivity
   - Verify mirror availability

3. **GRUB update failed**
   - Check for GRUB installation issues
   - Verify disk partitioning

## Limitations

This workflow tests:
- ✓ Script argument parsing
- ✓ Installer file downloads
- ✓ GRUB configuration updates
- ✓ Pre-reboot preparation

It does NOT test:
- ✗ Actual reboot into installer
- ✗ Complete installation process
- ✗ Post-installation system state

Testing the full installation would require:
- Nested virtualization or bare metal
- Automated installer interaction
- Significantly longer execution time (30+ minutes)

## Security Note

The workflow uses hardcoded passwords (`testpass123`, `rootpass123`) for test VMs. This is acceptable because:
- VMs are ephemeral (created and destroyed during workflow execution)
- VMs run only on the GitHub Actions runner (not publicly accessible)
- SSH port forwarding is local to the runner only
- All test VMs are destroyed after the workflow completes

These passwords are not used for any production systems.

## Adding New Test Cases

To add a new test configuration:

```yaml
- name: "Your test description"
  debian_version: "12"
  args: "--version 12 --your --custom --args"
  image_url: "https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2"
```

## Troubleshooting

### KVM not available
If KVM is not available, the workflow will fail. GitHub Actions runners support KVM, but some self-hosted runners may not.

### Network timeouts
If downloads fail, consider:
- Adding retry logic
- Using alternative mirrors
- Checking GitHub Actions network restrictions

### VM boot failures
Check the serial console log artifact for kernel messages and boot errors.
