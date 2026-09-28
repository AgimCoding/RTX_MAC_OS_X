```markdown

https://buymeacoffee.com/agimcoding

# NVIDIA Ampere on macOS

> Experimental NVIDIA Ampere GPU driver development for macOS, currently targeting the **GeForce RTX 3060 Ti (GA104)**.

This project explores bringing modern NVIDIA Ampere GPUs to macOS by building the required low-level driver stack step by step.

The current work focuses on communication with NVIDIA's **GSP-RM firmware**, VRAM access and management, GPU virtual memory (GMMU), GPU channels, and copy-engine execution.

The long-term objective is to progress from low-level hardware initialization to:

**GPU command execution → display output → IOFramebuffer / WindowServer → GPU acceleration → Metal**

---

## ⚠️ Experimental Project

This is **not a usable NVIDIA macOS driver yet**.

The project performs experimental low-level operations directly against NVIDIA hardware and firmware.

Current builds are intended for development and research only.

Do not install or run experimental builds on a production system.

---

# Target Hardware

Development currently targets:

| Component | Hardware |
|---|---|
| GPU | NVIDIA GeForce RTX 3060 Ti |
| Architecture | Ampere |
| GPU | GA104-A |
| PCI Device | `10DE:2489` |
| VRAM | 8 GB |
| macOS | Sequoia |
| Architecture | x86_64 |

Current development GPU:

```text
NVIDIA GeForce RTX 3060 Ti
GA104-A
8192 MiB VRAM
PCIe x16
```

Support for other Ampere GPUs is **not currently guaranteed**.

---

# Current Status

The project has progressed well beyond simple PCI device detection.

### Hardware-validated milestones

- [x] PCI device detection
- [x] GPU power state / PCI Memory Space Decode
- [x] Safe BAR0 MMIO mapping
- [x] `PMC_BOOT_0` / GPU identity
- [x] GA104 identification
- [x] VRAM capacity detection
- [x] VBIOS analysis
- [x] FWSEC/FRTS execution
- [x] WPR2 setup
- [x] GSP firmware boot
- [x] GSP-RM initialization
- [x] `GET_GSP_STATIC_INFO`
- [x] RTX 3060 Ti identity obtained directly from GSP-RM
- [x] RM client/device/subdevice creation
- [x] FBMEM memory-list registration
- [x] Physical VRAM read/write/restore through PRAMIN
- [x] BAR1 physical VRAM access
- [x] `FERMI_VASPACE_A`
- [x] Host-built GMMU page tables
- [x] `SET_PAGE_DIRECTORY` accepted by GSP-RM
- [ ] GPU channel execution
- [ ] GPFIFO submission
- [ ] Copy Engine command execution
- [ ] Display engine / modesetting
- [ ] IOFramebuffer
- [ ] WindowServer rendering
- [ ] GPU acceleration
- [ ] Metal

---

# GSP-RM

One of the biggest milestones was successfully booting NVIDIA's **GSP-RM firmware** on the RTX 3060 Ti from macOS.

The GPU currently reaches:

```text
FWSEC / FRTS
      ↓
WPR2
      ↓
GSP Booter
      ↓
RISC-V
      ↓
GSP-RM INIT_DONE
      ↓
GET_GSP_STATIC_INFO
      ↓
RTX 3060 Ti / GA104-A
```

Example information returned by GSP-RM:

```text
GPU      : NVIDIA GeForce RTX 3060 Ti
Chip     : GA104-A
VRAM     : 8192 MiB
SKU      : 202
Project  : G190-0012
```

This information is obtained from the running NVIDIA firmware rather than being hardcoded into the driver.

---

# VRAM

Physical VRAM access has been validated on real hardware.

The driver can currently:

```text
Save original VRAM
        ↓
Write test pattern
        ↓
Read it back
        ↓
Verify contents
        ↓
Write inverse pattern
        ↓
Verify again
        ↓
Restore original VRAM
        ↓
Verify SHA-256
```

Example:

```text
write pattern : DE AD BE EF DE AD BE EB ...
readback      : DE AD BE EF DE AD BE EB ...

compare       : PASS
restore       : PASS
```

Both PRAMIN and BAR1 paths have been tested during development.

---

# RM Objects

GSP-RM object creation is functional.

The project has successfully created:

```text
RM Client
    │
    └── Device
          │
          └── Subdevice
