---
title: "diskread: reading beyond end of ramdisk (& How I Recovered)"
date: 2008-05-16 12:00:00 -0500
categories: [Solaris]
tags: [solaris, recovery, svm, mirror, ramdisk, sun-blade]
---

This post documents a system recovery after hardware maintenance on a Sun Blade 8000 went wrong. During a NEM module replacement, power cycling issues corrupted boot archives on multiple blades, causing panic errors during startup.

## The Problem

After technicians replaced a NEM module and the chassis experienced unexpected power cycles, two blades failed to boot with errors:

```text
diskread: reading beyond end of ramdisk
```

and

```text
panic: cannot mount boot archive
```

## Recovery Steps

I booted into Solaris 10 Failsafe mode and discovered filesystem corruption. The solution involved several steps.

### Step 1: Run fsck

I ran `fsck` multiple times on `/dev/dsk/c2t0d0s0` to repair inconsistencies:

```text
bash-3.00# fsck /dev/dsk/c2t0d0s0
** /dev/rdsk/c2t0d0s0
** Last Mounted on /
** Phase 1 - Check Blocks and Sizes
** Phase 2 - Check Pathnames
** Phase 3a - Check Connectivity
** Phase 3b - Verify Shadows/ACLs
** Phase 4 - Check Reference Counts
UNREF FILE  I=1457  OWNER=root MODE=100644
SIZE=657 MTIME=May 15 18:01 2008
RECONNECT? y

UNREF FILE  I=1458  OWNER=root MODE=100644
SIZE=675 MTIME=May 15 18:06 2008
RECONNECT? y

** Phase 5 - Check Cylinder Groups

CORRECT BAD CG SUMMARIES? y

CORRECTED SUMMARY FOR CG 0
FRAG BITMAP WRONG
FIX? y

FRAG BITMAP WRONG (CORRECTED)
CORRECTED SUMMARY FOR CG 4
CORRECTED SUMMARY FOR CG 12
CORRECTED SUMMARY FOR CG 30
CORRECTED SUMMARY FOR CG 70
CORRECT GLOBAL SUMMARY
SALVAGE? y

Log was discarded, updating cyl groups
46737 files, 1720899 used, 24099860 free (21460 frags, 3009800 blocks, 0.1% fragmentation)
```

### Step 2: Recreate Ramdisks

```bash
/boot/solaris/bin/create_ramdisk -R /a
```

```text
Creating ram disk for /a
updating /a/platform/i86pc/boot_archive...this may take a minute
```

### Step 3: Rebuild the Root Mirror

I wrote a script to automate the mirror rebuild process, which cyclically detaches each submirror, clears metadata, creates new filesystems, reinitializes metadevices, and reattaches them while monitoring resync status.

```bash
#!/bin/sh
#
# fix-mirror.sh
#
# 05-16-2008 Tim Kennedy
#
# This script will take one argument, which should be the
# metadevice of the mirror you want to rebuild.  This script
# will determine the Submirrors, and one at a time, detach,
# clear, newfs, re-init, and reattach them.
# For me this has solved problems with ailing filesystems,
# while replacement storage is procured.
#
# YMMV.  Use at your own risk.  This is not in any way to
# be considered a Sun Microsystems product, and is not in
# any way supported by Sun Microsystems.
#

PATH=/usr/bin:/usr/sbin
export PATH

MIRROR=$1

check_return () {
        RETURN=$1
        if [ $RETURN = 0 ]; then
                printf "%-6s\n" "[ok]"
        else
                printf "%-6s\n" "[err]"
                echo
                echo "please check the last step manually to see why it failed."
                echo
                exit 1
        fi
}

for m in `metastat $MIRROR | grep "Submirror of $MIRROR" | cut -d: -f1`; do
        echo "Found Submirror $m"
        DEVICE=`metastat -p $m | awk '{print $NF}'`
        printf "%-72s" "    -- metadetach $MIRROR $m"
        metadetach $MIRROR $m >/dev/null 2>&1
        check_return $?
        printf "%-72s" "    -- metaclear $m"
        metaclear $m >/dev/null 2>&1
        check_return $?
        printf "%-72s" "    -- newfs /dev/rdsk/$DEVICE"
        echo y | newfs /dev/rdsk/$DEVICE >/dev/null 2>&1
        check_return $?
        printf "%-72s" "    -- metainit $m 1 1 /dev/dsk/$DEVICE"
        metainit $m 1 1 /dev/dsk/$DEVICE >/dev/null 2>&1
        check_return $?
        printf "%-72s" "    -- metattach $MIRROR $m"
        metattach $MIRROR $m >/dev/null 2>&1
        check_return $?
        printf "%-72s" "    -- checking resync status before continuing "
        while [ 1 ]; do
                STATE=`metastat -c $MIRROR | head -1 | grep resync`
                if [ "x${STATE}" = "x" ]; then
                        printf "%-6s\n" "[ok]"
                        break;
                else
                        sleep 60
                fi
        done
done
```

After these steps, both blades successfully booted to runlevel 3, though minor inode warnings remained.
