# QlsOS (Quick Light Secure OS)

QlsOS is a fast, ultra-lightweight, and secure capability-based microkernel operating system written completely from scratch (including custom bootloader, kernel, and userspace environment) with zero third-party dependencies. It is engineered specifically for next-generation IoT and embedded devices.

Project Summary & Roadmap: `PROJECT.md` · Microkernel Architecture Contract: `ARCHITECTURE.md` · Code Layout Guide: `docs/architecture.md`

---

## Commercial Vision & Architecture Shift

QlsOS is actively transitioning towards a full capability-based microkernel (Targeting v1.0.0) to solve critical Linux security and overhead issues in modern smart devices:
*   Zero PIDs & Signals: All task communication is handled strictly through isolated channels and secure capability handles.
*   True Microkernel Isolation: All core system services (VGA Text/Graphics Console, PS/2 Keyboard/Mouse, Storage Drivers, and File Systems) are moved out of the kernel space into isolated ring3 userspace tasks.
*   64-bit Clean Architecture: The entire shared codebase is engineered to be 64-bit safe (`uintptr_t`, explicit width types, `make check64` enforcement) while remaining compatible with 32-bit environments for v1.0.0.

### Current Transition Status:
*   `LEGACY`: POSIX-like `fork`/`waitpid` and primitive UNIX signals are deprecated and being fully stripped down.
*   `WORKING`: Core microkernel handle tables, synchronous/asynchronous channel IPC, and isolated ring3 memory page directories are implemented and verified via automated host and QEMU testing.

---

## Key Features & Capabilities

### x86 (32-bit Native / 64-bit Clean Ready)
*   Boot & Core: Custom stage1 bootloader, protected mode execution, hardware abstraction layer (HAL contract).
*   Memory Management: Dynamic page-table mapping, physical page bitmap manager, resilient kernel heap allocator (`kmalloc`/`kfree`).
*   Scheduler: Preemptive round-robin scheduler with proper blocking sleep mechanisms (tasks yield CPU resources instantly when blocked).
*   Isolation & Fault Tolerance: True Ring3 process space. CPU exceptions and application crashes are fully sandboxed—a user space fault terminates ONLY the faulty task, keeping the core microkernel running.
*   Drivers & Storage: Native PS/2 Keyboard & Mouse drivers, VGA text console with custom scrollback, IDE/ATA disk access, PCI scanning, ACPI power state management. Virtual File System (VFS) with Ramdisk support and QlsFS (a custom flat file system layout mapped to physical disk).

### ARM (Cortex-A15 Platform / QEMU Virt)
*   Integrated UART serial shell interface.
*   GICv2 interrupt handling and generic hardware timer tracking.
*   PSCI hardware reset and shutdown routines.
*   Multitasking ready (Paging and Ring3 isolation layers are currently under active development).

---

## Environment Setup & System Requirements

To build and compile the QlsOS toolchain, you need a Linux environment (Native or Ubuntu via Windows Subsystem for Linux - WSL).

### Required Toolchain Packages:
```bash
# Core x86 build tools and compilers
sudo apt update && sudo apt install nasm build-essential gcc-multilib qemu-system-x86

# ARM development tools (Required for the ARM port only)
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi qemu-system-arm
```

---

## Quick Start & Build System

### Execution on Windows (WSL)
Depending on your local virtualization setup, choose the correct execution script from the root folder:
*   WSL2 with GUI support (WSLg): Run `run.bat` (For ARM emulation, use: `run.bat arm`).
*   WSL1 Fallback Mode: Run `run_wsl1.bat`. This builds the system inside the WSL1 environment and securely passes the artifact to native Windows QEMU.
*   *Note: Compilation and execution logs are dynamically written to the `logs/` directory.*

### Automated Build Commands (Linux / WSL Terminal)
```bash
make                  # Compiles kernel and builds disk.img (+ data images)
make run              # Launches QlsOS target system inside a windowed QEMU instance
make run-headless     # Launches QlsOS without a GUI, piping raw serial output to the host console
make arm              # Compiles the ARM standalone architecture build
make run-arm          # Launches the ARM build inside qemu-system-arm (virt machine target)
make test             # Executes host-side automated unit tests for the ELF loader structure
make test-handle-host # Stress-tests the microkernel handle tables on the host machine using ASan/UBSan
make test-channel-host# Stress-tests the IPC communication channels on the host machine using ASan/UBSan
make check64          # Validates code syntax with -m64 -fsyntax-only to guarantee 64-bit pointer cleanliness
make ksize            # Compares compiled kernel size against the strict storage budget
make clean            # Flushes build caches and object structures
```

---

## Native Microkernel Shell Commands

Boot into the interactive QlsOS command-line interface to execute the following core primitives:
*   `help` / `help <topic>`: Show organized technical topics (`basic`, `files`, `programs`, `system`, `tests`).
*   `ps`: Dumps active tasks, process status, hierarchy mappings, and exact CPU ticks.
*   `exec <elf_program>`: Executes a static ELF32 binary within isolated ring3 execution rights.
*   `write <file> <text>` / `cat <file>`: Creates data sequences or prints file buffers directly from QlsFS.
*   `faulttest` / `limtest` / `slottest`: Triggers deliberate memory page faults, execution limits, and recycling loops to demonstrate microkernel resilience.

---

## Repository Directory Map
*   `boot/` — Low-level x86 assembler bootloader.
*   `kernel/src/` — Architecture-agnostic microkernel source code (VFS, QlsFS, scheduler, IPC channels).
*   `kernel/include/` — Global internal core kernel headers.
*   `kernel/hal/hal.h` — The strictly defined Hardware Abstraction Layer API contract.
*   `kernel/arch/x86/` — Architecture-specific layers for Intel/AMD (paging, ring3, hardware interrupts, ELF loaders).
*   `kernel/arch/arm/` — Architecture-specific layers for ARM boards and platforms.
*   `user/` — Standard ring3 applications (built with custom crt0 entry blocks and standalone syscall wrappers).
*   `tests/` — Automated test harnesses compiled with host signatures (ASan/UBSan).

---

## License & Contributor License Agreement (CLA)

QlsOS is open-source software released under the **Mozilla Public License 2.0 (MPL 2.0)**. 

### Contribution Rules & Commercial Grant:
By submitting code, bug fixes, or pull requests to the QlsOS project repository, you explicitly agree to the following terms:
1. Your contributions are licensed under the terms of the MPL 2.0 license.
2. You grant the project owner a perpetual, worldwide, non-exclusive, sublicensable, royalty-free, and irrevocable license to use, modify, copy, adapt, bundle, distribute, and commercialize your contributions in any business application, software product, or hardware target worldwide (including commercial licensing models for embedded and IoT platforms) without any restriction or financial obligation.
