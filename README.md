# xv6-lottery-scheduler
Lottery (proportional-share) CPU scheduler for the xv6 RISC-V teaching OS, with a settickets() system call and procdump ticket stats.

This project replaces the default round-robin scheduler in [xv6-riscv](https://github.com/mit-pdos/xv6-riscv) (MIT's teaching operating system) with a **lottery scheduler**. Each process holds a number of tickets, and on every scheduling decision the kernel draws a random winning ticket. Processes with more tickets win more often, so they get a proportionally larger share of the CPU.

## How it works

On each pass of the scheduler loop, the kernel:

1. Adds up the tickets of all `RUNNABLE` processes.
2. Picks a random number between `0` and `total_tickets - 1` using a Park–Miller pseudo-random number generator.
3. Walks the process table, keeping a running ticket count, and runs the first process whose count passes the winning number.

A process with 30 tickets competing against one with 10 should, over time, get roughly three times as much CPU.

## Changes made

| File | Change |
|------|--------|
| `kernel/proc.h` | Added `tickets` and `rounds` fields to `struct proc` |
| `kernel/proc.c` | Lottery logic in `scheduler()`, `rand()` helper, default of 1 ticket per new process, 10 tickets for `init`, children inherit their parent's tickets on `fork()`, and `procdump()` now prints tickets and rounds |
| `kernel/syscall.c` / `kernel/syscall.h` | New system call `settickets` (syscall number 23) |
| `user/user.h` / `user/usys.pl` | User-space stub for `settickets()` |
| `user/test_scheduler.c` | Test program that sets its tickets and then spins forever |
| `Makefile` | Added `_test_scheduler` to `UPROGS`; set `CPUS := 1` so the ticket split is easy to observe |

### New system call

```c
int settickets(int n);
```

Sets the calling process's ticket count to `n`. Returns `0` on success, or `-1` if `n` is not a positive integer.

## Building and running

You need the RISC-V GNU toolchain and QEMU built for `riscv64-softmmu`.

```bash
make qemu
```

## Testing the scheduler

Inside the xv6 shell, start several CPU-bound processes with different ticket counts:

```
$ test_scheduler 30 &
$ test_scheduler 20 &
$ test_scheduler 10 &
```

Wait a few seconds, then press **Ctrl-P**. The process dump shows each process's ticket count and how many times it has been scheduled:

```
3 run    test_scheduler tickets: 30 rounds: ...
4 runble test_scheduler tickets: 20 rounds: ...
5 runble test_scheduler tickets: 10 rounds: ...
```

The `rounds` values should come out roughly in a 3 : 2 : 1 ratio, matching the ticket split. Press Ctrl-P a few more times to watch the ratio settle.

The standard xv6 test suite can be run with:

```bash
./test-xv6.py usertests
./test-xv6.py -q usertests   # quick version
```

## Credits

Based on [xv6-riscv](https://github.com/mit-pdos/xv6-riscv) by Frans Kaashoek, Robert Morris, and contributors at MIT PDOS. See `LICENSE` for the original license.
