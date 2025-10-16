# Environment Variable and Set-UID Lab

## Question 1 — Tasks 1–6

### Task 1 — Manipulating Environment Variables

**Goal:** Learn how to view, set, and unset environment variables in Bash.

---

### 🔧 Commands Used

```bash
printenv             # Print all environment variables
printenv PWD         # Print only the PWD variable
env | grep PWD       # Filter for PWD using grep
export ANYNAME=hello # Set a new environment variable
unset ANYNAME        # Remove the environment variable
```

### Code example:
![Task 1 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task1.png?ref_type=heads)

### Task 2 — Parent to child inheritance (fork)

**Program:** myprintenv.c prints environ (the environment array) in a loop.

Steps:
1. Compile: gcc myprintenv.c -o myprintenv
2. Run child printing: ./myprintenv > file
3. Modify program so parent prints and child does not, recompile, run: ./myprintenv > file2_
4. Compare: diff file file 2

**Conclusion:** The child process created by fork() inherits the parent’s environment (exported variables are duplicated to the child).

### Code example:
![Task 2 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task2.png?ref_type=heads)

### Task 3 — `execve()` and Environment

**Program:** `myenv.c`

```c
#include <unistd.h>
extern char **environ;

int main() {
    char *argv[2];
    argv[0] = "/usr/bin/env";
    argv[1] = NULL;

    execve("/usr/bin/env", argv, NULL);   /* Variant 1 */
    return 0;
}
```

-  Variant 1: execve(..., NULL) → running this prints nothing because the new program received no environment.

-  Variant 2: execve(..., environ) → the new program prints the full environment inherited from the caller.

**Conclusion:** The environment passed to a program executed via execve() is determined by the envp (third) argument; it is not automatic if you pass NULL.

### Task 4 — `system()` and environment

**Program:** `mysystem.c`

```c
#include <stdio.h>
#include <stdlib.h>
int main() { 
    system("/usr/bin/env"); 
    return 0; 
}
```

**Observation:** ``system()`` internally runs ``/bin/sh -c <command>;`` the shell is executed with the caller’s environment (because the internal ``execl()/execve()`` passes the environment). Therefore, the command run by ``system()`` receives the same environment as the calling process.

### Task 5 — Set-UID programs and environment variables

Procedure summary:

1. Create a program that prints environ (same as Task 2).
2. Compile and make it Set-UID root:
   ![Task 5.1 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task5.1.png?ref_type=heads)
3. As a normal user (e.g., seed), export variables:
   ![Task 5.2 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task5.2.png?ref_type=heads)
4. Run ``./env``.

**Observations:**
-  Some variables defined by the user (e.g., ``ANYNAME``) often remain visible.
-  **Security-sensitive variables** such as ``LD_LIBRARY_PATH``, ``LD_PRELOAD``, etc., are ignored or cleared by the dynamic loader when a Set-UID binary runs. This prevents attackers from forcing a privileged program to load untrusted libraries.
- **Conclusion:** Set-UID programs do not unconditionally inherit all user environment variables; the system filters potentially dangerous env vars.

### Task 6 — The PATH Environment Variable and Set-UID Programs

**Goal:** Demonstrate that a Set-UID program that calls ``system("ls")`` can be tricked into running attacker code if ``PATH`` is controlled and the invoked shell does not drop privileges. Document all steps, expected outputs, and final cleanup (restore ``/bin/sh`` to ``/bin/dash``).

#### Step 1 — Victim program

Create ``victim.c``:
```c
#include <stdlib.h>
int main() {
    system("ls");
    return 0;
}
```
Compile and make Set-UID root:
![Task 6.1 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task6.1.png?ref_type=heads)

**Purpose:** ``victim`` calls ``system("ls")`` → ``/bin/sh -c "ls"`` is executed.

#### Step 2 — Attacker binary (ls placed in home)

Create ``mal_ls.c``:
```c
#include <stdio.h>
#include <unistd.h>
int main() {
    printf("malicious ls running: UID=%d EUID=%d\n", (int)getuid(), (int)geteuid());
    return 0;
}
```

Build and place it as ``~/ls``:
![Task 6.5 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task6.5.png?ref_type=heads)

**Purpose:** Show whether the attacker binary runs with EUID 0 (root).

#### Step 3 — Put attacker directory first in PATH

Save current PATH and set new PATH:
![Task 6.2 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task6.2.png?ref_type=heads)

**Why:** If ``/home/seed`` is checked before /``bin``, the shell will run ``~/ls`` when asked to run ``ls``.

#### Step 4 — Test with default /bin/sh (dash)

Run the victim:
![Task 6.3 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task6.3.png?ref_type=heads)

