---
layout: post
title: Logging Shell Commands to Syslog on Compliant Systems
date: 2010-12-07 12:07:00
category: Sysadmin
---

## Logging Shell Commands to Syslog on Compliant Systems

I had recently come across a blog post describing methods for capturing commands entered on the command line, and recording them to syslog.  Either by function() or by patching the actual shell itself.   I found this article because I was asked by my boss to find a way to add CLI logging to some hosts on our network, to support audits and accountability.
Some of the environments I work on are more secure than average.  In a typical corporate environment, whether internet connected or not, there is generally no need or requirements to use system auditing to track all user actions.  Some government systems, whether classified or not, do require this, and some commercial systems in regulated industries, or who service government agencies, also require this level of auditing and accountability.  In some cases it can be a smart idea for non-regulated systems.  For instance, if you're a managed services company that uses a team of operators to manage multiple customer environments, there may be some value to tracking user activity.

Most operating environments these days come with some sort of auditing facility, however I have found that these are usually fairly unintuitive, and most people that implement auditing do so by following a How-To, and then end up not actually having a SME on staff when things do go wrong.  Audit logs can also consume a lot of space, so lots of sysadmins just delete old audit logs in an effort to reclaim disk space.

One easy, and portable, way to quickly and intuitively audit user activity involves using patched shells to send all the commands run to syslog.  I should note that there are a few weaknesses in logging all commands to syslog, such as password exposure.  Some people do put passwords in the command line of such tools as ldapsearch, or mysql, or sqlplus.  Those passwords will then be recorded in plain-text in your system logs.

And there are always ways to work around being logged, like running a shell that doesn't log to syslog, which can be done as simply as by uploading your own, non-logging, shell and running it.

Aside from the weaknesses outlined above, though, in an environment where users are not malicious, and where team members can find themselves on any of a number of systems at any particular time, logging user can provide very valuable context in a familiar way.  And quickly, too.

Using syslogging shells is simply a tool like any other.  It doesn't replace real system auditing, but it definitely has it's place.

GNU Bash 4.1 has all the code to enable command logging, simply by editing the config-top.h file.

Just Change:

```
/* Define if you want each line saved to the history list in bashhist.c:
   bash_add_history() to be sent to syslog(). */
/* #define SYSLOG_HISTORY */ 
#if defined (SYSLOG_HISTORY)
#  define SYSLOG_FACILITY LOG_USER
#  define SYSLOG_LEVEL LOG_INFO
#endif
```

to

```
/* Define if you want each line saved to the history list in bashhist.c:
   bash_add_history() to be sent to syslog(). */
#define SYSLOG_HISTORY 
#if defined (SYSLOG_HISTORY)
#  define SYSLOG_FACILITY LOG_USER
#  define SYSLOG_LEVEL LOG_INFO
#endif
```

and BOOM, all commands run in bash, by any user are logged to syslog, and can now easily be offloaded from the system in realtime by using a remote syslog server.

One thing I did notice is that in Solaris, the PID was being logged with the log entry to syslog, however this was not the case in linux.  Rather the PID was being logged by the log entry %s itself.
```
if (strlen(line) < SYSLOG_MAXLEN)
    syslog (SYSLOG_FACILITY|SYSLOG_LEVEL, "HISTORY: PID=%d UID=%d %s", 
 getpid(), current_user.uid, line);
```

Resulting in log entries like:
```bash
Dec  7 23:13:02 linux bash: HISTORY: PID=1752 UID=1001 ls /tmp/
```

I don't really like that format, either. I'd rather see usernames and commands, and have the pid over on the left with the 'bash:'. This was a pretty simple change in code to:
```
 openlog("bash",LOG_PID,SYSLOG_FACILITY);
  if (strlen(line) < SYSLOG_MAXLEN)
    syslog (SYSLOG_LEVEL, "[%s] %s", current_user.user_name, line);
```
This results in log entries that look like:
```bash
Dec  7 23:26:39 linux bash[1846]: [tkennedy] ls /tmp/
```

To me, this makes for a more readable log file.

{% gist 7470268 %}
