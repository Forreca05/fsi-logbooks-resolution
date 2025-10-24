# Buffer-Overflow Attack Lab - Set-UID Version

## Question 1 — Tasks 1–3

### Task 1 — Invoking the Shellcode

In this task, we compiled ``call_shellcode.c`` with the provided Makefile and executed both the 32-bit and 64-bit binaries. Both binaries launched an interactive shell, and each shell started in the directory from which the program was executed.

![Task 1 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%235/Images/Task1.png?ref_type=heads)

### Task 2 — Understanding the Vulnerable Program

We initially studied the code of the program with a buffer overflow vulnerability. We noticed an attempt to copy a character array that can be up to 517 bytes into a buffer with a maximum size of 100 bytes. Since strcpy doesn't check buffer bounds, an overflow eventually occurs.

We then changed the variable L1 present on the Makefile to 132 (100 + 4 * 8(G)) and then compiled the program with ``make stack-L1`` which disabled StackGuard and protections against code execution invoked from the stack, changed the program's owner to root and enabled Set-UID.

![Task 2 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%235/Images/Task2.png?ref_type=heads)

### Task 3 - Lauching Attack on 32-bit Program

To exploit the vulnerability, we first create an empty ``badfile`` and run the vulnerable code in debug mode to determine the position of the ``bof()`` function's return address relative to the beginning of the buffer. To do this, we use ``gdb`` to debug and place a breakpoint in the ``bof()`` function.

```bash
$ touch badfile # Create the empty badfile file
$ gdb stack-L1-dbg # Run the program in debug mode
...
gdb-peda$ b bof # Create a breakpoint in the bof function
...
gdb-peda$ run # Run the program until it stops at the breakpoint
...
gdb-peda$ next # Advance a few instructions until the ebp register no longer points to the stack frame of the bof() function, whereas previously it still pointed to the stack frame of the function that called bof()
...
gdb-peda$ p $ebp # Get the value of ebp
$1 = (void *) 0xffffcb48
gdb-peda$ p &buffer # Get the address of the beginning of the buffer
$2 = (char (*)[132]) 0xffffcabc
```

Once we've obtained the necessary addresses, we need to insert the content we want into the buffer (shellcode and the return address pointing to that shellcode) into the badfile file. To do this, we use the provided Python program (exploit.py), with the following modifications.

In the shellcode variable, we insert the 32-bit shellcode that executes a shell.

![Task 3.1 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%235/Images/Task3.1.png?ref_type=heads)

A byte array of size 517 (maximum size of the character array read from the file by the vulnerable program) was created, where all bytes are NOP (0x90).

![Task 3.2 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%235/Images/Task3.2.png?ref_type=heads)

Placed shellcode at the end of the byte array.

![Task 3.3 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%235/Images/Task3.3.png?ref_type=heads)

Calculated the new return address that points to the shell code to be executed.

![Task 3.4 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%235/Images/Task3.4.png?ref_type=heads)

Using the 2 addresses obtained in the debug, the location of the return address relative to the beginning of the array (offset) was calculated and the new return address was placed that points to the shellcode calculated previously (ret).

![Task 3.5 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%235/Images/Task3.5.png?ref_type=heads)

The program was executed and generated the badfile file.

Finally, the vulnerable program that caused the buffer overflow was executed and launched a shell with root permissions, as expected.

![Task 3.6 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%235/Images/Task3.6.png?ref_type=heads)

## Question 2 - Visualization of the memory region affected by the overflow

After generating ``badfile`` with ``exploit.py``, I verified the file contents with ``hexdump -C``. The file contains a NOP sled (``0x90``) from the start. At ``offset`` = 0x90 (144 decimal) the 4‑byte sequence ``a6 cc ff ff`` appears — in little‑endian this corresponds to the integer ``0xffffcca6``, which is the value written to overwrite the saved return address. The shellcode (27 bytes) begins at ``0x1ea`` (490 decimal) and contains the bytes:

```bash
31 c0 50 68 2f 2f 73 68 68 2f 62 69 6e 89 e3 50 53 89 e1 31 d2 31 c0 b0 0b cd 80
```

In GDB (binary compiled with -g) I set a breakpoint at bof() and, after the function prologue, obtained the following:

```bash
p $ebp     
$1 = (void *) 0xffffcb48

p &buffer 
$2 = (char (*)[100]) 0xffffcabc
```

From these values the offsets are computed as follows:

- Distance between saved EBP and buffer start:
  ```arduino
  $ebp - &buffer = 0xffffcb48 - 0xffffcabc = 0x8c  (140 decimal)
  ```
- Saved return address is 4 bytes above saved EBP, so:
  ```ini
  offset = 0x8c + 0x4 = 0x90  (144 decimal)
  ```
  This matches the location in badfile where a6 cc ff ff was written.
- The shellcode start address in memory is:
  ```ini
  ret = &buffer + start = 0xffffcabc + 490 = 0xffffcca6
  ```

**Conclusion**: the shellcode is located at ``&buffer + start`` (file offset ``0x1ea`` → memory address ``0xffffcca6``), and the saved return address (located at ``$ebp + 4``, corresponding to file offset ``0x90``) was overwritten with ``0xffffcca6``. When ``bof()`` returns, execution is redirected to the start of the shellcode.
