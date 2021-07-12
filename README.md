# x86_hardware_debug_linux
Example functioning code of x86 hardware debug breakpoints and explanations


## Goal
1. x86 is the main architecture explained in this write-up. Other CPUs TBD?
2. Linux is the main OS. Windows TBD?
3. Simplicity. No fancy code. Just real examples for copy paste


## Prologue
This README was updated on 11Jul2021, while Intel updated their manuals on 06Jul2021. For most up to date behaviors and notes, [download the manuals here](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)

We will want the manual for "debugging" which is: **Intel® 64 and IA-32 Architectures Software Developer's Manual Combined Volumes 3A, 3B, 3C, and 3D: System Programming Guide** and scrolling down to **CHAPTER 17 DEBUG, BRANCH PROFILE, TSC, AND INTEL® RESOURCE DIRECTORTECHNOLOGY (INTEL® RDT) FEATURES** Vol. 3B 17-1 at the time of this write-up.


[Originally in my research, this Stack Overflow post was the closest thing to what I needed](https://stackoverflow.com/questions/40818920/how-to-set-the-value-of-dr7-register-in-order-to-create-a-hardware-breakpoint-on), but it lacked some details on further implementation. I will take parts from the code here:
```#define DR_OFFSET(x) (((struct user *)0)->u_debugreg + x)
typedef struct {
    uint dr0_local:     1;
    uint dr0_global:    1;
    uint dr1_local:     1;
    uint dr1_global:    1;
    uint dr2_local:     1;
    uint dr2_global:    1;
    uint dr3_local:     1;
    uint dr3_global:    1;
    uint LE:            1;
    uint GE:            1;
    uint reserved_10    1;
    uint RTM            1;
    uint reserved_12    1;
    uint GD             1;
    uint reserved 14_15 2;
    uint dr0_break:     2;
    uint dr0_len:       2;
    uint dr1_break:     2;
    uint dr1_len:       2;
    uint dr2_break:     2;
    uint dr2_len:       2;
    uint dr3_break:     2;
    uint dr3_len:       2;
} dr7_t;
```


## Summary
Set the x86 register DR7, and then control it with the EFLAGS register. Registers DR0, DR1, DR2, and DR3 are your breakpoint location, and DR6 is your status.On Linux, the breakpoint will get hit and send a signal to the kernel. As the user, you will receive a [SIGTRAP](https://man7.org/linux/man-pages/man7/signal.7.html) on the process. Catch this SIGTRAP, do whatever you need, and then resume execution by setting EFLAGS. For the most part, setting the registers and controlling execution is done with [ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html).


## Implementation

1. `ptrace(PTRACE_ATTACH, target_pid, NULL, NULL);`
This will attach your proc on to the target_pid. Only one process can attach to a proc at a time. This can fail (and assume any ptrace could fail).

2. `waitpid(target_pid, &status, 0);`
Attaching is not immediate. You will need to wait until it is finished.

3. `WIFSTOPPED(status) == TRUE && WSTOPSIG(status) == SIGTRAP`
If you're debugging something that actively uses signals, ensure you're grabbing the correct signal. This is optional.

At this point, you will need to get your breakpoint address, create your struct, and set them in. The following will set the REQUIRED values when doing one breakpoint.

4. `dr7_t.reserved_10 = 1`
This is set to 1 as depicted in Vol. 3B 17-7 Figure 17-2.

5. `dr7_t.dr0_local = 1`
We are doing a local breakpoint. Meaning, only the attached process will break, and not any other proc child/parent 17-4 Vol. 3B. If you want a global, set dr0_global=1.

6. `dr7_t.LE = 1; dr7_t.GE = 1;`
Set the breakpoint to hit on exact match of the address. For compatability, these are set to 1 regardless. See manual 17-4 Vol. 3B if you have a specific case.

7. `ptrace(PTRACE_POKEUSER, target_pid, DR_OFFSET(7), dr7_t);`
Now that we have our struct completed, we can throw it in.

8. `ptrace(PTRACE_POKEUSER, target_pid, DR_OFFSET(0), (void*)address_value);`
Set your desired address.

9. `ptrace(PTRACE_POKEUSER, target_pid, DR_OFFSET(6), 0);`
Clear the status register. Just in case.

11. `ptrace(PTRACE_CONT, target_pid, NULL, NULL);`
Resume.

So how do we step and continue execution? If you did a software breakpoint, you'd have to walk backwards and do repeated syscalls to continue. Not here.
10. `ptrace(PTRACE_GETREGS, target_pid, NULL, &regs);`
regs will come from a struct `struct user_regs_struct regs;` as defined by [user.h](https://sites.uclouvain.be/SystInfo/usr/include/sys/user.h.html)

12. `regs.eflags = regs.eflags | 0x10000;`
This will set the flag for continue. Single step is 0x1000. After a single successful execution, the flag is cleared 17-8 Vol. 3B.

10. `ptrace(PTRACE_SETREGS, target_pid, NULL, regs);`
Set the value.

11. `ptrace(PTRACE_CONT, target_pid, NULL, NULL);`
Resume.

The CPU will break at the address BEFORE it executes the address' instructions.

If there's an error occurred when setting the debug registers, GETREGS and check the GD flag if it's enabled. This is a detection method for a program changing debug registers 17-10 Vol. 3B.


## Extra

Software breakpoints are more memory manipulation related. There's tons of resources on how to do this already, but this is a brief summary if sw bps are for you.

1. attach
2. PTRACE_POKEDATA at the instructions you want to place 0xCC at
3. continue execution.
4. break
5. get registers
6. set regs.rip = regs.rip - 1
7. replace the 0xCC with the previous instruction value.
8. single step
9. place back 0xCC if the breakpoint is still desired
10. continue.