**Explanation:** ``dash`` detects it was invoked in a Set-UID process and drops privileges (sets EUID=UID) before running commands. Attack fails (no root).

#### Step 5 — Replace ``/bin/sh`` with ``/bin/zsh``

```bash
sudo ln -sf /bin/zsh /bin/sh
ls -l /bin/sh  
```

#### Step 6 — Re-run the victim
With ``/home/seed`` at the front of PATH and ``/bin/sh`` now ``zsh``, run:
![Task 6.4 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task6.4.png?ref_type=heads)

**Interpretation:** The attacker ``~/ls`` was executed with effective UID 0, i.e., ran as root. Attack succeeded because:
-  ``system()`` invoked a shell;
-  the shell used ``PATH`` to find ``ls`` and ran attacker code;
-  the shell did not drop privileges in the Set-UID context, so code ran with root EUID.

#### Technical explanation (why these steps are sufficient):

1.  system("ls") → /bin/sh -c "ls". The shell resolves ls using PATH.
2. If an attacker controls PATH and places their own ls earlier, the shell will execute the attacker binary.
3. If the shell runs the command with the Set-UID process’s effective privileges, attacker code executes as root (privilege escalation).
4. Some shells (dash) detect Set-UID and drop EUID before running commands, preventing this escalation. Replacing /bin/sh with a shell that lacks such a countermeasure (e.g., zsh in this lab) demonstrates the vulnerability.
5. Conclusion: Never use system() in privileged (Set-UID) code without sanitizing environment variables (especially PATH). Prefer absolute paths, execve() with controlled envp, or dropping privileges before executing user-controllable commands.


## Question 2 - Task 8.1 

### Task 8 — Invoking External Programs Using system() versus execve()~

#### Step 1 — Exploiting system() in Set-UID catall

**Goals:** Show that a Set-UID catall using system() can be abused to delete a protected file.

