# dracut-anaconda Module

This directory contains the **dracut-anaconda** module for the Anaconda installer.

## Structure

This is a dracut module that integrates Anaconda (the Fedora/RHEL installer) into the initramfs. It contains:

- **module-setup.sh**: Main module definition and installation script
- **parse-*.sh**: Command-line parsing hooks (options, kickstart, repo, network)
- **anaconda-lib.sh**: Shared library functions (18KB of utilities)
- **fetch-*.sh**: Scripts to fetch kickstart files and driver updates
- **driver_updates.py**: Python script for handling driver updates
- Various hook scripts for different dracut phases (cmdline, pre-udev, initqueue, etc.)

## Dracut Hooks

Scripts are installed at specific priorities in dracut hooks:
- **cmdline** (25-28, 99): Parse boot parameters
- **pre-udev** (30): Load kernel modules
- **pre-trigger** (50): Generate udev rules
- **initqueue** (settled/online): Handle network setup and kickstart fetching
- **pre-pivot**: Copy files to new root before switching

See README for detailed documentation on dracut hook ordering and variable sharing.

## Current Work

**TODO**: Auto-detect SSL certificates for Python dependencies
- See `IMPLEMENTATION_STEPS.md` for detailed implementation plan
- Reference: WIP commit `f21662157a`
- Target: module-setup.sh:95-128

## Parent Context

Git root: `../` (anaconda installer repository)
Working dir: `anaconda/dracut/`
