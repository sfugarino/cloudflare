# iSCSI TCP Kernel Module Setup on NVIDIA Jetson Orin (L4T 36 / JetPack 6)

This guide walks through building and loading the `iscsi_tcp` kernel module on a Jetson Orin device running L4T 36.x (JetPack 6 / Ubuntu 22.04 Jammy). The default JetPack kernel does not include `iscsi_tcp` as a loadable module, so it must be compiled from source.

---

## Prerequisites

- NVIDIA Jetson Orin board running JetPack 6 (L4T 36.x)
- Internet access
- At least ~10 GB of free disk space for kernel sources
- `sudo` privileges

---

## Steps

### 1. Install open-iscsi

Install the userspace iSCSI initiator tools. This provides `iscsid`, `iscsiadm`, and related utilities.

```bash
sudo apt install open-iscsi
```

---

### 2. Clone the Jetson Orin Kernel Builder

This repository contains helper scripts to download kernel sources and build modules specifically for Jetson Orin.

```bash
git clone https://github.com/jetsonhacks/jetson-orin-kernel-builder.git
```

---

### 3. Change into the Repository Directory

```bash
cd jetson-orin-kernel-builder
```

---

### 4. Download Kernel Sources

The script automatically detects your L4T version and downloads the matching kernel source tree into `/usr/src/`.

```bash
./scripts/get_kernel_sources.sh
```

**Example output:**

```
[INFO] 2026-04-07 09:07:07 - Detected L4T version: 36 (5.0)
[INFO] 2026-04-07 09:07:07 - Kernel sources directory: /usr/src/
```

If kernel sources already exist, you will be prompted:

```
Kernel sources already exist at /usr/src//kernel.
What would you like to do?
[K]eep existing sources (default)
[R]eplace (delete and re-download)
[B]ackup and download fresh sources
Enter your choice (K/R/B):
```

Press **K** to keep existing sources (fastest) or **R** to re-download fresh.

---

### 5. Enable iSCSI in the Kernel Build Configuration

Launch the interactive kernel configuration editor (based on `menuconfig`):

```bash
./scripts/edit_config_cli.sh
```

Navigate to the following menu path and **enable** the option as a module (`M`):

```
Device Drivers
  └── SCSI device support
        └── SCSI low-level drivers
              └── iSCSI Initiator over TCP/IP   [M]
```

> **Tip:** Use the arrow keys to navigate and press `Space` to toggle the selection to `<M>` (module). Save and exit when done.

---

### 6. Build the Kernel Modules

Compile only the kernel modules (not the full kernel image). This is significantly faster than a full kernel build.

```bash
./scripts/make_kernel_modules.sh
```

This may take 10–30 minutes depending on your hardware. The resulting `.ko` files will be placed under the kernel source build tree.

**Example output:**

```
Modules built successfully
Do you want to install the modules? (y/n): n
```
Press **n**. Will install scsi modules in next step. 
---

### 7. Copy the Built Modules to the System Modules Directory

Copy all newly built SCSI-related kernel modules to the correct location for the running kernel:

```bash
sudo cp /usr/src/kernel/kernel-jammy-src/drivers/scsi/*.ko /lib/modules/$(uname -r)/kernel/drivers/scsi
```

> `$(uname -r)` automatically expands to your current kernel version string.

---

### 8. Rebuild the Module Dependency Map

Regenerate the `modules.dep` and related dependency files so the kernel can find and load the new modules:

```bash
sudo depmod -a
```

---

### 9. Load the `libiscsi` Module

`libiscsi` is required by `iscsi_tcp` as a dependency. Load it first:

```bash
sudo modprobe libiscsi
```

---

### 10. Load the `iscsi_tcp` Module

Load the iSCSI TCP initiator module:

```bash
sudo modprobe iscsi_tcp
```

Verify it loaded successfully:

```bash
lsmod | grep iscsi
```

Expected output (example):

```
iscsi_tcp              32768  0
libiscsi_tcp           32768  1 iscsi_tcp
libiscsi               73728  3 iscsi_tcp,libiscsi_tcp
scsi_transport_iscsi   122880  3 iscsi_tcp,libiscsi
```

---

### 11. Restart the iSCSI Daemon

Restart `iscsid` so it picks up the newly loaded kernel modules:

```bash
sudo systemctl restart iscsid
```

Check that the service is running:

```bash
sudo systemctl status iscsid
```

---

### 12. Make the Modules Load Automatically at Boot

To ensure `libiscsi` and `iscsi_tcp` are loaded on every boot, add them to a modules configuration file:

```bash
sudo touch /etc/modules-load.d/iscsi.conf
echo "libiscsi" | sudo tee -a /etc/modules-load.d/iscsi.conf
echo "iscsi_tcp" | sudo tee -a /etc/modules-load.d/iscsi.conf
```

> **Note:** Using `sudo tee -a` is preferred over `echo ... >> file` with sudo, since the redirect `>>` runs as the current user and will fail on a root-owned file.

