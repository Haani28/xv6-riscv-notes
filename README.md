# xv6-riscv-notes
# Learning Operating Systems (xv6 + RISC-V)

This is a live dump of my raw notes, challenges, and architectural epiphanies while grinding the MIT 6.S081 labs and OSTEP during my post-SSC break. 

---

## 🧠 Core Notes & Architecture Epiphanies

### 1. The uservec() vs userret() Register Flip
I was having some trouble understanding the difference between registers like stvec, sscratch, pc, and sepc. But now I understand it.

During a trap, switching to supervisor mode requires `uservec()` code which saves the user registers ($a1, $a2, $ra, etc.) into the trapframe using the $a0 register pointer. Why doesn't `userret()` work like that? Why does it store the trapframe pointer in $a1 and the user pagetable in $a0?

*   **The Realization:** The trapframe itself is a raw block of memory, so it can't be overwritten. When returning from `usertrap()` to `userret()`, the C code itself passes the pagetable pointer as the first argument, so it gets stored in $a0. The trapframe pointer gets stored in $a1 as the second argument. This is just the standard C calling convention!
*   **What about the original user $a0?**
    $a1 holds the pointer to the original user registers. The system call's return value (success or failure) is written into the `a0` slot inside the trapframe block in RAM. This is perfect, because the user application was expecting its original $a0 (the system call argument/file descriptor) to be overwritten by the system call's return value anyway!

### 2. What even is initcode.s?

It’s the first minimal process launched by the kernel. Because the kernel cannot run user processes directly (they need to be called by `exec()` to launch), the kernel manually sets up a minimal user-space memory image in RAM containing the compiled bytes of `initcode.s`. It then forces the hardware $pc to point to those raw instructions. `initcode.s` then immediately executes an `exec()` system call to launch the first real process (`init`), letting the kernel boot everything else normally.

### 3. syscall.c vs syscall.h
*   **syscall.c**: Contains the actual logic. It decodes the system call number from the user's `a7` register, retrieves arguments from the trapframe, indexes into a function pointer array to execute the handler, and stores the final return value back into the trapframe's `a0` slot.
*   **syscall.h**: Just a header file holding the symbolic definitions. It maps system call names to their integer IDs (e.g., `#define SYS_fork 1`), matching the array indices used in `syscall.c`.

### 4. Hardware sepc vs Trapframe epc
At the start of a trap, the RISC-V hardware copies the current $pc to the hardware register `$sepc`. While the kernel executes trap code, it copies the value from the hardware `$sepc` register into the `epc` slot of the RAM-based `trapframe` struct. 

*   **The Difference:** `$sepc` is a highly volatile hardware register. If a nested interrupt or trap occurs while in the kernel, the hardware will overwrite `$sepc`, destroying our original return address. By saving it to the `epc` field in the RAM-based trapframe structure, we safely preserve the process's state across context switches.

### 5. Kernel Traps via kernelvec
When a trap occurs inside the kernel itself, the system completely skips using the user `trapframe`. Instead, `kernelvec` saves the registers directly onto the *current thread's kernel stack*. It moves the stack pointer (`$sp`) down to allocate space on the stack, dumps the registers there, processes the interrupt, and pops them back off before resuming kernel execution.

### 6. Bit Extraction Mechanics: PXMASK and PXSHIFT
Why do we use `PXMASK` after `PXSHIFT` in the `PX(level)` macro? 

Virtual addresses are broken into three 9-bit levels. `PXSHIFT` only shifts the bits down to the level we want. For example, if we want level 1 bits, `PXSHIFT` will shift past level 0 and the offset. But the higher level 2 bits still remain sitting right beside our level 1 bits. To isolate the level 1 bits completely, we use a bitwise `&` with `PXMASK` to turn off and wipe away those higher level 2 bits, leaving a clean 9-bit index.

### 7. The Purpose of sfence.vma
I just realized that `sfence.vma` flushes the TLB (Translation Lookaside Buffer) and not the page tables themselves. It acts as an instruction barrier. When the kernel modifies page tables in RAM, the CPU might still be using stale, cached address translations inside the TLB. Running `sfence.vma` forces the CPU to flush this cache and ensures all previous page table writes are fully completed before any new memory translation happens.

### 8. What's printk?
It’s the kernel-space equivalent of the standard user-space `printf` function. While a standard `printf` uses system calls to output text to a terminal screen via the shell, `printk` (or `printf` inside the xv6 kernel) talks directly to the hardware UART (serial port) to print debugging and error messages straight to the console.

### 9. Redundant if(killed(p)) Checks in trap.c
Why is `if (killed(p))` mentioned multiple times inside `trap.c`? 

Operating systems are highly asynchronous. While the kernel is busy handling a trap, processing a page fault, or waiting for an I/O device, another CPU core or a timer interrupt might have marked this process as killed. To ensure the kernel doesn't waste time executing code for a dead process—or worse, return back to user space as if nothing happened—it explicitly polls the `killed` flag at major transition points.
