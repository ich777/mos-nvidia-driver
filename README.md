# MOS Nvidia Driver

mos-nvidia-driver provides a **MOS plugin** for managing Nvidia drivers.

---

## Overview

This repository contains the **MOS plugin implementation**, optional helper
functions, and configuration files (such as `settings.json`) required to
integrate Nvidia driver management into MOS.

The plugin enables MOS to download and install Nvidia driver on demand directly
onto the system.

### Driver Source

- Nvidia: [https://www.nvidia.com/en-us/drivers/unix/](https://www.nvidia.com/en-us/drivers/unix/)
- Nvidia-Open Source: [https://github.com/NVIDIA/open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules)

No driver binaries are included in this repository.

---

## Build & Automation

This repository includes a **GitHub Actions workflow** used to build and package
the plugin for MOS.

The build process is fully automated and produces artifacts that can be consumed
by the MOS Hub or MOS release tooling.

---

## Licensing

The contents of this repository (plugin code, scripts, and configuration)
are licensed under **GPL-3.0**.

Downloaded Nvidia drivers remain licensed under their respective upstream licenses
and are not part of this repository.
