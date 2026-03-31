# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the **dracut-anaconda module** (`80anaconda`) - a dracut module that integrates Anaconda (Fedora/RHEL installer) into the initramfs. It handles boot parameter parsing, network setup, driver updates, kickstart parsing, and finding the installer runtime image.

**Context**: This `dracut/` directory is part of the larger Anaconda installer repository. Parent directory contains the main installer (`pyanaconda/`), tests, and build system.

## Structure

- **module-setup.sh**: Main module definition, installs files into initramfs
- **parse-*.sh**: Command-line parsing hooks (options, kickstart, repo, network, dd)
- **anaconda-lib.sh**: Shared library functions (18KB of utilities)
- **fetch-*.sh**: Scripts to fetch kickstart files and driver updates
- **driver_updates.py**: Python script for handling Driver Update Disks
- **dd/**: C utilities (`dd_list`, `dd_extract`) for processing driver RPMs
- Hook scripts for various dracut phases (see below)

## Development Commands

**Note**: All commands run from parent `anaconda/` directory, not from `dracut/`

### Build and Test

```bash
cd ..  # Go to anaconda root

# Build
./autogen.sh && ./configure && make
# For RHEL 10+: ./configure --disable-glade

# Run all tests
make check

# Run specific tests
make TESTS=unit_tests/unit_tests.sh UNIT_TESTS_PATTERN='test_driver_updates' check
```

### Container Testing (Recommended)

```bash
cd ..

# Run tests in container
make -f Makefile.am container-ci

# Interactive development
make -f Makefile.am container-shell
```

### Testing with Overlay Images

```bash
# Create overlay with changed files
mkdir overlay && cp -a dracut/module-setup.sh overlay/lib/dracut/modules.d/80anaconda/
chown -R root:root overlay
cd overlay && find . | cpio -o --format=newc | gzip -9cv > ../overlay.img

# Boot with: initrd=<initrd> overlay.img
```

See `README-testing-changes.rst` for details.

## Dracut Hook System

Scripts are installed at specific priorities in dracut hooks (see `module-setup.sh`):

- **cmdline** (25-29): Parse boot parameters (`parse-anaconda-*.sh`)
- **pre-udev** (30): Load kernel modules
- **pre-trigger** (50-55): Generate udev rules
- **initqueue/settled**: Runs once when udev settles
- **initqueue/online**: Runs when network interfaces come online
- **pre-pivot**: Copy files to `$NEWROOT` before switching
- **cleanup**, **pre-shutdown**: Cleanup phases

See `README` for detailed hook ordering and variable sharing.

**Key dracut variables**: `$NEWROOT` (usually `/sysroot`), `$root`, `$netroot`, `$kickstart`, `$anac_updates`

## Key Subsystems

### Driver Update Disks (DUD)

Handles loading kernel modules and installer enhancements before main installer starts.

**Flow**: `inst.dd=<source>` or `LABEL=OEMDRV` → parse-anaconda-dd.sh → driver-updates-genrules.sh → fetch-driver-net.sh → driver_updates.py → extract to `/updates` → copy repos to `/run/install/DD-*`

**Utilities** (`dd/`): `dd_list` (list matching RPMs), `dd_extract` (extract drivers/firmware)

See `README-driver-updates.md` for complete documentation.

### Python Dependencies

`python-deps` script uses `ModuleFinder` to discover dependencies for `parse-kickstart` and `driver_updates.py`. Module-setup.sh automatically scans installed Python files for SSL certificate paths during installation (e.g., `/etc/pki/ca-trust/extracted/pem/*.pem`).

## Initramfs File Locations

Hook scripts installed to `/lib/dracut/hooks/<hookname>/<priority>-<scriptname>.sh`. Key files:
- `anaconda-lib.sh` → `/lib/anaconda-lib.sh`
- `driver_updates.py` → `/bin/driver-updates`
- `parse-kickstart` → `/sbin/parse-kickstart`
- `dd_list`, `dd_extract` → `/bin/` (from `dd/` subdirectory)

## Debugging

**Live environment**:
```bash
# Boot with: rd.break=pre-pivot rd.shell
# Then check:
ls /tmp/                    # Config files from cmdline hooks
journalctl                  # Dracut logs
ls /run/install/            # DUD repos, package lists
ls /lib/dracut/hooks/       # Installed hook scripts
```

**Shellcheck**: Configured in `.shellcheckrc` (disables SC1090, SC1091, SC2230, SC2154 for dracut compatibility)
