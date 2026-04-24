# Harvester Multipath Configuration Flow

This document explains how the multipath configuration from PR #1117 flows through the Harvester installer codebase and gets applied to the system.

## Overview

The multipath configuration allows users to control which storage devices should be managed by Linux's multipath daemon (`multipathd`). It supports both simple vendor/product matching and advanced WWID-based patterns with blacklist exceptions.

## Configuration Flow

### Step 1: User Provides YAML Configuration

Users define their multipath settings in a YAML configuration file:

```yaml
os:
  externalStorageConfig:
    enabled: true
    multiPathConfig:
      blacklistWwids:
        - ".*"  # Blacklist all devices
      blacklistExceptionWwids:
        - "^0QEMU_QEMU_HARDDISK_disk[0-9]+"  # Allow specific QEMU disks
      blacklistExceptions:
        - vendor: "DELL"
          product: "POWERVAULT"
```

**Related Code:**
- Configuration struct definition: [pkg/config/config.go:224-229](../pkg/config/config.go#L224-L229)

---

### Step 2: YAML Parsing and Config Loading

The YAML is loaded and parsed into a `HarvesterConfig` struct.

**Function:** `LoadHarvesterConfig()`

**Location:** [pkg/config/schemas.go:28-45](../pkg/config/schemas.go#L28-L45)

```go
func LoadHarvesterConfig(yamlBytes []byte) (*HarvesterConfig, error) {
    result := NewHarvesterConfig()
    data := map[string]interface{}{}
    
    // Unmarshal YAML
    if err := yaml.Unmarshal(yamlBytes, &data); err != nil {
        return result, fmt.Errorf("failed to unmarshal yaml: %v", err)
    }
    
    // Convert to HarvesterConfig struct
    if err := convert.ToObj(data, result); err != nil {
        return result, fmt.Errorf("failed to convert to HarvesterConfig: %v", err)
    }
    
    // Parse multipath configuration
    if err := result.ExternalStorage.ParseMultiPathConfig(); err != nil {
        return result, fmt.Errorf("failed to parse external storage multi-path config: %v", err)
    }
    
    return result, nil
}
```

---

### Step 3: Multipath Config Parsing

The `multiPathConfig` field (which is an `interface{}`) gets parsed into the appropriate type.

**Function:** `ParseMultiPathConfig()`

**Location:** [pkg/config/config.go:233-266](../pkg/config/config.go#L233-L266)

**What it does:**
1. Accepts the raw `interface{}` value from YAML
2. Converts it to JSON bytes for easier parsing
3. Tries to parse as **MultiPathOption2** (new format with WWID support) first
4. Falls back to **MultipathOption1** (legacy vendor/product only format)
5. Returns error if neither format matches

```go
func (esc *ExternalStorageConfig) ParseMultiPathConfig() (err error) {
    if esc.MultiPathConfig == nil {
        return nil
    }

    var jsonBytes []byte

    // Convert interface{} to JSON bytes
    if jsonStr, ok := esc.MultiPathConfig.(string); ok {
        jsonBytes = []byte(jsonStr)
    } else {
        jsonBytes, err = json.Marshal(esc.MultiPathConfig)
        if err != nil {
            return fmt.Errorf("failed to marshal multiPathConfig: %w", err)
        }
    }

    // Try MultiPathOption2 first (new format)
    mp := &MultiPathOption2{}
    if err := json.Unmarshal(jsonBytes, mp); err == nil {
        esc.MultiPathConfig = *mp
        return nil
    }

    // Fall back to MultipathOption1 (legacy format)
    var diskConfigs MultipathOption1
    if err := json.Unmarshal(jsonBytes, &diskConfigs); err == nil {
        esc.MultiPathConfig = diskConfigs
        return nil
    }

    return fmt.Errorf("unsupported multiPathConfig format")
}
```

**Related Structs:**
- **MultiPathOption2** (new format): [pkg/config/config.go:273-278](../pkg/config/config.go#L273-L278)
- **MultipathOption1** (legacy format): [pkg/config/config.go:293](../pkg/config/config.go#L293)
- **DiskConfig** (vendor/product pair): [pkg/config/config.go:308-311](../pkg/config/config.go#L308-L311)

---

### Step 4: Convert to YIP Configuration

During installation, the HarvesterConfig is converted to a YIP (cloud-init-like) configuration.

**Function:** `ConvertToCOS()`

**Location:** [pkg/config/cos.go:134-242](../pkg/config/cos.go#L134-L242)

**What it does:**
1. Creates multiple YIP "stages" for different boot phases:
   - `rootfs` - Root filesystem setup
   - `initramfs` - Early boot in RAM filesystem
   - `network` - After network is available
2. Calls `setupExternalStorage()` to add multipath config to the `initramfs` stage
3. Returns a complete YipConfig with all stages assembled

```go
func ConvertToCOS(config *HarvesterConfig) (*yipSchema.YipConfig, error) {
    // Create stages
    rootfs := yipSchema.Stage{}
    initramfs := yipSchema.Stage{
        Users:     make(map[string]yipSchema.User),
        TimeSyncd: make(map[string]string),
    }
    afterNetwork := yipSchema.Stage{
        Hostname: config.OS.Hostname,
        SSHKeys:  make(map[string][]string),
    }
    
    // ... other initramfs setup ...
    
    // Setup external storage (multipath) - LINE 191
    if err := setupExternalStorage(config, &initramfs); err != nil {
        return nil, err
    }
    
    // Disable multipath for Longhorn devices
    disableLonghornMultipathing(&initramfs)
    
    // ... more setup ...
    
    // Assemble all stages into YipConfig
    cosConfig := &yipSchema.YipConfig{
        Name: "Harvester Configuration",
        Stages: map[string][]yipSchema.Stage{
            "rootfs":    {rootfs},
            "initramfs": {initramfs},      // multipath.conf goes here
            "network":   {afterNetwork},
        },
    }
    
    return cosConfig, nil
}
```

**Key call:** [pkg/config/cos.go:191](../pkg/config/cos.go#L191)

---

### Step 5: Setup External Storage

This function adds the multipath configuration to the YIP stage.

**Function:** `setupExternalStorage()`

**Location:** [pkg/config/cos.go:864-886](../pkg/config/cos.go#L864-L886)

**What it does:**
1. Checks if external storage is enabled
2. Enables the `multipathd` systemd service
3. Renders the multipath.conf content from the template
4. Adds `/etc/multipath.conf` to the list of files to be written

```go
func setupExternalStorage(config *HarvesterConfig, stage *yipSchema.Stage) error {
    if !config.OS.ExternalStorage.Enabled {
        return nil
    }
    
    // Enable multipathd service
    stage.Systemctl.Enable = append(stage.Systemctl.Enable, "multipathd")

    if config.ExternalStorage.MultiPathConfig == nil {
        return nil
    }

    // Render the template to generate multipath.conf content
    content, err := config.ExternalStorage.MultiPathConfig.(MultiPathOption).Render()
    if err != nil {
        return fmt.Errorf("error rending multipath.conf template: %v", err)
    }

    // Add file to YIP stage
    stage.Files = append(stage.Files, yipSchema.File{
        Path:        "/etc/multipath.conf",
        Content:     content,
        Permissions: 0755,
    })
    
    return nil
}
```

**File addition:** [pkg/config/cos.go:880-884](../pkg/config/cos.go#L880-L884)

---

### Step 6: Render Configuration Template

The `Render()` method generates the actual multipath.conf file content.

**Interface:** `MultiPathOption`

**Location:** [pkg/config/config.go:268-271](../pkg/config/config.go#L268-L271)

**Implementations:**

#### MultiPathOption2 (New Format)
**Location:** [pkg/config/config.go:280-282](../pkg/config/config.go#L280-L282)

Uses template: [pkg/config/templates/multipath.conf.option2.tmpl](../pkg/config/templates/multipath.conf.option2.tmpl)

**Template structure:**
```
blacklist {
    wwid ".*"
    device {
        vendor "QEMU"
        product "QEMU HARDDISK"
    }
}
blacklist_exceptions {
    wwid "^0QEMU_QEMU_HARDDISK_disk[0-9]+"
    device {
        vendor "DELL"
        product "POWERVAULT"
    }
}
```

#### MultipathOption1 (Legacy Format)
**Location:** [pkg/config/config.go:295-297](../pkg/config/config.go#L295-L297)

Uses template: [pkg/config/templates/multipath.conf.option1.tmpl](../pkg/config/templates/multipath.conf.option1.tmpl)

**Template structure:**
```
blacklist {
    device {
        vendor "!QEMU"
        product "!QEMU HARDDISK"
    }
}
```

Note: Uses `!` prefix for negation (blacklist everything except this).

---

### Step 7: YIP Executes During Boot

The YIP framework (from `github.com/rancher/yip`) processes the configuration during system initialization:

**What happens in the `initramfs` stage:**
1. **Files are written** - `/etc/multipath.conf` is created with the rendered content
2. **Services are enabled** - `multipathd.service` is enabled via systemd
3. **Commands are executed** - Any stage commands run
4. **Environment is configured** - Environment variables, sysctls, etc. are applied

**Timeline:**
- **Installation time**: YIP writes files during OS installation
- **Boot time**: Files are in place when `multipathd` starts
- **Runtime**: `multipathd` reads `/etc/multipath.conf` and manages devices accordingly

---

## Data Structures

### Configuration Types

#### MultiPathOption2 (PR #1117 - New Format)
[pkg/config/config.go:273-278](../pkg/config/config.go#L273-L278)

```go
type MultiPathOption2 struct {
    Blacklist               []DiskConfig `json:"blacklist,omitempty"`
    BlacklistWwids          []string     `json:"blacklistWwids,omitempty"`
    BlacklistExceptions     []DiskConfig `json:"blacklistExceptions,omitempty"`
    BlacklistExceptionWwids []string     `json:"blacklistExceptionWwids,omitempty"`
}
```

**Features:**
- WWID pattern matching with regex support
- Separate blacklist and exception lists
- Supports both vendor/product and WWID matching
- More flexible for complex storage scenarios

#### MultipathOption1 (Legacy Format)
[pkg/config/config.go:293](../pkg/config/config.go#L293)

```go
type MultipathOption1 []DiskConfig
```

**Features:**
- Simple array of vendor/product pairs
- Uses negation (`!`) for blacklisting
- Limited to vendor/product matching only

#### DiskConfig
[pkg/config/config.go:308-311](../pkg/config/config.go#L308-L311)

```go
type DiskConfig struct {
    Vendor  string `json:"vendor"`
    Product string `json:"product"`
}
```

---

## Testing

Test examples can be found in:
- [pkg/config/config_test.go](../pkg/config/config_test.go)

Example tests demonstrate:
- Parsing both configuration formats
- Rendering multipath.conf with WWID patterns
- Combining blacklists and exceptions
- Converting between formats

---

## Summary Diagram

```
┌─────────────────────────┐
│   User YAML Config      │
│  (multiPathConfig)      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ LoadHarvesterConfig()   │  schemas.go:28-45
│  - Parse YAML           │
│  - ParseMultiPathConfig │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ ParseMultiPathConfig()  │  config.go:233-266
│  - Try MultiPathOption2 │
│  - Fall back to Option1 │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   ConvertToCOS()        │  cos.go:134-242
│  - Create YIP stages    │
│  - Call setup functions │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ setupExternalStorage()  │  cos.go:864-886
│  - Enable multipathd    │
│  - Render template      │
│  - Add file to stage    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  Render() method        │  config.go:280-282
│  - Load template        │
│  - Generate content     │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  YIP Framework          │
│  - Write files to disk  │
│  - Enable services      │
│  - /etc/multipath.conf  │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  multipathd daemon      │
│  - Reads config         │
│  - Manages devices      │
└─────────────────────────┘
```

---

## Related Files

| File | Purpose |
|------|---------|
| [pkg/config/config.go](../pkg/config/config.go) | Core configuration structs and parsing logic |
| [pkg/config/schemas.go](../pkg/config/schemas.go) | YAML loading and schema validation |
| [pkg/config/cos.go](../pkg/config/cos.go) | YIP configuration generation |
| [pkg/config/templates/multipath.conf.option1.tmpl](../pkg/config/templates/multipath.conf.option1.tmpl) | Legacy template (vendor/product with negation) |
| [pkg/config/templates/multipath.conf.option2.tmpl](../pkg/config/templates/multipath.conf.option2.tmpl) | New template (WWID + exceptions support) |
| [pkg/config/config_test.go](../pkg/config/config_test.go) | Unit tests with usage examples |

---

## Key Concepts

### YIP Stages

YIP (Yet another Implementation of cloud-init) executes configuration in stages:

- **`rootfs`** - During installation when preparing the root filesystem
- **`initramfs`** - Early boot in the initial RAM filesystem (where multipath config goes)
- **`network`** - After network is available
- **`after-install-chroot`** - Post-installation in a chroot environment

### stage.Files

`stage.Files` is a slice of file definitions that will be written to disk during a specific stage. Each file has:
- **Path** - Where to write the file (e.g., `/etc/multipath.conf`)
- **Content** - The file contents (rendered from template)
- **Permissions** - File permissions (e.g., `0755`)
- **Owner** - Optional file owner

### Multipath Device Management

The Linux `multipathd` daemon uses `/etc/multipath.conf` to:
- Identify which storage devices should be managed as multipath devices
- Blacklist devices that should NOT use multipath (like local disks)
- Create exceptions to blacklist rules (like specific SAN devices)
- Configure path failover policies

WWID (World Wide ID) is a unique identifier for storage devices, more reliable than vendor/product names for device identification.

---

## Boot Process: How the OS Boots from a Multipath Device

This section explains how Harvester handles the scenario where the root disk is a LUN with multiple paths (e.g., a SAN device). The challenge is:

- **During install**: Multipath should be OFF so the installer sees the LUN as a simple raw device (e.g., `/dev/sda`)
- **After install**: The OS must boot through the multipath device (e.g., `/dev/mapper/mpathX`) for redundancy
- **Critical question**: Since the same LUN appears as `/dev/sda`, `/dev/sdb`, AND `/dev/mapper/mpathX` (all with the same filesystem labels), how does the kernel pick the multipath device?

### Phase 1: Installation Time

#### Kernel Arguments Setup

The installer adds `multipath=off` to the kernel command line **only when external storage is disabled**.

**Location:** [pkg/console/util.go:54-56](../pkg/console/util.go#L54-L56)

```go
multipathOff   = "multipath=off"
PartitionType  = "part"
MpathType      = "mpath"
```

**Location:** [pkg/console/util.go:555-560](../pkg/console/util.go#L555-L560)

```go
if !hvstConfig.OS.ExternalStorage.Enabled {
    env = append(env, fmt.Sprintf("HARVESTER_ADDITIONAL_KERNEL_ARGUMENTS=%s", multipathOff))
}
if hvstConfig.OS.AdditionalKernelArguments != "" {
    env = append(env, fmt.Sprintf("HARVESTER_ADDITIONAL_KERNEL_ARGUMENTS=%s", hvstConfig.OS.AdditionalKernelArguments))
}
```

**Key insight:** When external storage IS enabled (multipath scenario), `multipath=off` is **NOT** added. This means multipath WILL activate on first boot.

#### Installer Boot

The installer ISO itself boots with `multipath=off` via the live-boot kernel arguments. This is why during installation:
- The LUN appears as raw devices like `/dev/sda`
- Installer writes partition table directly
- Filesystem labels (`COS_STATE`, `COS_OEM`, `COS_PERSISTENT`, `COS_RECOVERY`) are written to partitions

#### Saving Kernel Args to grubenv

During install, the harv-install script saves the kernel arguments to `/oem/grubenv`:

**Location:** [package/harvester-os/files/usr/sbin/harv-install:446-453](../package/harvester-os/files/usr/sbin/harv-install#L446-L453)

```bash
# /etc/cos/bootargs.cfg appends a new variable $third_party_kernel_args
# if harvester config has os.externalStorageConfig.additionalKernelArguments specified
# then these will be mapped to HARVESTER_ADDITIONAL_KERNEL_ARGUMENTS
# and will be added to /oem/grubenv file
TARGET_FILE="${oem_dir}/grubenv"
if [ -n "${HARVESTER_ADDITIONAL_KERNEL_ARGUMENTS}" ]; then
    grub2-editenv ${TARGET_FILE} set third_party_kernel_args="${HARVESTER_ADDITIONAL_KERNEL_ARGUMENTS}"
fi
```

When external storage is enabled, `multipath=off` is **NOT** in `third_party_kernel_args`, allowing multipath to be active at next boot.

### Phase 2: GRUB Configuration — Boot by Label

The kernel command line uses **filesystem labels**, not device paths.

**Location:** [package/harvester-os/files/etc/cos/bootargs.cfg](../package/harvester-os/files/etc/cos/bootargs.cfg)

```bash
set kernelcmd="$console_params root=LABEL=$state_label cos-img/filename=$img \
    panic=0 net.ifnames=1 \
    rd.cos.oemlabel=$oem_label \
    rd.cos.mount=LABEL=$oem_label:/oem \
    rd.cos.mount=LABEL=$persistent_label:/usr/local \
    rd.cos.oemtimeout=120 audit=1 audit_backlog_limit=8192 \
    intel_iommu=on amd_iommu=on iommu=pt \
    $third_party_kernel_args"
```

**Critical part:** `root=LABEL=$state_label` and `rd.cos.mount=LABEL=...`

The kernel doesn't care if the device is `/dev/sda` or `/dev/mapper/mpathX` — it searches all available block devices for the label `COS_STATE`. This makes the boot completely device-path agnostic.

### Phase 3: Patched multipathd Service

The Harvester installer patches the systemd unit for multipathd to ensure correct boot ordering.

**Location:** [pkg/config/cos.go:920-944](../pkg/config/cos.go#L920-L944)

```go
multipathdUnitPatch := []byte(`[Unit]
Description=Device-Mapper Multipath Device Controller
Before=lvm2-activation-early.service
Before=local-fs-pre.target blk-availability.service shutdown.target
Wants=systemd-udevd-kernel.socket modprobe@dm_multipath.service
After=systemd-udevd-kernel.socket modprobe@dm_multipath.service
After=multipathd.socket systemd-remount-fs.service
Before=initrd-cleanup.service
DefaultDependencies=no
Conflicts=shutdown.target
Conflicts=initrd-cleanup.service
ConditionKernelCommandLine=!nompath
ConditionVirtualization=!container
...`)
```

**Key directives:**
- `Before=local-fs-pre.target` — multipathd MUST start before any filesystem mounting
- `Before=lvm2-activation-early.service` — multipathd starts before LVM
- `ConditionKernelCommandLine=!nompath` — only starts if `nompath` is NOT in kernel cmdline
- `ConditionVirtualization=!container` — doesn't start in containers

### Phase 4: First Boot Sequence

```
Time 0: Kernel boots
        ├─ kernel cmdline: root=LABEL=COS_STATE
        └─ initramfs unpacked

Time 1: initramfs runs systemd
        ├─ Loads dm_multipath module
        ├─ udevd starts
        └─ udev triggers SCSI device discovery

Time 2: SCSI devices appear: /dev/sda, /dev/sdb
        ├─ udev sees them
        ├─ Normally would create /dev/sda1, /dev/sda2 nodes
        └─ BUT: multipath udev rules check first

Time 3: multipathd.service starts (Before=local-fs-pre.target)
        ├─ Reads /etc/multipath.conf (written by setupExternalStorage)
        ├─ Identifies sda+sdb as same LUN (via WWID)
        ├─ Calls device-mapper IOCTL to create dm-0
        └─ kpartx creates /dev/mapper/mpathX-part1, -part2, etc.

Time 4: udev re-evaluates devices
        ├─ sda, sdb now have holders (dm-0)
        ├─ Tags them: SYSTEMD_READY=0, DM_MULTIPATH_DEVICE_PATH=1
        └─ Underlying partition nodes either not created or tagged unusable

Time 5: dracut/cos-init searches for LABEL=COS_STATE
        ├─ blkid scans block devices
        ├─ Skips devices with SYSTEMD_READY=0 or holders
        ├─ Finds COS_STATE on /dev/mapper/mpathX-part2
        └─ Mounts root from multipath device

Time 6: System pivots to real root via multipath
        └─ All subsequent disk access goes through dm-0
```

### Phase 5: Why the OS Boots on mpath, Not on Raw Device

This is the critical question: when `/dev/sda`, `/dev/sdb`, and `/dev/mapper/mpathX` all expose the same `COS_STATE` label, how does the kernel pick the multipath device?

The answer involves multiple layered mechanisms working together:

#### Mechanism 1: Device-Mapper Holders (Exclusive Claim)

Once multipathd claims the underlying SCSI devices, the kernel marks them as "path members":

```
/sys/block/sda/holders/dm-0    # sda is now held by dm-0
/sys/block/sdb/holders/dm-0    # sdb is now held by dm-0
```

When a device has a "holder", it becomes **exclusively claimed**. Tools like `blkid` and `udev` see this and won't expose the filesystem label from the underlying paths.

#### Mechanism 2: udev Rules Tag Path Members

The standard multipath udev rules (from the SUSE base OS) tag the underlying devices:

```
ENV{DM_MULTIPATH_DEVICE_PATH}="1"
ENV{SYSTEMD_READY}="0"
```

The `SYSTEMD_READY=0` tag tells systemd: **"This device is not ready for use as a regular block device"**. This means:
- systemd won't auto-mount it
- The filesystem label is essentially "hidden" from normal device discovery
- Only the multipath device (`dm-0`) is treated as a real, mountable block device

#### Mechanism 3: Partition Handling via kpartx

When the LUN has partitions (your case — `COS_STATE` is a partition label), multipath runs `kpartx` to create:

```
/dev/mapper/mpathX           # the whole multipath device
/dev/mapper/mpathX-part1     # partition 1 of mpathX
/dev/mapper/mpathX-part2     # partition 2 (e.g., COS_STATE)
/dev/mapper/mpathX-part3
```

The underlying `/dev/sda1`, `/dev/sda2` partition nodes either:
- Don't get created by udev (because the parent device `sda` is claimed)
- Or get created but are tagged unusable

So when `blkid` scans for `LABEL=COS_STATE`, it only finds it on `/dev/mapper/mpathX-part2`.

#### Mechanism 4: Strict systemd Ordering

The patched `multipathd.service` (described in Phase 3) ensures:

> **multipathd MUST claim devices BEFORE any filesystem mount operation can begin.**

By the time anything tries to find `LABEL=COS_STATE`, the underlying paths are already "owned" by the multipath device.

#### Mechanism 5: dracut Multipath Module

When `dracut -f --regenerate-all` runs in [package/harvester-os/Dockerfile:23](../package/harvester-os/Dockerfile#L23):

```dockerfile
RUN dracut -f --regenerate-all
```

Dracut detects multipath is configured and includes the **multipath dracut module** in the initramfs. This module:

1. Includes `multipathd`, `multipath`, `kpartx` binaries in initramfs
2. Includes `dm_multipath` kernel module
3. Adds udev rules for multipath in initramfs
4. Includes a service that runs multipathd **before** the root-mount stage

### Summary: Boot Device Selection

| Layer | Mechanism |
|-------|-----------|
| **Kernel** | Device-mapper marks underlying paths as "held" (exclusive claim) |
| **udev** | Tags path members with `SYSTEMD_READY=0`, hiding them from mount |
| **systemd** | `multipathd.service` ordered `Before=local-fs-pre.target` |
| **kpartx** | Creates `/dev/mapper/mpathX-partN` instead of using `/dev/sdaN` |
| **dracut** | Includes multipath module in initramfs to claim devices early |
| **blkid** | Returns the multipath device when scanning for labels |

**The key insight:** *Claiming happens BEFORE searching.* By the time anything tries to find `LABEL=COS_STATE`, the underlying paths are already "owned" by the multipath device, so the label search naturally lands on `/dev/mapper/mpathX-partN`.

### Failure Mode

If for some reason multipathd fails to start (e.g., misconfigured multipath.conf), the system will:
1. Fall back to mounting from `/dev/sda` directly (since the label exists there too)
2. Boot will succeed, but **without multipath redundancy**
3. If one path fails later, the system would crash

This is why the multipath.conf settings from PR #1117 are so important — they need to correctly identify which devices should be claimed by multipath.

### Comparison: Raw Device vs Multipath Device

| Aspect | Raw Device | Multipath Device |
|--------|-----------|------------------|
| Device path | `/dev/sda` | `/dev/mapper/mpathX` |
| Filesystem label | `COS_STATE` | `COS_STATE` (same!) |
| Partition UUID | unchanged | unchanged |
| GRUB binary location | sector on disk | sector on disk |
| Visible to mount | Hidden via udev tags | Yes |
| Has holders | Yes (held by dm-0) | No |

The **filesystem label `COS_STATE` is part of the filesystem itself**, written to the partition during install. When multipath claims the device, the label is still there — just accessed through a different device node.
