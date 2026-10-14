# QuaOS — about the project

A hand-written educational operating system: a bootloader, a kernel and a command
shell built from scratch, with no third-party libraries. It runs on two
architectures — **x86 (32-bit)** and **ARM (Cortex-A15, QEMU `virt`)** — and
is launched in the QEMU emulator.

## New direction (plan changed, target v1.0.0)

The old plan (Phase 3: POSIX-style processes — exec, fork, wait, signals, pipes) is **dropped**.
QuaOS is being turned into a **capability microkernel**: small, safe, easy to understand, with fast
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
**E** disk and programs (ATA service, QuaFS directories, file protocol over channels, mini-libc,
utilities) · **F** release (measurement table, docs, ISO + installer, tag v1.0.0).

After v1.0.0: GUI (framebuffer, display server, windows as capabilities) · parallel `x86_64` port via the
HAL · SMP · POSIX layer / toolchain / WASM runtime · later orthogonal persistence, USB, networking.

**What this means for the current code (state at the `unix-13` snapshot):**
`fork`, `waitpid`, signals (`kill`, `signal`, `sigreturn`), PIDs and the shell command `kill <pid>` are
**legacy**. Signals are rolled back first (task 1; the snapshot is kept as git tag `unix-13`);
fork/wait are removed in task 21 after spawn and the result channel exist. The ELF loader,
ring3 and per-process page directories stay and are the base of the new design.
The descriptions of these legacy parts below are kept because they describe the code as it is today.

## What it is written in

| Language | Where | Size |
|---|---|---|
| **C** (freestanding, no libc) | the kernel: scheduler, file systems, shell, drivers | ~5,700 lines (39 files) + ~1,450 lines of headers; user programs ~560 lines |
| **x86 assembly (NASM)** | bootloader, kernel entry, interrupts, context switching, ring3 | ~600 lines (7 files) |
| **ARM assembly (GAS, `.S`)** | startup, vector table, context switching | ~130 lines (3 files); x86 `.S`: `userbins.S`, `user/crt0.S` |
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
- **wait()/waitpid()** *(legacy — to be deleted, task 21; replaced by a result channel)*: `SYS_WAITPID` (ebx = pid or -1, ecx = `int* status`, edx = options, `WNOHANG` supported). A finished process stays a zombie (slot + exit status) until its parent collects it; a parent that waits is BLOCKED (not scheduled, no CPU burn) and is woken by the child's exit. Status is POSIX-style (`WIFEXITED/WEXITSTATUS/WIFSIGNALED/WTERMSIG` in `user/usys.h`): `main`'s return value is the exit code; a fault kills with SIGSEGV/SIGFPE/SIGILL, Esc in the shell with SIGKILL. Orphans (parent exited first) are re-parented to nobody and freed automatically. `-1` = no such child / bad pointer. Tests: `exec waittest` (and `exec waittest hang`), `exec forktest [multi|bomb]`; `ps` shows the parent id and the `waiting` state.
- **Signals (kill)** *(legacy — to be rolled back, task 1)*: `SYS_KILL`, `SYS_SIGNAL`, `SYS_SIGRETURN`; shell command `kill [-sig] <pid>`. Per task: pending set, blocked set, one handler per signal (`SIG_DFL`/`SIG_IGN`/ring3 address). Default action kills at once (status = "killed by sig", `SIGCHLD` is ignored); `SIGKILL` cannot be caught/ignored/blocked; hardware faults still always kill (not catchable yet). A caught signal is queued and delivered on the way back to ring3 (end of a syscall or of a timer/keyboard IRQ, so even a busy loop gets it): a `sigframe` with a trampoline is built on the user stack, the handler runs as `void h(int signo)` with its own signal blocked, and `sigreturn` restores everything (eflags sanitised). A sleeping `sleep()` returns early (result = ticks left), a blocked `wait()` returns `-2` (EINTR). `fork` copies the handlers; the parent gets `SIGCHLD` when a child ends (if it has a handler). Not there yet: signal masks syscalls (`sigprocmask`), `SA_RESTART`, catching fault signals, process groups. Test: `exec sigtest`.
- **Bootloader:** reads the kernel in two `int 0x13` chunks (up to 128 KB); `make` fails above that.
- **ELF loader:** `exec <file> [args]` runs static ELF32 programs in ring3
  (argc/argv, 16 KB stack with a guard page, validated input); `elfinfo`. See `docs/elf-loader.md`.
- **File systems:** a VFS layer, a RAM disk, initrd, and its own **QuaFS**
  (flat, up to 64 files, name up to 23 characters) on an ATA disk.
- **Shell:** a line editor (cursor, history, Home/End/Delete), colored output. Commands: `help`, `clear`, `echo`,
  `uptime`, `date`, `lspci`, `mouse`, `mount`, `disktest`, `diskdump`, `format`, `ls`, `write`, `cat`, `rm`,
  `ps`, `ptest`, `proctest`, `frames`, `exec`, `elfinfo`, `kill`, `reboot`, `shutdown`, `dmesg`.
- **ARM port:** PL011 UART, GICv2, the ARM generic timer, PSCI (reset/off),
  an interactive shell and multitasking. The console is right in the terminal (UART).

## What is not there yet

- ARM has no paging/PMM, ring3 or system calls (stubs).
- QuaFS: no directories, and deleting a file does not free space on the disk.
- No networking, graphics mode or sound.

## The architecture in one picture

```
boot/boot.asm (x86 only)  ->  kernel/arch/<arch>/  (entry + drivers)
                                        |
                                        v
                    kernel/src/kernel.c : kernel_main()   (common code)
                    shell, task, kheap, kprintf, VFS, QuaFS
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
tests/                host-side tests of the ELF loader (`make test`)
docs/                 architecture.md, hal-design.md, elf-loader.md, history.md (old README)
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
address spaces, preemptive scheduler. Next: block A, task 1 (roll back signals).

## Development history

Committed in git (12 commits): Stage 6 (HAL + ARM + ACPI/PSCI + documentation) → ATA/IDE, RTC, PCI, a
terminal with colors and scrolling → QuaFS v0 → Phase 0 (mouse, VFS) → Phase 1 (open/read/write/close,
RAM disk, initrd) → Phase 2.1 (preemptive multitasking) → Phase 2.2 (separate process address spaces).

**Not committed yet:** the rename FlowOS → QuaOS (FlowFS → QuaFS), the ELF loader, `fork`, `waitpid`,
signals (`unix-13` state), `user/`, `tests/`, and the documentation. Task 1 of the new plan expects the `unix-13` tag, so commit first. Run `git add -A && git commit` to record them.

More about the early stages and troubleshooting: `docs/history.md`.
