dcos16 - A simple cooperative multitasking kernel for DCPU16
============================================================

The dcos16 kernel for Notch's DCPU16 processor manages up to 16
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

The code has been tested in dcpustudio.

The license is MIT as that is the first well known open source license I
found that did in fact exist in 1988 (GPL v1 is 1989).

The kernel provides the following calls. All calls that return
preserve registers X, Y, Z, I, J, yield does addtionaly keep A, B, C
and SP. No assumptions should be made about O.

Process Management
==================

yield
-----

```yield``` takes no parameters and allows the next process to run

fork
----

```fork``` takes the start address of the routine to start as a process in
register ```A``` and a mode parameter in register ```B```.

Mode 0 starts a daemon process that is completely detached from the
calling process.

Mode 1 starts a child process that runs until termination and then
lingers as a zombie process until the caller queries its exit status
with wait.

Mode 2 starts the child process as a synchronous subprocess and the
calling process is suspended until the child exits.

If no new process can be started, fork returns -1 in both ```A``` and
```B```.  Otherwise it retuns 0 in ```B``` and in ```A``` either 0 (in
mode 0), the PID (a number from 0 to 0xF) of the child process (in
mode 1) or the value the child passed to the exit call in register
```A``` (in mode 2).

fork clobbers registers ```A``` to ```C``` in the caller, in the child
```A``` to ```C``` are 0 and ```X, Y, Z, I, J``` are the same as in
the caller.

exit
----

```exit``` terminates a process and passes the value in register ```A``` to the
parent if it was forked in mode 1 or 2. Note that a mode 1 process
still occupies one of the 16 process slots after it exited until the
parent queries the result with wait. Since exit never returns, it
doen's make any difference it it is called as ```JSR exit``` or ```SET
PC, exit```.

wait
----

```wait``` queries the result of a terminated child process. It takes
the PID of the child process in register ```A``` and a mode in
register ```B```. The mode can be either 0 for a blocking wait or 1
for a non-blocking wait.

It returns -1 in both ```A``` and ```B``` if the number passed in
```A``` is not a valid PID of a child of the calling process. If the
child has terminated, it returns 0 in ```B``` and the return value
(see above) in ```A```.  If the child is not terminated and the call
is non-blocking, it returns 0 in ```A``` and -1 in ```B```. If the
call is blocking, the processes will behave as if the fork had been
done in mode 2 from then on.

Status display
==============

Dcos16 reserves the top two lines of the screen as a status display
for the running processes (in the end we want to have a process for
each of the ships main functions and a way to see if everything is ok
at a glance). By default, it displays the content of the A register
when a process calls yield in black on white.

The colors are three bit rgb values:

<table>
<tr><td>000</td><td>0x0</td><td>White</td></tr>
<tr><td>001</td><td>0x1</td><td>Blue</td></tr>
<tr><td>010</td><td>0x2</td><td>Green</td></tr>
<tr><td>011</td><td>0x3</td><td>Cyan</td></tr>
<tr><td>100</td><td>0x4</td><td>Red</td></tr>
<tr><td>101</td><td>0x5</td><td>Magenta</td></tr>
<tr><td>110</td><td>0x6</td><td>Yellowish</td></tr>
<tr><td>111</td><td>0x7</td><td>White</td></tr>
</table>

(yellowish can be brown or orange or olive or whatever)

status_color
------------

```status_color``` sets the foregrond and background colors of the
status cell of the currently running process. It takes a color between
0 and 7 in eac of the lowest two nibbles of register ```A``` and
clobbers registers ```A``` to ```C```. So

```
SET A, 0x24
JSR status_color
```

will set the foreground to green (2) and the background to red (4)