**Program:** catall.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stddef.h>
int main(int argc, char *argv[])
{
char *v[3];
char *command;
if(argc < 2) {
printf("Please type a file name.\n");
return 1;
}
v[0] = "/bin/cat"; v[1] = argv[1]; v[2] = NULL;
command = malloc(strlen(v[0]) + strlen(v[1]) + 2);
sprintf(command, "%s %s", v[0], v[1]);
system(command);

return 0 ;
}
```
![Task 8.1 Example](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%234/Images/Task8.1.png?ref_type=heads)

#### Technical Explanation
1. Catall builds a command string "/bin/cat " + argv[1] 
2. Calls system(command) and system() runs sh -c "<command>" 
3. Shell parses metacharacters like ;.

Because catall is Set-UID root, the shell and the injected /bin/rm execute with effective UID 0, allowing deletion of files the unprivileged user cannot normally remove.

**Conclusion:** This shows that using system() in a Set-UID program is dangerous because it allows command injection through user input.To prevent such vulnerabilities, privileged programs should avoid using system(), validate all user input, and use safer functions like execve() with controlled arguments.


## Question 3 - Task 9

**Goal**: Demonstrate the vulnerability known as **capability leaking** in set-UID programs. The example program opens a privileged file while it runs with elevated privileges, then revokes privileges with setuid(getuid()), and finally executes a shell. The exercise’s goal is to explain why the process can still write to the protected file even after the UID downgrade.

## 1. Target code (conceptual analysis)

```c
void main()
    {
    int fd;
    char *v[2];

    /* Assume that /etc/zzz is an important system file,
    * and it is owned by root with permission 0644.
    * Before running this program, you should create
    * the file /etc/zzz first. */
    fd = open("/etc/zzz", O_RDWR | O_APPEND);
    if (fd == -1) {
    printf("Cannot open /etc/zzz\n");
    exit(0);
    }

    // Print out the file descriptor value
    printf("fd is %d\n", fd);

    // Permanently disable the privilege by making the
    // effective uid the same as the real uid
    setuid(getuid());

    // Execute /bin/sh
    v[0] = "/bin/sh"; v[1] = 0;
    execve(v[0], v, 0);
}
```
 
The reference program essentially does:

1. open("/etc/zzz", O_RDWR | O_APPEND); — opens the important file while the process is still privileged.
2. prints the file descriptor number (fd).
3. setuid(getuid()); — attempts to permanently drop privileges (sets effective UID to the real UID).
4. execve("/bin/sh", ...) — executes a shell.

**Key point (conceptual):** The file descriptor returned by open() is a kernel resource belonging to the process. If that descriptor remains open and not closed, the process (or its exec'ed descendant) still has access to the file through that descriptor (even if the effective UID has been changed). This is not the same as checking UIDs; it's about preserved kernel resources.

## 2. Environment preparation

* Create the empty test file (/etc/zzz) as root and set its permissions so root can read/write, while others can only read.
* Compile the cap_leak.c file 

![Task 9 Example (1)](Images/Task9(1).png)

## 3. Evidence of execution

* Change the owner of cap_leak from seed to root and set the set-UID bit so the program runs with root EUID (ls -l should show -rwsr-xr-x)
* Run cap_leak as a normal user, which prints the open file descriptor, exposing the privileged FD to the user.
* Use that descriptor from the spawned shell to write into /etc/zzz and verify the change.

![Task 9 Example (2)](Images/Task9(2).png)


## 4. Technical explanation (detailed)

### 4.1 What is capability leaking?

* Modern Linux uses both user IDs (UIDs) and capabilities (fine-grained kernel privileges such as CAP_NET_ADMIN, CAP_DAC_OVERRIDE, etc.).
* A process that begins with elevated privileges (for example, a set-UID root program) may acquire privileged resources (file descriptors for protected files, kernel capabilities, keys, etc.).
* After dropping privileges (e.g., via setuid(nonroot)), *some resources obtained while privileged can remain accessible* if they are not explicitly revoked or closed. This is called capability leaking: the process no longer has an effective root UID, but still retains capabilities or resources that allow privileged actions.

### 4.2 Which resource is exploited in cap_leak.c?

* *Concrete resource:* the file descriptor returned by open("/etc/zzz", ...) while the process was privileged.
* Why this matters: file descriptors are kernel resources associated with a process (and are inherited by exec/children unless they have the close-on-exec flag). Even if the effective UID is changed afterwards, the descriptor still refers to the file with the originally granted access mode (here, O_RDWR | O_APPEND), so writes via the descriptor remain possible.

### 4.3 Why does opening before setuid() enable the bypass?

* open() is performed while the process is privileged; the kernel permits opening /etc/zzz even if the eventual unprivileged user could not open it. The kernel returns a file descriptor with the requested access.
* setuid(getuid()) changes the effective UID (and often real and saved UIDs when the caller was root), but it *does not automatically close already-open file descriptors* or remove acquired capabilities unless the program explicitly does so.
* When the program calls execve("/bin/sh", ...), the new process inherits descriptors that do not have the FD_CLOEXEC flag. Thus the invoked shell can use the inherited FD to perform writes to the file, even though the shell runs with a non-privileged UID.
* In short: *an opened FD that persists across the UID drop and across exec is the bypass*.

### 4.4 Notes on kernel capabilities

* In other scenarios, the process could also hold kernel capabilities (e.g., CAP_DAC_OVERRIDE) that allow bypassing permission checks. If these capabilities are not cleared after the UID downgrade (using libcap or prctl appropriately), the process continues to perform privileged operations despite not being UID 0.
* Therefore, vulnerability sources include both preserved FDs and residual capabilities.

## 5. Mitigation measures (countermeasures)

To prevent this class of vulnerability in set-UID programs:

1. *Do not open privileged files before dropping privileges.* If possible, perform privileged operations only when absolutely necessary and avoid holding privileged resources while relinquishing privileges.
2. *Close all privileged file descriptors* before dropping privileges. Enumerate and close known FDs (including sockets and pipes).
3. *Use FD_CLOEXEC* on descriptors that must not survive an execve(); this ensures they are automatically closed on exec.
4. *Explicitly clear capabilities* after dropping privileges, using libcap APIs (e.g., cap_set_proc()) or ensure prctl(PR_SET_KEEPCAPS, 0) is configured appropriately before changing UIDs. Make sure permitted/effective/inheritable capability sets are cleaned as needed.
5. *Avoid set-UID when possible.* Prefer minimal-privilege designs: small privileged helpers with narrow, well-validated interfaces (via IPC) instead of making a large program set-UID root.
6. *Sanitize the environment* before invoking system() — remove dangerous variables (e.g., LD_PRELOAD) and avoid system() when you can use execve() with explicit arguments.
7. *Design for least privilege*: minimize the code paths that run with elevation, limit time spent privileged, and reduce accessible resources while privileged.
8. *Audit and test*: include static analysis and dynamic testing for privileged code paths, and perform security code reviews for set-UID components.


## 6. Final evidence and conclusions (summary)

* *Technical conclusion:* Opening sensitive resources (e.g., files) while privileged and then merely calling setuid() without closing those resources or clearing capabilities can allow a process to retain practical privileged access. This is a classic instance of capability leaking.
* *Practical takeaway:* Always close or otherwise revoke privileged resources before dropping privileges; prefer minimized, audited privileged helpers and explicit capability control.
* *Security-by-design* is the most effective defense: minimize privileged code, apply the principle of least privilege, and perform careful cleanup after privilege transitions.