Verify the file contents:

```bash
cat /etc/modules-load.d/iscsi.conf
```

Expected output:

```
libiscsi
iscsi_tcp
```

---

## Connecting to an iSCSI Target

Once the module is loaded and `iscsid` is running, you can discover and log in to iSCSI targets:

```bash
# Discover targets on a remote host
sudo iscsiadm -m discovery -t sendtargets -p <target-ip>

# Log in to a target
sudo iscsiadm -m node -T <target-name> -p <target-ip> --login
```

---

## Troubleshooting

### Module Loading Issues

If you encounter problems loading the `iscsi_tcp` module, follow these diagnostic and fix steps:

#### 1. Module Not Found Error

**Symptom:**
```
sudo modprobe iscsi_tcp
modprobe: FATAL: Module iscsi_tcp not found in directory /lib/modules/5.15.122-tegra-ubuntu-jammy
```

**Diagnostic Steps:**

a. Check if the `.ko` file exists:
```bash
ls -la /lib/modules/$(uname -r)/scsi/iscsi_tcp.ko
```

If not found, the module was not copied. Continue to fix step 1c.

b. Verify the directory structure:
```bash
uname -r
ls -la /lib/modules/$(uname -r)/
```

c. **Fix:** Copy the module from the kernel build directory:
```bash
sudo cp /usr/src/kernel/kernel-jammy-src/drivers/scsi/iscsi_tcp.ko \
  /lib/modules/$(uname -r)/scsi/iscsi_tcp.ko
sudo cp /usr/src/kernel/kernel-jammy-src/drivers/scsi/libiscsi.ko \
  /lib/modules/$(uname -r)/scsi/libiscsi.ko
sudo cp /usr/src/kernel/kernel-jammy-src/net/scsi/libiscsi_tcp.ko \
  /lib/modules/$(uname -r)/scsi/libiscsi_tcp.ko
```

d. Rebuild the dependency map:
```bash
sudo depmod -a
```

e. Try loading again:
```bash
sudo modprobe iscsi_tcp
```

---

#### 2. Module Version Mismatch

**Symptom:**
```
modprobe: ERROR: could not insert 'iscsi_tcp': Exec format error
```

This usually means the `.ko` file was built for a different kernel version.

**Diagnostic Steps:**

a. Check the running kernel version:
```bash
uname -r
```

b. Check the kernel source version:
```bash
cat /usr/src/kernel/kernel-jammy-src/include/generated/utsrelease.h | grep UTS_RELEASE
```

c. Verify what kernel you compiled for:
```bash
grep "kernel-jammy-src" /usr/src/kernel/kernel-jammy-src/.config | head -5
```

**Fix:** If versions don't match, rebuild the kernel modules:

```bash
cd jetson-orin-kernel-builder
./scripts/get_kernel_sources.sh  # Choose [R] to replace
./scripts/make_kernel_modules.sh
sudo cp /usr/src/kernel/kernel-jammy-src/drivers/scsi/*.ko /lib/modules/$(uname -r)/kernel/drivers/scsi/
sudo depmod -a
```

---

#### 3. Dependency Issues

**Symptom:**
```
modprobe: ERROR: could not insert 'iscsi_tcp': Unknown symbol in module
```

The module depends on other symbols that aren't available.

**Diagnostic Steps:**

a. Check what dependencies `iscsi_tcp` has:
```bash
modinfo /lib/modules/$(uname -r)/scsi/iscsi_tcp.ko | grep depends
```

b. Load dependencies in order:
```bash
sudo modprobe libiscsi
sudo modprobe libiscsi_tcp
sudo modprobe iscsi_tcp
```

c. Verify all are loaded:
```bash
lsmod | grep -E "libiscsi|iscsi_tcp|scsi_transport_iscsi"
```

**Fix:** If dependencies fail to load, ensure all related `.ko` files are in the correct location:

```bash
sudo ls -la /lib/modules/$(uname -r)/scsi/ | grep -E "iscsi|libiscsi"
```

---

#### 4. Module File Permissions

**Symptom:**
```
modprobe: ERROR: could not open '/lib/modules/5.15.122-tegra-ubuntu-jammy/scsi/iscsi_tcp.ko': Permission denied
```

**Fix:** Check and correct file permissions:

```bash
sudo chmod 644 /lib/modules/$(uname -r)/scsi/iscsi_tcp.ko
sudo chmod 644 /lib/modules/$(uname -r)/scsi/libiscsi.ko
sudo chmod 644 /lib/modules/$(uname -r)/scsi/libiscsi_tcp.ko
```

---

#### 5. Kernel Sources Not Found

**Symptom:**
```
./scripts/make_kernel_modules.sh
Error: Kernel sources not found at /usr/src/kernel
```

**Fix:** Download kernel sources first:

```bash
./scripts/get_kernel_sources.sh
```

Select one of the options:
- **[K]eep** if sources already exist
- **[R]eplace** to download fresh
- **[B]ackup** to preserve and download new

---

