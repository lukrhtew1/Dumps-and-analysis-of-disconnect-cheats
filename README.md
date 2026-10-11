# Dumps-and-analysis-of-disconnect-cheats

**Website:** disconnectcheats.com

---

## Overview

This *cheat* provider is very well known in the community of specifically Rust cheating and has been on the market for 4+ years, and has experienced a large set of issues along its timeline.

Today we bring the full dump and analysis of their software including:

- Loader analysis
- Bypass analysis
- Cheat itself analysis

We have also provided all of the information in this repo, over to anti-cheats so they can dissect further and learn from this software.

---

## What We Have Found

### Architecture

Loader wise, this loads a vulnerable driver (cormem.sys) which has been dumped from disconnect a while ago, and they still use it.

It is used primarily for physical read/write to prep their bypass.

**Steam injected:** The cheat itself (which lives a dll, and is also dumped to system32 before being injected into steam) is injected into a specific steam sub-process, steamwebhelper.exe

**IPC:** Shared memory between their components (we will unravel more about this soon)

We found that, once we closed steam with it injected, and we experienced a BSOD.

- Bug check: MEMORY_MANAGEMENT (0x1A) subtype: 0x61948
- Process: steamwebhelper.exe
- Path: process had exited -> MmCleanProcessAddressSpace -> VAD/PTE was torn down -> MiDerferenceIoPages
- Bucket: 0x1a_61948_nt!MiDereferenceIoPages
- Args: P2/P4 looked like a PFN (0x9c4d2)

The pattern means pages were mapped inside of steamwebhelper in a non-standard way (phys/pte/pfn refs) on exit, MM walked those VADs, the PFN refcount is wrong, and therefore bluescreened.

The BSOD was evidence that:

- Their lasting access was tied to steamwebhelper
- A kernel phys/pte mapping was involved, usermode teardown alone can not clean it

This lines up with the vulnerable driver planted a phys window into steamwebhelper, and leave steam with bad mappings -> kill steam -> MiDereferenceIoPages dies.

---

## Vulnerable Driver

**Driver:** Teledyne DALSA Sapera `CORMEM.SYS`

**Dispatch:** Opens `\\Device\\PhysicalMemory` and maps via ZwOpenSection/ZwMapViewOfSection

### IOCTL Codes

| Code | Name | Role |
|------|------|------|
| `0x22200C` | **MAPBUFFER** | Map physical range -> usermode VA |
| `0x222010` | **UNMAPBUFFER** | Remove the view |

Disconnects runtime traffic we captured when intercepting cormems ioctls were mostly just mapbuffer + unmapbuffer

Around 482 unique phys targets, very hot pfns hit hundreds of times, some of the runs walk page aligned ranges.

### Caller

IOCTL caller was their loader process.

### Takeaway

1. Load cormem
2. Spam map/unmap to walk / patch what they need
3. Plant kernel r/w into steamwebhelper
4. Unload

This also matches the steam close BSOD, leftover bad PFN mappings into steam host after cormem is gone.

---

## Obfuscation

### Loader

- MBA on control / data paths
- Return continuation style flow (just ret-based chaining) they do not utilize any form of virtualization
- Looks like a base ollvm style obfuscation straight off the bat

### User-Mode Cheat

- Very thin ollvm transforms (control-flow flattening and MBA noise)
- Still very reversable in any form of dissasembler
- All imports are in place with xrefs (imports are not hidden)
- Again, no vm layer

---

> **WE ARE NOT AFFILIATED WITH ANY ANTI-CHEAT VENDORS, BUT WE DO SEND INFORMATION OFF TO ANTI-CHEATS**
