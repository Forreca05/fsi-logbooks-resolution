# CVE: CVE-2014-6271
**Common name:** Shellshock  
**Summary:** Flaw in Bash's processing of function definitions in environment variables, allowing remote code execution.

---

### Identification
The CVE-2014-6271 (Shellshock) is a critical vulnerability in Bash, where functions defined in environment variables can include malicious commands. It affects versions up to 4.3 on Unix-like operating systems such as Linux and macOS, and is exploitable when services expose environment variables and invoke Bash, such as in CGI scripts or DHCP clients.

---

### Cataloging
The initial flaw was publicly disclosed on September 24, 2014, with emergency patches released by several vendors. Due to the high risk of remote code execution, the severity was rated critical. In addition to the initial fix, other related CVEs (e.g., CVE-2014-7169) were created to address gaps in the first mitigation.

---

### Exploit (type / automation)
Exploitation occurs through manipulated environment variables that inject commands into Bash, allowing remote execution. Common attack vectors included CGI on web servers, OpenSSH with ForceCommand, DHCP clients, and large-scale exploitation enabling remote shells. Modules for automation appeared quickly in tools like Metasploit.

---

### Attacks (reports / impact)
Following disclosure, the internet was massively scanned to find and compromise vulnerable servers. The impact was widespread, affecting web servers, routers, embedded devices, and cloud instances. Service interruptions and significant operational compromises were reported globally.

---

### Mitigation / Countermeasures
Recommended mitigation consisted of immediately applying official Bash patches provided by vendors. Additionally, it was advised to disable or isolate vulnerable CGI scripts, strengthen input sanitization, and restrict services that passed environment variables to Bash. Post-patching verification should follow vendor guidelines to ensure complete elimination of the flaw.
