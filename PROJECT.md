# QlsOS (Quick Light Secure OS) — about the project

A hand-written educational operating system: a bootloader, a kernel and a command
shell built from scratch, with no third-party libraries. It runs on two
architectures — **x86 (32-bit)** and **ARM (Cortex-A15, QEMU `virt`)** — and
is launched in the QEMU emulator.

## New direction (plan changed, target v1.0.0)

The old plan (Phase 3: POSIX-style processes — exec, fork, wait, signals, pipes) is **dropped**.
QlsOS is being turned into a **capability microkernel**: small, safe, easy to understand, with fast
program start. Target: **v1.0.0**.

Rules of the new design:
- All communication goes through **channels and handles** (capabilities). **No PIDs and no signals.**
- A task sees only the handles it was given; rights can only be narrowed (`handle_dup`), never widened.
- The system lives in userspace: a System Dispatcher task hands out channels to services
  (console, keyboard, disk, file system); the shell becomes a ring3 program.
- Code is **64-bit clean** (`uintptr_t`, `size_t`, fixed-width types); v1.0.0 itself stays 32-bit
  x86, a 64-bit port comes later through the HAL.
- Every task is closed by a test in QEMU; tasks go strictly in order.

Task list (6 blocks, 38 tasks):
**A** preparation (roll back signals, task slot generations, per-task limits, clean page fault,
blocking sleep, baseline measurements, ARCHITECTURE.md v0, 64-bit audit and `make check64`) ·
**B** handles and channels · **C** spawn and task control (result channel instead of wait, task
control handle, revoke; then **fork and wait are deleted**) · **D** the system in userspace
(dispatcher, console/keyboard services, ring3 shell, `caps`, real-hardware test) ·
**E** disk and programs (ATA service, QlsFS directories, file protocol over channels, mini-libc,
utilities) · **F** release (measurement table, docs, ISO + installer, tag v1.0.0).

After v1.0.0: GUI (framebuffer, display server, windows as capabilities) · parallel `x86_64` port via the
HAL · SMP · POSIX layer / toolchain / WASM runtime · later orthogonal persistence, USB, networking.

## Where the project is now

Block **A (preparation) is done** and block **B (handles and channels) is in progress**:

| Task | What | Status |
|---|---|---|
| 1 | Signals rolled back (`SYS_KILL`, `SYS_SIGNAL`, `SYS_SIGRETURN`, `sigtest` are gone; the old state is the zip snapshot `QuaOS_9_oct_signals.zip` / tag `unix-13`) | done |
| 2 | Task slot generations: a task is named by `task_ref_t = (slot, generation)`, a stale reference is invalid (`slottest`) | done |
| 3 | Per-task limits: memory frames and handles; going over a limit is an error for that task only (`limtest`) | done |
| 4 | Clean page fault: any fault in ring3 kills only that task, the reason stays in its status (`faulttest`) | done |
| 5 | Blocking `sleep`: a sleeping task gets no CPU (`sleeptest`) | done |
| 6 | One general blocking mechanism `task_block(reason, arg)`; sleep and wait use it (`blocktest`) | done |
| 7 | Baseline measurements: boot time by stage + spawn time, written to `baseline.txt` (`bench`) | done |
| 8, 8a-8c | `ARCHITECTURE.md` v0 (the contract); 64-bit audit of shared code; `make ksize` (kernel size budget) and `make check64` | done |
| 9 | Handle table per task: install, lookup, rights (`handletest`) | done |
| 10 | `handle_close` and `handle_dup` (rights only shrink, generation goes up on close) | done |
| 11 | Channels: a pair of endpoints, fixed-size message slots (64 bytes, 16 per queue), close, peer death (`channeltest`) | done |
| 12 | Waiting on channels (blocking send/receive) | next |

Not done yet: channel syscalls for ring3 (the channel API is kernel-internal so far), handles sent in messages
(task 13), spawn with a result channel, task control handles, the userspace system.

**What is still legacy in the code:** `fork` and `waitpid` (and the `forktest` / `waittest` programs), PIDs and
the shell command `kill <pid>`. They are removed or replaced in tasks 18 (task control handle) and 21
(fork/wait deleted after spawn and the result channel exist). The ELF loader, ring3 and per-process
page directories stay and are the base of the new design. Signals no longer exist.

## What it is written in

