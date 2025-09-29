# Project 1: Buffer overflows

This project will give you first-hand experience with buffer overflow attacks.
This attack exploits a buffer overflow vulnerability in a program to make the
program bypass its usual execution and instead jump to alternative code (which
typically starts a shell). There are several defenses against this attack
(other than fixing the overflow vulnerability itself), such as address space
randomization, compiling with stack guard, and making the stack non-executable.
We will be turning off some of these defenses, and you will be circumventing
other ones.

**Your project must work in the VM/QEMU image that we provided.** As such,
you should be constantly testing your project within that VM.

## Task 0: Off to a malicious start!

To start things off, we provide a program that you get to write part of.

The program we provide:
- Takes as a command-line argument the name of a file, which it opens.
- The program then asks your function (`your_fcn()`) for a string, writes
  that string to the file, and closes the file.
- It then re-opens the file, and attempts to "sanitize" the string you provided
  by removing all instances of the substring "`414`" from it.
- Finally, it overwrites the file with this "sanitized" version of the string.

For example, if your function generated the string "I love CMSC414!" then
we would get the following output:

```
cmsc414@cmsc414:~$ ./task0a.x file.txt
cmsc414@cmsc414:~$ cat file.txt
I love CMSC!
```

**Your goal is to write `your_fcn()` in such a way that the file whose name is
provided on the command-line still has the substring "`414`" in it.**

A simple way to test for this is with the command `grep 414` which returns lines
containing the string "`414`" as follows:

```
cmsc414@cmsc414:~$ grep 414 file.txt
```

If this returns nothing (as the above case would), then that means the file
does not have "`414`" in it, resulting in no credit.

We provide three copies of this program: `task0a.c`, `task0b.c`, and
`task0c.c`. All of them have the same goal, but **you must provide a
*functionally unique* solution in each of the three**; they cannot simply be
slight variations of one another (e.g., if one of your solutions was
`printf(x)` then another cannot be `puts(x)` as those are functionally the same
thing).

There are many different ways to solve this! Some might involve buffer
overflows, and some might not. We are not requiring any one particular
solution; just like with any attack, what matters is whether it achieves
the goal, even if it's not a method we anticipated. *Be creative* and have fun
with it!

Please note that you are *only* allowed to modify `your_fcn()` in these files.

## Task 1: Attacking a vulnerable program with malicious inputs

In Task 1, you will be exploiting programs that have a buffer overflow
vulnerability. Unlike Task 0, you are not allowed to modify the program itself;
instead, you will be attacking it by cleverly constructing malicious inputs to
the program.

The vulnerable programs are vulnerable1.c through vulnerable3.c; your solutions
cannot involve modifying those programs (when we grade your project, we will use
unmodified versions of the vulnerable programs). Rather, you will be modifying
the corresponding exploit programs, exploit1.c through exploit3.c

Each of the vulnerable programs is a little different. We designed them so that
they increase in difficulty; you apply what you learn in solving vulnerable1 to
inform how to do vulnerable2, which will in turn inform you how to do
vulnerable3, where you will be injecting code that will reveal a secret!

### `vulnerable1.c`

The goal with this program is to get `sensitive_function()` to get called.
Note that the only time that function is called is from within
`buffer_overflow()`; `exploit1.c` must provide a string that will result in
this happening. This will give you some basic experience with overflowing
buffers.

### `vulnerable2.c`

The goal here is also to get `sensitive_function()` to be called, but you'll
see that that function is actually *never* called in this program.
`exploit2.c` will have to provide a string that somehow induces
`sensitive_function()` to be called by overwriting the saved `%eip` to be the
address of `sensitive_function()`. However, you will have to contend with ASLR;
`sensitive_function()`'s address is not guaranteed to be the same every time
you run the program, nor on every machine. Fortunately, you'll note that
`vulnerable2.c` has *another* vulnerability: a format string vulnerability.
Unlike in `exploit1.c`, `exploit2.c` gets to send *two* strings to
`vulnerable2.c`; it gets a response from `vulnerable2.c` before constructing
and sending the final buffer for the buffer overflow attack. You'll have to
think about how to leverage the format string vulnerability to construct a
buffer that works *every time*, regardless of ASLR.

### `vulnerable3.c`

This is similar to `vulnerable2.c`, but instead of jumping to a function that
already exists in the program, you need to inject your own code. We have
provided code to inject; rather than open a shell, it executes `/bin/cat
/var/secret/token`: this outputs the contents of the `/var/secret/token`; this
is a file created by the Makefile that only root has access to read. Because
vulnerable3.x is compiled as root, when injected with this command, it will be
able to read the token---even if the user running vulnerable3.x is not root.
This shows the power of privilege escalation!

The way the `exploit` and `vulnerable` programs interact with one another is
through some networking code that we provide for you in `comms.h` and
`comms.c`; you do not have to concern yourself with the networking code. Just
focus on constructing the right `greeting` (when appropriate), analyzing the
vulnerable program's `response` (when appropriate), and constructing the
correct `buffer` to solve the given challenge.

### Building and running the programs

We provided a Makefile for compiling the files in Task 1. Your solution cannot
involve modifying the Makefile; when we grade your projects, we will be using
the unmodified version of the Makefile.

To run, first build using `make`. Then, run the `vulnerable` program first,
followed by the corresponding `exploit` program. For example:

```
cmsc414@cmsc414:~$ ./vulnerable1.x &
cmsc414@cmsc414:~$ ./exploit1.x
```

The first command's use of `&` means to run the vulnerable program in the
background; this means that it will immediately return us to the terminal but
it will keep running. However, **we suggest that you run `vulnerable` in one
terminal and `exploit` in another**; this is especially helpful if you want to
run the `vulnerable` programs in  `gdb`, and trust us, you do!

## Some helpful tips

Here are a few tips that we think will help you throughout this project.

### Not all character arrays are "C strings"

Recall: `strlen()` only works for null-terminated ASCII strings; if you are
sending binary data then you either want to compute the length of the data
you're sending or at least use `sizeof(buffer)`.
     
### gdb is your friend!

Here are some useful commands for running and stepping through a running program:
- `b <function name>`: Set a breakpoint at `<function name>` (pause execution whenever
  that function is reached).
- `r`: Run the program from the beginning.
- `s`: (Used after you've hit a breakpoint.) "Step": Run the current line of code and
  step to the next line of code. If the current line of code is a function call, it
  will step into the function call (i.e., the next line of code is the first line of
  code in the function).
- `n`: (Used after you've hit a breakpoint.) "Next": Like the `s` command, but it will
  not step into the function; it will execute it in its entirety and pick up on the
  line after the function call.
- `c`: (Used after you've hit a breakpoint.) "Continue": let the program execute
  until it hits another breakpoint.

Here are some useful commands for inspecting the memory of the running process:
- `i f` (short for `info frame`): Shows info about the current stack frame,
  including the addresses where the saved %eip and %ebp are.
- `x/32xw <address>`: Print 32 words (`w`) in hex format (`x`) starting at
  address `<address>`. The `<address>` can either be a raw address (e.g.,
  `0x12345678`) or a function name (e.g., `main`).
- `p <C expression>`: Print the result of the expression, which could be a
  variable (`p var`), the address of a variable (`p &var`), or a more 
  complicated C expression (`p (int) strlen(str)`).
