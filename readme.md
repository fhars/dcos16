dcos16 - A simple cooperative multitasking kernel for DCPU16
============================================================

The dcos16 kernel for <a href="http://twitter.com/#!/notch">Notch</a>'s
<a href="http://0x10c.com/doc/dcpu-16.txt">DCPU16 processor</a> manages up to 16
cooperative processes with a very simple round robin scheduler.

New processes are started with the ```JSR fork``` call described
below.  Every process is reponsible for not writing over the other
processes memory and to call ```JSR yield``` at appropriate times to
allow other processes to run. All registers
except O are saved over a yield call, and each process has its own 256
word stack, where again each process is responsible for not
overflowing it. The process local stack is initialized with a pointer
to the exit routine at the top, so a process can exit with ```SET PC,
POP``` like any normal subroutine, or it can explicitly call
```exit``` if it wants to terminate when the stack is not empty.

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

set_interactive
---------------

By calling this routine a process can say whether it wants to be
suspended while it does not have the focus.

Status display
==============

Dcos16 reserves the top two lines of the screen as a status display
for the running processes (in the end we want to have a process for
each of the ships main functions and a way to see if everything is ok
at a glance). One process is considered as the currently active
process, its status cell is highlighted. Users can cycle between
processes using the Tab key.

By default, the status cell displays the content of the A register
when a process calls yield in black on white.

A process can change its status display color with the
```status_color``` routine The colors are three bit rgb values:

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
0 and 7 in each of the lowest two nibbles of register ```A``` and
clobbers registers ```A``` to ```C```. So

```dasm16
SET A, 0x24
JSR status_color
```

will set the foreground to green (2) and the background to red (4)

status_blink
------------

```status_blink``` sets the blink attribute of the current process
status cell if A is 1 and clears it otherwise. The blink bit is also
cleared if the user tabs to or from the process. ```status_blink``` is
meant as a way for processes that are not currently displayed to
notify the user that something has happened that requires immediate
attention.


status_i2hex
------------

```status_i2hex``` displays the content of the A register in the
process status cell. After a process has called this routine, the
automatic status update on yield will be deactivated.

status_string
-------------

```status_string``` displays the four characters in the lower seven
bits of the four words pointed to by the A register in the process
status cell. After a process has called this routine, the automatic
status update on yield will be deactivated.

status_raw
-----------

```status_raw``` displays the four characters in the lower seven bits
of the foure words pointed to by the A register in the process status
cell, including color, hilight and blink bits. The highlight and blink
bits will be reset the next time the process recieves focus. After a
process has called this routine, the automatic status update on yield
will be deactivated.

Utility functions
=================

random
------

```random``` uses a 32 bit LCG to generate random numbers. It returns
the next number in registers A (high) and B (low). It is initialized
with a pregenerated random number, and the state is updated on every
keypress (as that is currently the only source of nondeterminism in
the system).

Input and output
================

One of the processes is always considered as being the one that is
displayed on the screen below the status lines. That process is also
said to "have focus". The user can cycle between the processes using
tab (and, in some emulators, Pg_Up and Pg_Dn).

By default, everytime the user switches to the next process, the
screen is cleared and output starts on the top of the
screen. Alternatively, a process may register a 320 word buffer to use
a a backing store that is uses for the video content while the process
is not running.

Key presses will be deliverd to the process that has the focus. dcos16
stores one key per process, additional keys pressed before the
last key has been read are discarded.

The current set of output functions has been adapted from <a
href="https://github.com/jdiez17/0x42c">0x42c</a> and currently
clobber more registers than they probabaly should. They set the
foreground color to bright white and leave any background color and
blink attribute that might be in the higher bits of the words
containing the characters in their lower seven bits.

The output functions have no effect if the process does not have
focus.

register_screen
---------------

Call this method if the process should have a persistent video output
even if it loses focus. The process must provide a 320 word buffer to
store the video content.

getch
-----

```getch``` performs a nonblocking read of the keyboard, returning
either the last key pressed for the current process, or 0.

readch
------

```readch``` performs a blocking read of the keyboard, returning
either the last key pressed, suspending the process until a key has
been presse if none is available.

readline
--------

```readline``` reads a line of up to 32 characters (including the
terminating zero) into the buffer pointed to by A, blocking until
characters are available. B contains the length of the buffer. For
```readline``` to be useful, the process must have reserved screen
memory, as the function directly modifies the screen buffer.

```readline``` supports basic line editing using the left and right
arrow keys and backspace.

editline
--------

```editline``` is basically the same function as readline, except that
it allows to put some text into the edit buffer and puts the insertion
cursor at the end of that text. In addition to the parameters of
```readline```, it expects the length of the existing text in C.

println
-------

prints the zero-terminated string pointed to by A followed by a
newline.

print
-----
prints the zero-terminated string pointed to by A.

printchar
---------
prints the character in A.

newline
-------
Starts a new line

clear_screen
------------
Clears the screen.