| Language | Where | Size |
|---|---|---|
| **C** (freestanding, no libc) | the kernel: scheduler, file systems, shell, drivers | ~7,200 lines (43 files, x86 + ARM + shared) + ~1,700 lines of headers; user programs ~375 lines; host tests ~900 lines |
| **x86 assembly (NASM)** | bootloader, kernel entry, interrupts, context switching, ring3 | ~600 lines (7 files) |
| **ARM assembly (GAS, `.S`)** | startup, vector table, context switching | ~125 lines (3 files); x86 `.S`: `userbins.S`, `user/crt0.S` |
| **Linker script (`.ld`)** | kernel memory layout (x86 — at the 1 MB address, ARM — its own) | 2 files |
| **Make / Bash / Batch** | build and launch (`Makefile`, `build_and_run.sh`, `run.bat`, `run_wsl1.bat`) | — |

Build tools: `nasm`, `gcc -m32`, `ld` (x86); `arm-none-eabi-gcc` (ARM);
launched with `qemu-system-i386` / `qemu-system-arm`.

## What it can already do

- **x86 boot:** its own bootloader (stage1): A20, the BIOS memory map (E820),
  protected mode, relocating the kernel to 1 MB.
- **Memory:** a physical page manager (bitmap), paged memory (paging),
  its own `kmalloc`/`kfree` (free-list).
- **Interrupts and hardware (x86):** GDT/IDT, PIC, a 100 Hz PIT timer, PS/2 keyboard
  and mouse, VGA text mode with scrolling, COM1, CMOS RTC, PCI, ATA/IDE (PIO),
  ACPI (reboot / shutdown).
- **Multitasking:** a preemptive timer-driven round-robin scheduler, each
  task has its own kernel stack.
- **Processes:** ring3 (user mode), `int 0x80` system calls,
  a separate page directory per process; a crash of a user process does not
  kill the kernel.
- **fork()** *(legacy — to be deleted, task 21)*: `SYS_FORK` clones a ring3 process: private copy of the whole user area (eager copy, no COW yet) + a copy of the syscall trap frame, so the child returns from `int 0x80` with `eax = 0`. Also `SYS_GETPID`. Test: `exec forktest [multi|bomb]`.
- **wait()/waitpid()** *(legacy — to be deleted, task 21; replaced by a result channel)*: `SYS_WAITPID` (ebx = pid or -1, ecx = `int* status`, edx = options, `WNOHANG` supported). A finished process stays a zombie (slot + exit status) until its parent collects it; a parent that waits is BLOCKED (not scheduled, no CPU burn) and is woken by the child's exit. Status is POSIX-style (`WIFEXITED/WEXITSTATUS/WIFSIGNALED/WTERMSIG` in `user/usys.h`): `main`'s return value is the exit code; a fault or Esc in the shell kills the task with a termination reason (these are task reasons, not signals). Orphans (parent exited first) are re-parented to nobody and freed automatically. `-1` = no such child / bad pointer. Tests: `exec waittest` (and `exec waittest hang`), `exec forktest [multi|bomb]`; `ps` shows the parent id and the `waiting` state.
- **Tasks and limits:** task identity is `(slot, generation)`; per-task limits on memory frames and handles; a fault or an
  over-limit only ever hurts the one task; one general blocking mechanism (`task_block`) behind `sleep` and `waitpid`.
- **Handles and channels (kernel API, tasks 9-11):** a per-task handle table (`handle_install/lookup/close/dup`, rights only
  shrink, a closed number never works again) and channels with bounded queues. Not reachable from ring3 yet (no syscalls
  for them until the next tasks). Tested in QEMU (`handletest`, `channeltest`) and on the host under ASan/UBSan.
- **Measurements:** `bench` times the boot stages and the program start and writes `baseline.txt` — the numbers v1.0.0 is compared with.
- **Build checks:** `make check64` (shared code compiles as x86-64 with `-Wall -Werror`, no `arch/` includes or `#if __i386__`
  in shared code), `make ksize` (kernel code+data against the size budget: 96 KiB ceiling now, 64 KiB target for v1.0.0).
- **Bootloader:** reads the kernel in two `int 0x13` chunks (up to 128 KB); `make` fails above that.
- **ELF loader:** `exec <file> [args]` runs static ELF32 programs in ring3
  (argc/argv, 16 KB stack with a guard page, validated input); `elfinfo`. See `docs/elf-loader.md`.
- **File systems:** a VFS layer, a RAM disk, initrd, and its own **QlsFS**
  (flat, up to 64 files, name up to 23 characters) on an ATA disk.
