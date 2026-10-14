
# QuaOS architecture

This document is a general map of the system. For how exactly the split between
common code and architecture code (the HAL) works, see `docs/hal-design.md` separately —
here that question is only mentioned, not duplicated.

## The big picture

```
BIOS/QEMU -> boot/boot.asm (stage1, real mode) -> kernel/arch/<arch>/...
             (x86 only; on ARM the role of stage1               (arch-specific
              is played by arch/arm/src/boot.S itself,           entry + drivers)
              which is loaded directly via QEMU -kernel)
                                                        |
                                                        v
                                          kernel/src/kernel.c: kernel_main()
                                          (arch-independent, sees only hal_*)
                                                        |
                                          arch_platform_init()  (arch-specific)
                                          hal_timer_init / kheap_init / tasks_init
                                          arch_extra_demo()     (arch-specific)
                                                        |
                                                        v
                                              kernel/src/shell.c: shell_run()
```

## Boot (x86)

1. The BIOS puts the first disk sector (`boot/boot.asm`, 512 bytes, signed
   with `0xAA55`) at address `0x7C00` and hands control over to it in 16-bit real mode.
2. Stage1 by itself, with no third-party libraries: collects the memory map via
   BIOS `int 0x15/E820` (this is the last moment when the BIOS is available at all),
   enables the A20 line, loads the remaining disk sectors (the kernel itself) into a
   TEMPORARY buffer at `0x10000` via `int 0x13` (real mode cannot
   address above ~1MB directly), builds a temporary GDT and switches
   to 32-bit protected mode (`CR0.PE=1`).
3. **Already in protected mode**, BEFORE jumping into C code: it copies the kernel (`rep movsd`)
   from the temporary buffer `0x10000` to its real link address
   `0x100000` (1MB) — see below for why this is mandatory — and only then
   jumps to `kernel/arch/x86/src/entry.asm` (already at address `0x100000+`).
4. `entry.asm` sets the segment registers for the new GDT, zeroes
   `.bss`, sets up the stack, calls `kernel_main()` — from this moment we are in the common,
   arch-independent C code.

### Why the kernel must live above 1MB (found in practice, not in theory)

The classic PC memory map reserves `0xA0000-0xBFFFF` for VGA video memory
(graphics modes) and `0xC0000-0xFFFFF` for the BIOS/option ROMs — this is not ordinary
RAM, but a real "hole" in the address space, which QEMU reproduces as well.
Previously the kernel was linked at `0x10000`, and over time its `.bss` (a 256KB
kheap arena + everything else) grew so much that it ran exactly into
this hole. Static variables that were unlucky enough to land in this range
(`kheap`'s `free_list`, the shell's cursor, etc.) physically did not store what was
written to them — the write went "into video memory", and the read returned something other than what
was written. The symptoms were all different and at first glance unrelated to
each other: input in the shell "drifted" onto a random screen row, `kmalloc` crashed into a
page fault at address `0xFFFFFFFF` — and it was all the same cause.
It was fixed by moving the kernel to `0x100000`, the standard place for any real
OS. The moral: if a static variable somewhere "loses" values for no visible
reason — the first thing to do is check its real link address (`nm`),
rather than immediately suspecting a race condition or the initialization order.

## Boot (ARM)

ARM has no BIOS and no stage1 in the usual sense — QEMU (`-M virt -kernel
kernel-arm.elf`) itself puts the ELF into memory (`0x40010000`, see
`kernel/arch/arm/link.ld`) and hands control to the entry point in SVC mode.
`kernel/arch/arm/src/boot.S`:
1. Sets up separate stacks for the SVC and IRQ modes (a shared stack for both
   would be a bug: the interrupt handler would trample the stack of the interrupted task).
2. Programs `VBAR` (the Vector Base Address Register) with the address of its own table of
   exception vectors (`vectors.S`) — without this the processor would look for the
   vectors at address `0x0`, where we loaded nothing.
3. Zeroes `.bss` manually (the ELF loader in QEMU does not guarantee that BSS
   is already zero in memory — on real hardware even less so).
4. Calls `kernel_main()`.

