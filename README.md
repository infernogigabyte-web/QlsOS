# QlsOS (Quick Light Secure OS)

**QlsOS** (*Quick Light Secure OS*) is a small operating system written from scratch: bootloader, kernel and shell, no third-party
libraries. Runs on **x86 (32-bit)** and **ARM (Cortex-A15, QEMU `virt`)** inside QEMU.
It is a hobby/educational project — the kernel is fully its own (Linux/WSL is only used as the build tool).

> Project summary and status: [`PROJECT.md`](PROJECT.md) · Architecture contract: [`ARCHITECTURE.md`](ARCHITECTURE.md) · Code layout: [`docs/architecture.md`](docs/architecture.md)

## Where the project is going

The plan changed: QlsOS is becoming a **capability microkernel** (target **v1.0.0**) — all
communication through channels and handles, no PIDs and no signals, services (console, keyboard, disk,
file system) and the shell as userspace tasks, code ready for a 64-bit port.
**Status:** signals are already gone; handles (tasks 9-10) and channels (task 11) exist as a kernel API and are tested,
the next step is waiting on channels (task 12). `fork` / `waitpid` are **legacy** and will be removed (task 21).
Details and the task list: [`PROJECT.md`](PROJECT.md).

## What works today

- **x86:** own bootloader, protected mode, paging, physical memory manager, heap, preemptive
  round-robin scheduler, ring3 processes with their own page directories, `int 0x80` syscalls,
  PS/2 keyboard and mouse, VGA text console with scrollback, ATA/IDE, RTC, PCI, ACPI reboot/shutdown.
- **Processes:** ELF32 loader (`exec`), per-task limits, clean fault handling (a crash kills only that task), blocking `sleep`; `fork` and `waitpid` are legacy and will be replaced by spawn + a result channel.
- **Handles and channels:** per-task handle table and channels with bounded queues (kernel API, tested in QEMU and on the host; not yet available to ring3 programs).
- **Storage:** VFS layer, RAM disk, initrd and **QlsFS** (flat file system on the ATA disk).
- **ARM:** UART shell, GICv2, generic timer, PSCI reset/off, multitasking (no paging/ring3 yet).

## Requirements

- Linux or **Windows + WSL**
- `nasm`, `gcc` (with `-m32` support), `binutils`, `make`, `qemu-system-i386`
- ARM port only: `gcc-arm-none-eabi`, `qemu-system-arm`

```
sudo apt install nasm build-essential gcc-multilib qemu-system-x86
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi qemu-system-arm   # ARM only
```

## Quick start

**Windows**

| Situation | Run |
|---|---|
| WSL2 works (WSLg shows the QEMU window) | `run.bat` (ARM: `run.bat arm`) |
| Only WSL1 (no "Virtual Machine Platform") | `run_wsl1.bat` — builds in WSL1, runs QEMU for Windows |

Logs go to the `logs/` folder next to the scripts (`last-build.log`, `last-errors.log`, `last-qemu.log`, ...).

**Linux / inside WSL**

```
make                # build disk.img (+ disk_data.img on first run)
make run            # QEMU with a window
make run-headless   # QEMU without a window, serial log in the console
make arm            # build the ARM kernel
make run-arm        # run it in qemu-system-arm -M virt (UART in the terminal)
make test           # host-side tests of the ELF loader (no QEMU needed)
make test-handle-host    # handle table on the host (ASan/UBSan)
make test-channel-host   # channels on the host (ASan/UBSan)
make check64        # shared code must be 64-bit clean
make ksize          # kernel size against the budget
make clean
```

## First steps in the shell

```
help                  list of commands
help tests            list of self-tests
ps                    tasks (state, parent id, CPU ticks)
exec hello            run a user program (ELF) in ring3
exec forktest         fork() test (legacy)   exec waittest   waitpid() test (legacy)
faulttest             every crash kills only the process
sleeptest  blocktest  blocking sleep and the general blocking mechanism
slottest   limtest    task slot generations, per-task limits
handletest channeltest  handle table and channels
bench                 boot/spawn baseline, written to baseline.txt
write note hello      create a file on QlsFS  cat note       print it
```

`exec` programs available out of the box: `hello`, `crash`, `forktest`, `waittest`, `sleeper`, `nop`.
Esc kills the running program. Full command list: `help`.

## Repository layout

```
boot/               x86 bootloader
kernel/src/         arch-independent code (shell, scheduler, heap, VFS, QlsFS, ...)
kernel/include/     arch-independent headers
kernel/hal/hal.h    the HAL contract between common code and the architectures
kernel/arch/x86/    everything x86-specific (drivers, paging, ring3, syscalls, ELF loader)
kernel/arch/arm/    everything ARM-specific
user/               user programs (C, built as static ELF) + crt0 + linker script + usys.h
tests/              host-side tests (ELF loader, user stack builder, handles, channels) under ASan/UBSan
ARCHITECTURE.md     the v0 architecture contract of the capability microkernel
docs/               architecture.md, hal-design.md, elf-loader.md, history.md
```

## Limits to know about

- The bootloader loads at most **128 KB** of kernel (`make` fails loudly above that).
- QlsFS is flat: up to 64 files, names up to 23 characters; `rm` does not reclaim disk space.
- ARM has no paging, PMM, ring3 or syscalls yet.
- If QEMU "stops working" after several launches, a previous QEMU is probably still holding
  `disk.img` — see the troubleshooting section in [`docs/history.md`](docs/history.md)
  (`wsl --shutdown` from Windows fixes it).
