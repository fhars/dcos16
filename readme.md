dcos16 - A simple cooperative multitasking kernel for DCPU16
============================================================

The dcos16 kernel for notchs DCPU16 processor manages up to 16
cooperative processes with a very simple round robin scheduler.  Every
process is reponsible for not writing over the other processes memory
and to call

```
    JSR yield
```

at appropriate time to allow other processes to run. All registers
except O are saved over a yield call, and each process has its own 256
word stack, where again each process is responsible for not
overflowing it.

The kernel provides the following calls. All calls that return
preserve registers X, Y, Z, I, J, yield does addtionaly keep A, B, C
and SP. No assumptions should be made about O.

yield
-----

yield takes no parameters and allows the next process to run

fork
----

fork takes the start address of the routine to start as a process in
register A and a mode parameter in register B.

Mode 0 starts a daemon process that is completely detached from the
calling process.

Mode 1 starts a child process that runs until termination and then
lingers as a zombie process until the caller queries its exit status
with wait.

Mode 2 starts the child process as a synchronous subprocess and the
calling process is suspended until the child exits.

If no new process can be started, fork returns -1 in both A and B.
Otherwise it retuns 0 in B and in A either 0 (in mode 0), the PID (a
number from 0 to 0xF) of the child process (in mode 1) or the value
the child passed to the exit call in register A (in mode 2).

fork clobbers registers A to C in the caller, in the child A to C are 0
and X, Y, Z, I, J are the same as in the caller.

exit
----

Exit terminates a process and passes the value in register A to the
parent if it was forked in mode 1 or 2. Note that a mode 1 process
still occupies one of the 16 process slots after it exited until the
parent queries the result with wait. Since exit never returns, it
doen's make any difference it it is called as ```JSR exit``` or ```SET
PC, exit```.

wait
----

Wait queries the result of a terminated child process. It takes the
PID of the child process in register A and a mode in register B. The
mode can be either 0 for a blocking wait or 1 for a non-blocking wait.

It returns -1 in both A and B if the number passed in A is not a valid
PID of a child of the calling process. If the child has terminated, it
returns 0 in B and the return value (see above) in A.  If the child is
not terminated and the call is non-blocking, it returns 0 in A and -1
in B. If the call is blocking, the processes will behave as if the
fork had been done in mode 2 from then on.