- **Shell:** a line editor (cursor, history, Home/End/Delete), colored output. `help` is split into topics (`help`, `help <topic>`, `help all`). Commands: `help`, `clear`, `echo`,
  `uptime`, `bench`, `date`, `dmesg`, `mount`, `ls`, `write`, `cat`, `rm`, `diskdump`, `format`, `exec`, `elfinfo`,
  `ps`, `kill`, `proctest`, `frames`, `lspci`, `mouse`, `reboot`, `shutdown`; self-tests: `faulttest`, `sleeptest`,
  `blocktest`, `slottest`, `limtest`, `handletest`, `channeltest`, `ptest`, `disktest`.
- **ARM port:** PL011 UART, GICv2, the ARM generic timer, PSCI (reset/off),
  an interactive shell and multitasking. The console is right in the terminal (UART).

## What is not there yet

- ARM has no paging/PMM, ring3 or system calls (stubs).
- QlsFS: no directories, and deleting a file does not free space on the disk.
- No networking, graphics mode or sound.

## The architecture in one picture

```
boot/boot.asm (x86 only)  ->  kernel/arch/<arch>/  (entry + drivers)
                                        |
                                        v
                    kernel/src/kernel.c : kernel_main()   (common code)
                    shell, task, kheap, kprintf, VFS, QlsFS
                                        |
                          hal_* interface (kernel/hal/hal.h)
                                        |
                         x86 (kernel/arch/x86)  |  ARM (kernel/arch/arm)
```

The main rule: the common code in `kernel/src/` does not touch hardware directly, only
through the functions from `kernel/hal/hal.h`. See `docs/architecture.md` and
`docs/hal-design.md` for details.

## Folder structure

```
boot/                 the x86 bootloader
kernel/src/           common code (shell, scheduler, heap, file systems)
kernel/include/       common headers
kernel/hal/hal.h      the HAL interface
kernel/arch/x86/      everything x86-specific
kernel/arch/arm/      everything ARM-specific
user/                 user programs (ELF) + crt0/linker script
tests/                host-side tests (ELF loader, handles, channels; `make test`, `make test-handle-host`, `make test-channel-host`), QEMU bench test, `make check64` canary
ARCHITECTURE.md       the v0 architecture contract for the capability microkernel (read this first)
docs/                 architecture.md (how the code is split), hal-design.md, elf-loader.md, history.md (old README)
```

## How to build and run

| Method | File | When |
|---|---|---|
| WSL2 + a QEMU window from WSL | `run.bat` (`run.bat arm` — ARM) | WSL2 works (needs "Virtual Machine Platform") |
| WSL1 for building + QEMU for Windows | `run_wsl1.bat` | WSL2 is unavailable; needs `qemu-system-i386.exe` for Windows |

Manually in WSL: `make`, `make run`, `make run-headless`, `make arm`, `make run-arm`.

## Run logs

The `logs/` folder next to the scripts: `last-build.log` (the full output),
`last-errors.log` (errors only), `last-qemu.log` (QEMU's internal log),
`last-arm-serial.log` (kernel output to the UART, ARM only), `run-bat.log`
(the actions of the `.bat` itself). The 20 most recent runs are kept.

## Roadmap

The old Phase 3 table (ELF → exec → fork → wait → signals → IPC) is **obsolete**; see
"New direction" above. Done and kept: the ELF loader (`exec` shell command), ring3 and per-process
address spaces, preemptive scheduler. Done in the new plan: block A (tasks 1-8c) and tasks 9-11 of block B
(see the table above). Next: task 12 (waiting on channels).

## Development history

Stages so far: Stage 6 (HAL + ARM + ACPI/PSCI + documentation) → ATA/IDE, RTC, PCI, a terminal with colors
and scrolling → QlsFS v0 → Phase 0 (mouse, VFS) → Phase 1 (open/read/write/close, RAM disk, initrd) →
Phase 2.1 (preemptive multitasking) → Phase 2.2 (separate process address spaces) → rename FlowOS → QlsOS (the names QuaOS and QuasarOS turned out to be taken),
ELF loader, `fork`, `waitpid`, signals (the `unix-13` state) → new plan (capability microkernel): block A, then handles (9, 10) and channels (11).

The source is hosted on GitHub: https://github.com/infernogigabyte-web/QlsOS .
Working snapshots between plan tasks are kept as zip files.

More about the early stages and troubleshooting: `docs/history.md`.
