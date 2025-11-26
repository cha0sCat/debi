# GitHub Actions Workflow for Testing debi.sh

## Overview

This workflow tests the Debian Network Reinstall Script (`debi.sh`) using KVM virtualization in GitHub Actions. It validates that the script correctly prepares a system for Debian installation, performs the actual installation, and verifies the newly installed system.

## Test Matrix

The workflow tests multiple configurations:

1. **Debian 11 with USTC mirror** - Chinese mirror with ethx naming
2. **Debian 12 with USTC mirror** - Chinese mirror with ethx naming
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
- Configures root SSH access with password `rootpass123`

### 3. VM Execution
- Starts QEMU VM with KVM acceleration
- Waits for SSH to become available
- Uploads `debi.sh` script to the VM

### 4. Script Testing
- Executes `debi.sh` with matrix-specific arguments
- Each test uses a new password (`newpass123`) for the installed system
- Captures output for debugging

### 5. Installation Validation
The workflow validates that the script:
- Successfully downloads Debian installer components to `/boot/debian-*/`
- Creates the required `linux` and `initrd.gz` files
- Updates GRUB configuration with Debian Installer entry
- Sets GRUB default to boot into the installer

### 6. Full Installation Test
After validation, the workflow:
- Reboots the VM to start the Debian installer
- Waits for the installation to complete (up to 30 minutes)
- Polls for SSH availability with the **new password** (`newpass123`)
- Verifies the newly installed system

### 7. Success Criteria

A test passes if:
- ✓ Script executes without critical errors
- ✓ Installer files exist in `/boot/debian-*/`
- ✓ GRUB configuration contains "Debian Installer" entry
- ✓ VM successfully reboots into installer
- ✓ Installation completes automatically
- ✓ New system boots and is accessible via SSH with new credentials

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
- New system is accessible with new credentials
- Green checkmark in GitHub Actions UI

### ❌ Failure
Common failure scenarios and debugging:

1. **SSH connection timeout (old credentials)**
   - Check serial console log
   - Verify cloud-init configuration

2. **Installer files not downloaded**
   - Review debi-output.log
   - Check network connectivity
   - Verify mirror availability

3. **GRUB update failed**
   - Check for GRUB installation issues
   - Verify disk partitioning

4. **Installation timeout**
   - Installation may take longer than expected
   - Check serial console for installer progress
   - Verify network connectivity to mirrors

5. **Cannot connect with new credentials**
   - Installation may have failed
   - Check serial console for installation errors
   - Verify preseed configuration

## Password Strategy

The workflow uses two different passwords to verify installation success:

| Phase | User | Password | Purpose |
|-------|------|----------|---------|
| Initial VM (cloud-init) | root | `rootpass123` | Access the base system to run debi.sh |
| New System (installed) | root/debian | `newpass123` | Verify the new system was installed |

If SSH connects with `newpass123`, it proves the new Debian system was installed successfully.

## Limitations

This workflow tests the complete installation process:
- ✓ Script argument parsing
- ✓ Installer file downloads
- ✓ GRUB configuration updates
- ✓ Reboot into installer
- ✓ Unattended installation
- ✓ New system verification

Note: Full installation takes 10-30 minutes per test case.

## Security Note

The workflow uses hardcoded passwords for test VMs. This is acceptable because:
- VMs are ephemeral (created and destroyed during workflow execution)
- VMs run only on the GitHub Actions runner (not publicly accessible)
- SSH port forwarding is local to the runner only
- All test VMs are destroyed after the workflow completes

These passwords are not used for any production systems.

## Adding New Test Cases

To add a new test configuration:

```yaml
- name: "Your test description"
  test_id: "unique-test-id"
  debian_version: "12"
  args: "--version 12 --your --custom --args --user root --password newpass123"
  image_url: "https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2"
  new_user: "root"
  new_password: "newpass123"
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

### Installation hangs
If the installation takes too long:
- Check the serial console for progress
- Verify mirror connectivity
- Consider using a faster mirror (USTC, Cloudflare)
