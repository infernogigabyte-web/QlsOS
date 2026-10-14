# ELF loader

Runs static 32-bit x86 ELF executables as ring3 processes.

```
exec <file> [args...]    run a program (active filesystem first, then ramfs); Esc kills it
elfinfo <file>           show header/segments and whether the file is loadable
```
Built-in programs (embedded in the kernel, copied to ramfs at boot): `hello` (self-test of
.text/.rodata/.data/.bss/stack/argv), `crash <null|kernel|unmapped|stack|div|ud|cli|spin>`
(each must kill only that process).

## Pieces

| File | Role |
|---|---|
| `kernel/arch/x86/src/elf.c`, `include/elf.h` | validation + mapping of PT_LOAD segments (`elf_check/validate/load`) |
| `kernel/arch/x86/src/ustack.c` | initial user stack: `argc, argv[], NULL, strings` |
| `kernel/arch/x86/src/process.c` | `process_spawn_elf()`: directory + segments + 4-page stack (+ guard page) + task |
| `kernel/arch/x86/src/exec.c` | file lookup (active VFS, then ramfs), built-in program table |
| `kernel/src/task.c` | `task_alive()`, `task_kill()` (used by the foreground `exec`) |
| `user/` | `crt0.S`, `link.ld`, `usys.h` (syscall wrappers), `hello.c`, `crash.c` |
| `tests/` | host-side tests of the real loader code (`make test`) |

## What the loader guarantees
* Only ET_EXEC / EM_386 / ELF32 / little endian. PIE, shared objects, PT_INTERP/PT_DYNAMIC are rejected with a specific error.
* The whole file is validated BEFORE any memory is allocated (64-bit arithmetic for every offset/size).
* Every segment must lie in `[0x40000000, 0x7FFFC000)`; the entry point must be inside an executable segment.
* Max 16 program headers, max 2048 pages (8 MB) per image.
* .bss is zero (frames are zeroed), segments sharing a page are supported (page becomes writable if any is).
* Failure leaves nothing behind (frames and page tables are freed).

## Process ABI
* Image at `0x40000000` (see `user/link.ld`), stack top `0x80000000`, 16 KB, unmapped guard page below.
* Entry: `esp -> argc, argv[0..], NULL, strings` (crt0 turns it into `main(argc, argv)`; return = exit).
* Syscalls (`int 0x80`, eax): 1 write(ebx=str), 2 exit, 3 sleep(ecx=ticks), 4 yield, 5 write_serial(ebx=str).
* Any fault in ring3 (page fault, #GP, #UD, #DE) kills only that process.

## Adding a program
1. `user/foo.c` (include `usys.h`, define `int main(int, char**)`).
2. Add `foo` to `USER_PROGS` in the Makefile.
3. Add a `.incbin` block to `kernel/arch/x86/src/userbins.S` and a row to `builtin[]` in `exec.c`.

## Tests
`make test` builds the user ELFs and runs `tests/elf_host_test` under ASan/UBSan: real binaries,
crafted layouts, ~30 malformed-image cases, out-of-memory/map-failure leak checks at every
failure point, `ustack_build`, and 200k mutation-fuzz iterations. It does NOT boot the kernel.

## Limits / next steps
* **The bootloader reads the kernel with one `int 0x13` into 0x10000; the kernel image must stay
  <= 64 KB** (the BIOS cannot DMA across 0x20000). The Makefile now fails the build above that.
  Fixing it properly means reading in two chunks in `boot/boot.asm`.
* Built-in programs only live in the kernel image; there is no way yet to copy a binary into QuaFS
  (a `cp` command or a host tool would fix that).
* No exit code, heap (brk), wait/spawn syscalls, argv-based environment, or Ctrl+C (Esc is used).
