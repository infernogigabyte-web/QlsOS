# QlsOS — development notes and troubleshooting (old README)

This is the former `README.txt`, kept unchanged below the line for the history of the early stages
and the troubleshooting notes. The current overview is in `README.md`, the project summary in `PROJECT.md`.

---

```
QlsOS — standalone bootloader + kernel + shell
==================================================

STRUCTURE (outdated after the split into HAL/arch — the current directory map
and the "subsystem -> x86/ARM" table are in docs/architecture.md; what is below is left
as is, for the history of the first stages, while the flat structure still existed):
  boot/boot.asm            — stage1 bootloader (A20, GDT, protected mode, kernel loading)
  kernel/src/entry.asm     — the 32-bit entry point, stack, call to kernel_main
  kernel/src/isr.asm       — stubs for the 32 CPU exceptions + 16 IRQs
  kernel/src/kernel.c      — kernel_main: initialization of all subsystems
  kernel/src/vga.c         — VGA text-mode driver + kprintf (a mini-printf)
  kernel/src/serial.c      — output to COM1 (for debug logs)
  kernel/src/idt.c         — IDT setup, interrupt dispatching
  kernel/src/pic.c         — 8259 PIC remap (IRQ0-15 -> vectors 32-47)
  kernel/src/timer.c       — PIT timer (100 Hz)
  kernel/src/keyboard.c    — PS/2 keyboard driver: shift, caps lock, ring buffer
  kernel/src/shell.c       — interactive shell + a full line editor (cursor, history)
  kernel/src/mmap.c        — parser for the BIOS E820 memory map
  kernel/src/pmm.c         — physical memory manager (a bitmap frame allocator)
  kernel/src/paging.c      — paging: identity map of the first 16MB, page fault handler
  kernel/src/kheap.c       — kmalloc/kfree (a free-list allocator)
  kernel/src/task.c        — tasks and a cooperative round-robin scheduler
  kernel/src/context_switch.asm — context switching between tasks
  kernel/src/gdt.c         — GDT: ring0+ring3 segments, TSS
  kernel/src/gdt_flush.asm — loading the GDT/TSS
  kernel/src/usermode.asm  — the switch to ring3 via iret
  kernel/src/syscall.c     — the int 0x80 dispatcher (write/exit/sleep/yield)
  kernel/src/string.c      — our own memset/memcpy/strlen
  kernel/link.ld           — linker script (the kernel is loaded at 0x10000)
  Makefile                 — build with nasm+gcc+ld, run via qemu
  run.bat / build_and_run.sh — build+run with one button from Windows (see below)

WHAT ACTUALLY WORKS (verified):
  - bootloader: A20, GDT, protected mode, the E820 memory map, reading the kernel from disk
  - IDT with 32 CPU exceptions + 16 IRQs, PIC remapped, no vector conflicts
  - the PIT timer ticks at 100Hz
  - an unhandled exception (e.g. division by zero) is caught and causes a panic
    with a message, rather than quietly breaking the system
  - keyboard: PS/2 IRQ1, scancode->ascii, Shift and Caps Lock really change
    the case (verified with automated input through QEMU QMP — Shift+H+i gave
    "Hi", Caps Lock+abc gave "ABC")
  - physical memory manager: the real map from the BIOS (E820), a bitmap allocator
    of 4KB physical frames
  - paging is enabled, the first 16MB are mapped 1:1, a page fault is caught with the address
    (cr2) and the cause (present/rw/user)
  - kmalloc/kfree — a real free-list allocator that reuses
    freed blocks (not a stub), with a self-test at boot
  - command line: help, clear, echo <text>, uptime, reboot — with backspace
    and history of the current line
  - line editor (verified with automated input through QEMU QMP):
    Left/Right move the cursor along the line, a character is inserted exactly
    at the cursor position (not only at the end), Home/End jump to the start/end,
    Delete deletes the character forward from the cursor, Up/Down scroll through the command history
    and correctly restore the draft of the current line when leaving the history
  - cooperative multitasking: a background task spins a spinner in the top
    right corner of the screen while the shell waits for input (task_yield() inside
    keyboard_wait_char) — verified with screenshots taken at irregular intervals,
    the character changes on every frame, and the shell stays responsive meanwhile
    (help keeps working while the background task runs).
    For now this is COOPERATIVE switching (tasks call task_yield themselves),
    not preemptive by timer — that is the next step, together with proper
    process management (separate address spaces per process)
  - ring3 (user mode) + system calls: the GDT is extended with ring3 code/data
    segments and a TSS (needed so that the CPU knows which stack to use
    when returning from ring3 to ring0). The demo program really executes
    with CPL=3 (it cannot touch hardware directly — only via int 0x80),
    and calls sys_write_serial/sys_sleep/sys_exit.
    IMPORTANT: the demo writes to the serial log, not to the screen — if a background task
    prints to the same screen where the shell draws the input line, the shell's cursor
    can "drift" when the screen scrolls, and everything visually slides
    (this really showed up in screenshots earlier — which is why it was fixed).
    A full console with independent output areas for different
    tasks is a separate future task.
    Verified by screenshot + OCR: the screen stays clean (no cut-off
    lines), while the serial log shows 5 lines "[ring3] hello from user
    mode" — that is, the ring3 task really runs, it just doesn't get in the way of
    the interactive shell

SHELL COMMANDS:
  help    - list of commands
  clear   - clear the screen
  echo    - print the arguments
  uptime  - ticks since boot (100 ticks ~= 1 second)
  reboot  - reboot the machine (via the 8042 keyboard controller)

HOW TO RUN (Windows + WSL):
  1. Unpack qlsos.zip — run.bat lies INSIDE the qlsos folder, next
     to build_and_run.sh and the Makefile (there is no need to move it anywhere)
  2. Double-click run.bat (x86) or run "run.bat arm" from cmd (the ARM port)
     - If nasm/qemu (or for arm: arm-none-eabi-gcc/qemu-system-arm) are not
       installed in WSL — the bat installs them itself (it asks for the sudo password in the console)
     - x86: builds disk.img and opens a QEMU window
     - arm: builds kernel-arm.elf and runs qemu-system-arm -M virt
       right in the same console window (UART, no separate graphics) —
       see docs/architecture.md for how the ARM port differs from x86

IF THE QEMU WINDOW DOES NOT APPEAR:
  This means you do not have WSLg (graphics from WSL on the Windows desktop).
  To check/enable it:
    wsl --update
    wsl --shutdown
  and try again. WSLg ships by default in Windows 11 and in recent
  builds of Windows 10. If it still doesn't work at all — you can run manually
  from a WSL terminal: cd qlsos && make run-headless
  (this won't show graphics, but will show the boot serial log in the console).

IF THE WINDOW SHOWS "[WARN:COPY MODE]" AND A BLANK SCREEN:
  This is a glitch of the WSLg/FreeRDP client window itself, not of the QlsOS system —
  if the kernel had really crashed, the serial log (see run-headless) would
  show panics or resets, and there are none. run.bat/build_and_run.sh now
  try the display options in turn themselves: gtk -> sdl -> curses
  (text mode right in the console, as the last option that
  works almost always). If that doesn't help either — try closing the
  window and running run.bat again, or wsl --shutdown and start over.

REBUILD AFTER CHANGES:
  Inside WSL: cd qlsos && make clean && make && make run

============================================================
STAGE 6, items 89-90: HAL + the ARM port
============================================================

Starting from this stage the code is split into:
  kernel/hal/hal.h       — the interface that arch-independent code must
                           use instead of direct hardware access
  kernel/src/            — common code (shell, task, kheap, kprintf, string) —
                           built once, works on both architectures
  kernel/arch/x86/       — GDT/IDT/PIC/VGA/PS2/PIT/E820/PMM/paging/ring3 (as before)
  kernel/arch/arm/       — PL011 UART, GICv2, ARM generic timer, its own boot/vectors

For the details of the split and what is NOT ported to ARM (paging/PMM/ring3/reboot) —
see docs/hal-design.md.

x86 BUILD (as before):
  make            -> disk.img
  make run        -> QEMU with graphics
  make run-headless -> QEMU without graphics, the serial log goes to the console

ARM BUILD (QEMU "virt", Cortex-A15):
  Needed: arm-none-eabi-gcc, arm-none-eabi-ld, qemu-system-arm
    sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi qemu-system-arm
  make arm        -> kernel/kernel-arm.elf
  make run-arm    -> run in qemu-system-arm -M virt -nographic
                     (the UART is right in the console, the interactive shell is there too)

The ARM port has really been tested in QEMU (not just built): it reaches
the interactive shell, line editing, command history, echo/uptime work,
and so does cooperative multitasking (the spinner in the corner) — that is, GICv2 + the ARM
generic timer + the IRQ dispatcher really tick, rather than merely compile.
What is NOT on ARM yet: paging/PMM (a static kheap arena is used, as a
temporary solution), ring3/usermode/syscalls, a real reboot/poweroff
(an honest stub with a message instead of an untested PSCI call).

============================================================
STAGE 6, item 91: architecture documentation
============================================================
docs/architecture.md — the general map of the system (boot, memory, subsystems,
directory structure, extension points). docs/hal-design.md — separately about
the HAL/x86/ARM split itself. Together they cover what previously would have had to be
read out of the code.

============================================================
STAGE 6, item 93: Safe reboot/shutdown (ACPI/PSCI)
============================================================
Previously reboot was via an 8042 pulse, and poweroff — via a QEMU-specific
hack with port 0x604. Now it is done for real:

  x86 (kernel/arch/x86/src/acpi.c):
    - parses RSDP -> RSDT -> FADT -> DSDT, extracts \_S5 (SLP_TYPa/b)
    - poweroff: writes to PM1_CNT (a real ACPI S5)
    - reboot: ACPI Reset Register -> 8042 pulse -> triple fault (3 levels of fallback)
    - this required extending the identity-map paging from 16MB to 256MB — the ACPI tables
      of SeaBIOS/QEMU in practice lie closer to the top of RAM, not in the first megabyte

  ARM (kernel/arch/arm/src/hal_arm.c):
    - PSCI SYSTEM_RESET/SYSTEM_OFF via HVC, with a fallback to SMC

Shell commands: reboot, shutdown. Both paths have been really verified in a live
QEMU (not just built) — on x86 via sendkey in the QEMU monitor (the x86 console
has no direct access to input otherwise), on ARM simply via stdin. In both
cases QEMU either really rebooted or exited the process itself —
and that is the proof that ACPI/PSCI reached the chipset/hypervisor.

============================================================
BUGFIX: the x86 kernel moved to 1MB (there was a video memory "hole")
============================================================
The bug was this: input in the shell sometimes "drifted" not into the prompt, but onto a random
old row of the screen, and could even erase the "> " itself. The cause turned out not to be in
the shell logic (everything there was correct), but in the fact that the kernel was linked at 0x10000,
and its .bss grew so much that it ran into the classic "hole" in the PC
memory map: 0xA0000-0xBFFFF — that is VGA video memory, not ordinary RAM. Static
variables that were unlucky enough to land in this range (including
the shell's cursor variables and the internal state of kheap) physically did not store
what was written to them.

Fixed: boot.asm now, after the switch to protected mode, copies the kernel from
the temporary buffer to 0x100000 (1MB, the standard place for any real
OS) and jumps there; kernel/arch/x86/link.ld links the kernel at this address.

Details — in docs/architecture.md, the section "Why the kernel must live
above 1MB".

============================================================
If QEMU "stops working" after a number of launches
============================================================
This is almost always not a bug of the system, but a hung qemu process from the PREVIOUS
launch (especially if the window was closed with the X button, rather than via Machine ->
Quit) — it keeps disk.img locked, and the next launch either
fails with a lock error, or simply shows nothing.

build_and_run.sh now kills hung qemu processes itself before starting
(pkill by disk.img/kernel-arm.elf). If that doesn't help either — from Windows
(not from a WSL console):
    wsl --shutdown
and run run.bat again — this restarts the whole WSL machine itself.

```
