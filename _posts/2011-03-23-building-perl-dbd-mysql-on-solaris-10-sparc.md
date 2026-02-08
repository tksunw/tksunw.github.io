---
title: "Building Perl DBD::mysql on Solaris 10 Sparc"
date: 2011-03-23 12:00:00 -0500
categories: [Solaris]
tags: [solaris, perl, mysql, dbd, sparc]
---

Having problems building the Perl DBD::mysql modules on Solaris 10 Sparc 64-bit? The Perl 5.8.4 binary that ships with Solaris 10 is a 32-bit application.

The core issue is a platform architecture mismatch. When attempting to compile DBD::mysql against a 64-bit MySQL installation while using Solaris 10's native 32-bit Perl, the build fails due to incompatible binaries.

## Solution

Download the 32-bit version of MySQL and link against it instead. In this example, 64-bit MySQL runs in `/opt/mysql/mysql` and the 32-bit version is unpacked to `/opt/mysql/mysql32`.

## Build Commands

```bash
/usr/perl5/5.8.4/bin/perlgcc Makefile.PL --libs '-R/usr/sfw/lib \
-R/opt/mysql/mysql32/lib -L/usr/sfw/lib -L/opt/mysql/mysql32/lib \
-lmysqlclient -lz -lposix4 -lcrypt -lgen -lsocket -lnsl -lm' \
--cflags '-I/usr/sfw/include -I/usr/include -I/opt/mysql/mysql32/include'
```

Then execute:

```bash
gmake install UNINST=1
```
