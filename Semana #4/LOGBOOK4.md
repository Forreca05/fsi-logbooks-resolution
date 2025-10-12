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