#### 6. Build Fails with Compiler Errors

**Symptom:**
```
error: 'some_symbol' undeclared
```

**Diagnostic Steps:**

a. Check if build-essential is installed:
```bash
gcc --version
make --version
```

b. Check if kernel headers are available:
```bash
ls /usr/src/linux-headers-$(uname -r)/
```

**Fix:** Install required build tools:

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) bison flex
cd jetson-orin-kernel-builder
./scripts/make_kernel_modules.sh
```

---

#### 7. Module Loads But `iscsid` Still Fails

**Symptom:**
```
sudo systemctl restart iscsid
sudo systemctl status iscsid
# Shows "failed"
```

**Diagnostic Steps:**

a. Check `iscsid` logs:
```bash
sudo journalctl -u iscsid -n 50
```

b. Verify modules are truly loaded:
```bash
lsmod | grep -i iscsi
```

c. Check if the iSCSI daemon can see the module:
```bash
sudo iscsiadm --version
sudo iscsid --help 2>&1 | head -20
```

**Fix:** Restart in correct order:

```bash
sudo modprobe libiscsi
sudo modprobe iscsi_tcp
sudo systemctl restart iscsid
sudo systemctl status iscsid
sudo systemctl enable iscsid
```

---

#### 8. Modules Unload After Reboot

**Symptom:**
```
# After reboot
lsmod | grep iscsi_tcp
# Returns empty
```

**Diagnostic Steps:**

a. Verify the autoload config file exists:
```bash
cat /etc/modules-load.d/iscsi.conf
```

b. Check if systemd is reading the config:
```bash
sudo systemctl status systemd-modules-load.service
```

**Fix:** Ensure the configuration file is correctly set up:

```bash
sudo touch /etc/modules-load.d/iscsi.conf
echo "libiscsi" | sudo tee /etc/modules-load.d/iscsi.conf
echo "iscsi_tcp" | sudo tee -a /etc/modules-load.d/iscsi.conf
echo "libiscsi_tcp" | sudo tee -a /etc/modules-load.d/iscsi.conf

# Verify:
cat /etc/modules-load.d/iscsi.conf

# Force systemd to reload:
sudo systemctl restart systemd-modules-load.service

# Reboot to test persistence:
sudo reboot
```

---

### Quick Reference: Common Commands

| Command | Purpose |
|---|---|
| `modinfo /lib/modules/$(uname -r)/scsi/iscsi_tcp.ko` | Show module details and dependencies |
| `modprobe -v iscsi_tcp` | Load module with verbose output |
| `modprobe -r iscsi_tcp` | Unload module (requires no active connections) |
| `depmod -v` | Rebuild dependency map with verbose output |
| `lsmod \| grep iscsi` | Show all loaded iSCSI-related modules |
| `dmesg \| grep -i iscsi` | Check kernel logs for iSCSI errors |
| `sudo journalctl -u iscsid -f` | Follow iSCSI daemon logs in real time |

---

### Quick Checklist

Before troubleshooting, verify these basics:

```bash
# 1. Kernel version
uname -r

# 2. Module file exists
ls -la /lib/modules/$(uname -r)/scsi/ | grep iscsi

# 3. Depmod cache is current
sudo depmod -a

# 4. Try loading with verbose output
sudo modprobe -v iscsi_tcp

# 5. Check kernel logs
dmesg | tail -20

# 6. Verify iscsid is running
sudo systemctl status iscsid

# 7. Check autoload config
cat /etc/modules-load.d/iscsi.conf
```

---

### Issue Summary Table

| Issue | Resolution |
|---|---|
| `modprobe: FATAL: Module iscsi_tcp not found` | Verify the `.ko` was copied to `/lib/modules/$(uname -r)/scsi` and `depmod -a` was run. |
| `Exec format error` | Kernel version mismatch. Rebuild modules for the running kernel version. |
| `Unknown symbol in module` | Load dependencies first with `modprobe libiscsi && modprobe libiscsi_tcp`. |
| `Permission denied` | Run with `sudo` or fix file permissions with `sudo chmod 644`. |
| `make_kernel_modules.sh` fails | Ensure build tools and kernel headers are installed. Re-run `apt install build-essential`. |
| `iscsid` fails to start | Check `journalctl -u iscsid`. Ensure modules are loaded and `/etc/modules-load.d/iscsi.conf` is set up. |
| Module unloads after reboot | Confirm `/etc/modules-load.d/iscsi.conf` contains all module names and `systemd-modules-load.service` is enabled. |

---

## References

- [JetsonHacks – jetson-orin-kernel-builder](https://github.com/jetsonhacks/jetson-orin-kernel-builder)
- [open-iscsi project](https://github.com/open-iscsi/open-iscsi)
- [NVIDIA Jetson Linux Developer Guide](https://docs.nvidia.com/jetson/archives/r36.2/DeveloperGuide/)
- [kernel.org – iSCSI documentation](https://www.kernel.org/doc/html/latest/scsi/index.html)