```

FBMEM memory descriptors can also be registered with GSP-RM using:

```text
NV01_MEMORY_LIST_FBMEM
```

A key milestone was determining the correct contiguous memory-list representation expected by GSP-RM.

---

# GPU Virtual Memory / GMMU

The next major layer is GPU virtual memory.

The project implements host-built NVIDIA GMMU page tables and has successfully created:

```text
FERMI_VASPACE_A
```

with an externally supplied page-directory hierarchy.

The current page-table work includes the GA104/GP100-style multi-level hierarchy and NVIDIA PTE/PDE formats.

On real hardware, GSP-RM has accepted:

```text
FERMI_VASPACE_A
        ↓
Host-built GMMU page tables
        ↓
SET_PAGE_DIRECTORY
        ↓
NV_OK
```

This proves that GSP-RM accepts the host-provided page-directory configuration.

It does **not yet prove GPU-side virtual-address translation**.

That proof requires an actual GPU engine to dereference one of these virtual addresses.

---

# Next Major Milestone

The immediate goal is the first real GPU-engine command submitted from macOS.

The planned path is:

```text
GMMU
  │
  ▼
GPU Virtual Address Space
  │
  ▼
AMPERE_CHANNEL_GPFIFO_A
      0xC56F
  │
  ▼
GPFIFO
  │
  ▼
AMPERE_DMA_COPY_B
      0xC7B5
  │
  ▼
VRAM A ───────────────► VRAM B
```

The test will use two controlled VRAM regions.

```text
A = source
B = destination
C = guard region
```

Success requires:

```text
A unchanged
B == A
C unchanged
completion observed
no GPU fault
```

This will be the first unambiguous proof that the complete:

**GMMU → Channel → GPFIFO → Copy Engine**

pipeline is executing GPU commands under macOS.

---

# Display Support

Once basic command submission is proven, development will move toward making macOS actually use the GPU for display.

Planned stages include:

```text
GPU command execution
        ↓
Display engine
        ↓
Modesetting
        ↓
Framebuffer
        ↓
IOFramebuffer
        ↓
WindowServer
```

Getting the GPU listed by macOS is not considered display support.

The project will only claim working display output once the RTX 3060 Ti is actually responsible for scanout/framebuffer operation.

---

# Metal

Metal support is a long-term objective.

A complete Metal implementation requires considerably more than exposing the GPU to macOS.

Eventually this would require components such as:

```text
Kernel driver
     │
     ├── Memory management
     ├── Command submission
     ├── Synchronization
     ├── GPU channels
     └── Fault handling
            │
            ▼
Userspace GPU driver
            │
            ├── Metal device
            ├── Resource management
            └── Shader pipeline
                    │
                    ▼
                  Metal
```

The project is therefore intentionally progressing from the lowest layers upward.

---

# Development Philosophy

Every milestone is validated independently before moving forward.

The project avoids treating partial progress as full GPU support.

For example:

```text
GPU detected              ≠ GPU acceleration

VRAM detected             ≠ VRAM management

GSP running               ≠ GPU commands executing

GMMU accepted             ≠ GPU VA translation proven

System Information entry  ≠ display support

Framebuffer                ≠ Metal acceleration
```

Hardware results are separated from offline simulations and inferred behavior.

---

# Safety

Low-level GPU development can easily:

- freeze macOS
- trigger kernel panics
- corrupt GPU state
- leave GSP/FWSEC state active across warm reboots
- produce invalid MMIO accesses
- cause GPU faults

For this reason, new hardware stages are generally tested after a **complete power-off and cold boot**.

Experimental builds should only be used by people who understand the risks involved.

---

# References

The project studies publicly available documentation and open-source implementations, including:

- NVIDIA Open GPU Kernel Modules
- NVIDIA public SDK headers
- Nouveau / NVKM
- Linux DRM
- Apple IOKit documentation
- Apple Metal documentation

These projects are used as technical references for understanding NVIDIA hardware and driver architecture.

---

# Project Goal

The short version:

```text
Make an RTX 3060 Ti actually work on modern macOS.
```

The longer journey:

```text
PCI
 ↓
MMIO
 ↓
FWSEC
 ↓
GSP-RM
 ↓
VRAM
 ↓
GMMU
 ↓
Channel
 ↓
GPFIFO
 ↓
GPU Engines
 ↓
Display
 ↓
WindowServer
 ↓
Acceleration
 ↓
Metal
```

Several of these layers are already working.

The rest is under active development.

---

## Status

🚧 **Work in progress**

This repository documents the development process as the RTX 3060 Ti moves from being an unsupported PCI device toward becoming a usable GPU under modern macOS.
```