## Memory map

**x86**: the kernel is linked with a base of `0x100000` (1MB — see the section
"Boot (x86)" above for why exactly there, and not lower, and
`kernel/arch/x86/link.ld`), the PMM manages the first 64MB of physical memory
(a bitmap frame allocator on top of the E820 map), paging does an identity map of the
first 256MB (it was 16MB — extended when implementing item 93, because
the ACPI tables of SeaBIOS/QEMU in practice lie not in the first megabyte, but closer
to the top of RAM: without this `acpi_init()` fell into a page fault on any real
QEMU, not only in theory). The real RAM size above 256MB is visible to the PMM, but is not
mapped — full support for memory beyond this static ceiling
(dynamic mapping based on E820) is material for a separate future
item, not part of the current list 89-99.

**ARM**: the kernel is linked at `0x40010000` (the QEMU `virt` machine keeps RAM at
`0x40000000`). Paging/MMU and the PMM are not ported to ARM at all yet (see
`docs/hal-design.md`, the section "what is not ported") — `kheap` works as
a static arena in `.bss`, with no tie to the physical memory map.

## Subsystems (what is responsible for what)

| Subsystem           | x86                              | ARM                                  | Common code |
|---------------------|-----------------------------------|---------------------------------------|-----------|
| Output console      | VGA text-mode 80x25, hardware cursor | PL011 UART + ANSI escape codes      | `kprintf.c` |
| Input               | PS/2 keyboard, IRQ1              | PL011 UART, ANSI sequences            | — |
| Interrupts          | IDT (32 exceptions + 16 IRQs) + PIC | Vector table (`vectors.S`) + GICv2    | — |
| Timer               | PIT, 100Hz, via IRQ0              | ARM generic timer (CP15), via GIC IRQ 30 | — |
| Memory              | E820 -> PMM (bitmap) -> paging (identity 256MB) -> kheap | a static kheap arena (no PMM/paging) | `kheap.c` |
| Multitasking        | `context_switch.asm` (push ebx/esi/edi/ebp) | `context_switch.S` (push r4-r11,lr) | `task.c` (preemptive round-robin on x86, timer-driven) |
| Ring3/syscalls      | GDT+TSS, `int 0x80`, per-process page directories, ELF loader, fork/waitpid/signals | not ported | — |
| Shell               | —                                  | —                                       | `shell.c` (line editor, history, commands) |
| Reboot/poweroff     | ACPI (RSDT/FADT/DSDT `\_S5`, Reset Register) + fallback (8042/triple fault) | PSCI (`SYSTEM_RESET`/`SYSTEM_OFF` via HVC/SMC) | — |

## Directory structure

```
boot/boot.asm            — x86 stage1 (real mode, the only one for x86)
kernel/hal/hal.h          — the contract between the common code and the arch backends
kernel/include/           — arch-independent headers (task.h, kheap.h, shell.h,
                             string.h, kprintf.h)
kernel/src/               — arch-independent implementation (kernel.c, shell.c,
                             task.c, kheap.c, kprintf.c, string.c)
kernel/arch/x86/          — include/ + src/ (asm and C) + link.ld — everything x86-specific
kernel/arch/arm/          — include/ + src/ (asm and C) + link.ld — everything ARM-specific
user/                     — user programs (ELF) + crt0/linker script
tests/                    — host-side tests (`make test`)
docs/                     — this documentation
Makefile                  — targets: `make`/`make run` (x86), `make arm`/`make run-arm` (ARM)
```

## Extension points

If you need to add a new architecture (for example, RISC-V from the future
items, if any appear): create `kernel/arch/<name>/`, implement
the whole set of functions from `kernel/hal/hal.h` (`hal_console_*`, `hal_input_*`,
`hal_timer_*`, `hal_irq_*`, `hal_cpu_*`, `hal_reboot`/`hal_poweroff`,
`hal_context_switch`, `hal_new_task_stack`, `arch_platform_init`,
`arch_extra_demo`, `hal_arch_name`) — and all of `kernel/src/` will work without
a single edit. That is the whole point of the split described in `hal-design.md`.
