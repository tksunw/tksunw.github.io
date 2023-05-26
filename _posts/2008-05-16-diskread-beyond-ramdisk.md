---
layout: post
title: "diskread: reading beyond end of ramdisk (& how I recovered)
categories: misc, solaris, sun
---

#### `diskread: reading beyond end of ramdisk`

We had to do a maintenance to replace a NEM module in a Sun Blade 8000 Modular System.

Two of my team mates went on down to the datacenter on other business and graciously offered to SWAP the NEM for me. The pulled the old one out, stuck the new one in.

That's as simple as it should have been.

Should have been. I wish. Instead, the chassis started to freak out, cycling it's power over and over, and somehow was taking the CMM with it. In between one set of cycles, I was able to connect to the CMM via console and paste in a bunch of commands to shut down chassis power. I let it sit for a moment, then began to power up the system. First the chassis, then the individual blades. One blade came up, no problem. The next two, though, were very much less than happy, spitting out errors like:
```
diskread: reading beyond end of ramdisk
	start = 0x2000, size = 0x2000
failed to read superblock
diskread: reading beyond end of ramdisk
	start = 0x2000, size = 0x2000
failed to read superblock
panic: cannot mount boot archive
Press any key to reboot
```

The GRUB menu was coming up OK, though, so I pressed the trusty any key, booted into Solaris 10 Failsafe mode. This was no picnic either.

```
SunOS Release 5.10 Version Generic_120012-14 32-bit
Copyright 1983-2007 Sun Microsystems, Inc.  All rights reserved.
Use is subject to license terms.
Booting to milestone "milestone/single-user:default".
Configuring devices.
Searching for installed OS instances...
NOTICE: /a: unexpected free inode 5825, run fsck(1M)
/dev/dsk/c2t0d0s0 is under md control, skipping.
To manually recover the boot archive on a root mirror,mount the first
side (the one that the system boots from) and run:

        bootadm update-archive -R 

umount: /a busy

No installed OS instance found.

Starting shell.
#
```

My immediate thought was "WTF? No installed OS instance found?" Closer inspection revealed that it had in fact found two possibilities, one on c2t1d0s0 was inconsistent and needed a fsck, and the second on c2t1d0s0 was under md control, and so being skipped.

An fsck of /dev/dsk/c2t0d0s0 revealed a few inconsistencies. Here's an example. I think this was actually the 3rd of 4 fscks I ran on this dev:
```
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
46737 files, 1720899 used, 24099860 free (21460 frags, 3009800 blocks, 0.1% 
fragmentation) 

***** FILE SYSTEM WAS MODIFIED *****
```
So far, so good. Let's reboot, and see if we an come up in a multi-user state. So ... reboot ... wait ... wait ...

CRAP! Same panic as our previous boot. We're missing something. 

A further delve into google reveals that I need to recreate the ramdisks for boot. A boot into failsafe mode again, allows me to `fsck` c2t0d0s0, which is mounted on /a, and remount it -o rw. `bootadm update-archive` fails, due to fs inconsistency. Another fcsk, we're in single user, nothing is using that disk, so I just ran the fsck without remounting -o ro. Now, let's skip bootadm and just move straight along to `/boot/solaris/bin/create_ramdisk`.

```
bash-3.00# /boot/solaris/bin/create_ramdisk -R /a
Creating ram disk for /a
updating /a/platform/i86pc/boot_archive...this may take a minute
```

That's it! That's the little piece of magic that fixed it. After that, I was able to reboot, and the server came right up into runlevel 3. Not without a few minor errors, but at least it was up.

```
SunOS Release 5.10 Version Generic_127112-11 64-bit
Copyright 1983-2008 Sun Microsystems, Inc.  All rights reserved.
Use is subject to license terms.
Hostname: generic
NOTICE: /: unexpected free inode 9193, run fsck(1M) -o f
NOTICE: /: unexpected free inode 5961, run fsck(1M) -o f
WARNING: /: unexpected allocated inode 9637, run fsck(1M) -o f
Loading smf(5) service descriptions: 1/1
/dev/md/rdsk/d60 is clean
/dev/md/rdsk/d30 is clean
/dev/md/rdsk/d20 is clean

generic console login:
```

At this point it was pretty simple to complete the fix, which, not wanting to reboot into failsafe mode and fsck a bunch more to recover from the unexpected free and allocated inodes, I wrote a script to: by turns, detach each half of the root mirror, clear the detached metadevice, newfs the raw device, re-create the metadevice, and attach it once again to the mirror. Let it sit long enough to complete the resync, and repeat the same steps on the other half of the mirror.

```shell
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
        metadetach $MIRROR $m &gt;/dev/null 2&gt;&amp;1
        check_return $?
        printf "%-72s" "    -- metaclear $m"
        metaclear $m &gt;/dev/null 2&gt;&amp;1
        check_return $?
        printf "%-72s" "    -- newfs /dev/rdsk/$DEVICE"
        echo y | newfs /dev/rdsk/$DEVICE &gt;/dev/null 2&gt;&amp;1
        check_return $?
        printf "%-72s" "    -- metainit $m 1 1 /dev/dsk/$DEVICE"
        metainit $m 1 1 /dev/dsk/$DEVICE &gt;/dev/null 2&gt;&amp;1
        check_return $?
        printf "%-72s" "    -- metattach $MIRROR $m"
        metattach $MIRROR $m &gt;/dev/null 2&gt;&amp;1
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

Now these blades are happy once again. We'll see how long that lasts or if they continue to have problems of any sort. My hope is for the former.

Have a good weekend.

*this post was migrated from blogs.sun.com/tksunw
