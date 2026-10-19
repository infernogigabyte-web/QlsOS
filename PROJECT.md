# QlsOS (Quick Light Secure OS) — Project Status & Roadmap

QlsOS is an independent, capability-based microkernel operating system written from scratch with no third-party libraries. It supports native execution on two architectures — x86 (32-bit execution) and ARM (Cortex-A15, QEMU virt machine platform) — simulated via the QEMU emulator environment.

---

## Technical Direction: Capability Microkernel (Target v1.0.0)

The legacy monolithic design (POSIX-style processes including traditional `exec`, `fork`, `wait`, UNIX signals, and anonymous pipes) has been officially deprecated. QlsOS is being engineered into a robust, high-performance capability microkernel: compact, mathematically secure, isolated, and focused on ultra-fast userspace task spawning.

### Microkernel Core Engineering Rules:
1. **Object Capabilities:** All task synchronization and communications are performed via isolated communication channels and kernel-managed handles. Process IDs (PIDs) and UNIX signals are entirely removed from the new architecture.
2. **Strict Right Narrowing:** A running task only has visibility over the specific handle table it was explicitly granted during spawn operations. Access rights can only shrink (`handle_dup` narrowing), never expand.
3. **Userspace System Architecture:** The entire operating system infrastructure is pushed to Ring3. A dedicated System Dispatcher task manages and routes communication channels to independent microservices (Console, Keyboard, Disk, and File System layers). The shell layer executes strictly as a standalone Ring3 user application.
4. **64-bit Clean Codebase:** All platform-agnostic subsystems are audited to be 64-bit safe (`uintptr_t`, `size_t`, and explicit width types). While the v1.0.0 core targets 32-bit x86 hardware execution, a parallel 64-bit migration path is maintained via the rigid Hardware Abstraction Layer (HAL).
5. **Continuous Integration & Testing:** Every incremental task block is verified via automated QEMU host tests executing in strict chronological order.

---

## Roadmap & Milestone Task Blocks

The core development path towards the commercial v1.0.0 release is structured into 6 sequential engineering blocks (38 distinct tasks).

### Block A: Microkernel Preparation
*   Task 1: Roll back legacy UNIX signals (`SYS_KILL`, `SYS_SIGNAL`, `SYS_SIGRETURN` are fully deleted).
*   Task 2: Task slot generations implementation (Tasks are tracked via `task_ref_t = (slot, generation)` to invalidate stale references).
*   Task 3: Per-task hardware limits (Strict resource quotas on physical memory frames and handle tables; overflow faults terminate the task only).
*   Task 4: Isolated Page Fault Handling (Ring3 memory violations kill the offending task without inducing kernel panics; termination codes are safely captured in the task status structure).
*   Task 5: Preemptive Blocking Sleep (Yields CPU clock cycles instantly; sleeping tasks consume zero scheduling overhead).
*   Task 6: Unified Microkernel Blocking Primitives (`task_block(reason, arg)` abstraction implemented for sleep and synchronization frameworks).
*   Task 7: System Performance Benchmarking (Boot timeline tracking and spawn latency metrics written directly to `baseline.txt`).
*   Task 8, 8a-8c: System Audit & Contract Enforcement (Formalization of `ARCHITECTURE.md v0`, end-to-end 64-bit data cleanliness verification, compilation budget verification via `make ksize` and `make check64`).

### Block B: Handles and Channels (Current Active Phase)
*   Task 9: Per-Task Handle Tables (Implementation of `handle_install`, `handle_lookup`, access permissions mapping). -> *Status: Completed*
*   Task 10: Capability Object Management (`handle_close` tracking and `handle_dup` permission restriction loops). -> *Status: Completed*
*   Task 11: Microkernel Bounded Channels (Paired endpoints managing fixed-size 64-byte message structures with a 16-slot internal queue, handling abrupt peer destruction cleanly). -> *Status: Completed*
*   Task 12: Advanced Channel Synchronization (Blocking send and receive kernel routines). -> **Status: NEXT ACTIVE TASK**
*   Task 13: Capability Transfer in IPC Messages (Passing handles over channels with identical or decreased access rights). -> *Status: Pending*
*   Task 14: Channel Stress-Testing Infrastructure (Ping-pong IPC validation, bounded queue overflow stress tests, peer death tracking). -> *Status: Pending*
*   Task 15: Automated Object Lifetime Cleanup (Verification of automatic physical frame reclamation upon final handle closure). -> *Status: Pending*

### Block C: Task Spawning & Capability Control
*   Tasks 16-21: Implementation of `SYS_SPAWN` from user space, task lifecycle management via result channels instead of legacy `wait`, dedicated Task Control Handles, implementation of capability revocation, deletion of legacy `fork`/`waitpid` code and test manifests.

### Block D: Userspace System Realization
*   Tasks 22-28: Initialization of the Ring3 System Dispatcher, extraction of VGA Console and PS/2 Keyboard drivers into independent userspace tasks, executing the interactive shell in Ring3, resource capability inspections via `caps`, and regression testing under extreme conditions.

### Block E: Storage Subsystems & User Utilities
*   Tasks 29-34: Translating the ATA/IDE disk driver into a userspace service, expanding QlsFS to manage real subdirectories and block reclamation upon removal operations, implementing file access layers over IPC channels, building a lightweight standalone `mini-libc` runtime, and migrating core user utilities (`ls`, `cat`, `echo`, `ps`, `mkdir`, `rm`).

### Block F: v1.0.0 Production Release
*   Tasks 35-38: Final metrics collection, comprehensive architectural documentation updates, building production bootable ISOs and automated installers, and tagging the stable v1.0.0 microkernel release.

---

## Long-Term Post-v1.0.0 Evolution
*   **Userspace GUI Engine:** Standalone framebuffer virtualization, independent display server architecture, managing desktop windows as distinct capability handles.
*   **x86_64 Symmetric Symmetrical Porting:** Bringing up native 64-bit AMD64 execution frameworks via the HAL.
*   **Symmetric Multiprocessing (SMP):** Spawning multi-core scheduling threads.
*   **Advanced Runtimes:** Comprehensive POSIX compatibility translation layer, native compiler toolchain integration, and a production WebAssembly (WASM) sandboxed runtime environment.
*   **Hardware Expansion:** Orthogonal persistence, USB stack mapping over channel streams, and high-performance network stacks.

---

## Technical Stack & Codebase Metrics

### Language Allocation
*   **C (Freestanding, no compiler libc dependency):** Powering the scheduling algorithms, virtual file system, core shell primitives, and driver code blocks. (~7,200 lines across 43 source files for x86/ARM/Shared logic + ~1,700 lines of kernel headers).
*   **x86 Assembly (NASM):** Low-level stage1 bootloader code, context switching paths, interrupt gates, and Ring3 syscall entry traps. (~600 lines).
*   **ARM Assembly (GAS):** System initialization blocks, exception vector mappings, and thread context switching layout. (~125 lines).

### Toolchain Infrastructure
*   **Compilers & Linkers:** `nasm`, `gcc -m32`, native GNU `ld` (for x86 targets); `arm-none-eabi-gcc` (for ARM targets).
*   **Emulation Environment:** `qemu-system-i386` and `qemu-system-arm`.
*   **Build Optimization Rules:** Strict kernel text+data limits enforced through automated budget compliance tools (`make ksize` target limits: 96 KiB temporary maximum, 64 KiB final target).
