---
title: "Logging Shell Commands to Syslog on Secure Systems"
date: 2010-12-07 12:00:00 -0500
categories: [Linux]
tags: [bash, syslog, auditing, security, accountability]
---

While typical corporate systems may not require extensive auditing, government systems, whether classified or not, do require this level of accountability.

I was tasked with adding CLI logging to network hosts for audit and accountability support. Syslog-based command logging provides an intuitive, portable solution, though it has weaknesses -- notably that passwords entered on command lines are logged in plaintext.

## Implementation with Bash

GNU Bash 4.1 includes built-in logging capability. The configuration requires editing `config-top.h`. The original file has the `SYSLOG_HISTORY` define commented out:

```c
/* Define if you want each line saved to the history list in bashhist.c:
   bash_add_history() to be sent to syslog(). */
/* #define SYSLOG_HISTORY */
#if defined (SYSLOG_HISTORY)
#  define SYSLOG_FACILITY LOG_USER
#  define SYSLOG_LEVEL LOG_INFO
#endif
```

To enable it, simply uncomment the define:

```c
/* Define if you want each line saved to the history list in bashhist.c:
   bash_add_history() to be sent to syslog(). */
#define SYSLOG_HISTORY
#if defined (SYSLOG_HISTORY)
#  define SYSLOG_FACILITY LOG_USER
#  define SYSLOG_LEVEL LOG_INFO
#endif
```

## Log Format Modifications

The original log format looks like this:

```text
Dec  7 23:13:02 linux bash: HISTORY: PID=1752 UID=1001 ls
```

I modified the logging format for improved readability. The original `bashhist.c` syslog call:

```c
if (strlen(line) < SYSLOG_MAXLEN)
    syslog (SYSLOG_FACILITY|SYSLOG_LEVEL, "HISTORY: PID=%d UID=%d %s",
        getpid(), current_user.uid, line);
```

I replaced it with a modified `openlog()` and `syslog()` call:

```c
openlog("bash",LOG_PID,SYSLOG_FACILITY);
if (strlen(line) < SYSLOG_MAXLEN)
    syslog (SYSLOG_LEVEL, "[%s] %s", current_user.user_name, line);
```

Resulting in a much cleaner log format:

```text
Dec  7 23:26:39 linux bash[1846]: [tkennedy] ls
```

## Additional Context

The organization also uses the BOFH-patched tcsh shell for consistency across systems. I standardized on bash and tcsh for interactive shells on Linux, with no exceptions granted to users.

## Limitations

This approach is not a replacement for comprehensive system auditing but serves as a practical tool in environments with non-malicious users requiring activity tracking.
