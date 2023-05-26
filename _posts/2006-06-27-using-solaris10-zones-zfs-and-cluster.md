---
layout: post
title: "Using Solaris 10 Update 3, Sun Cluster 3.2, Zones, & ZFS in a Multi-Node Cluster of Sun Fire T-2000s"
categories: misc, containers, zfs, solaris
---

####It all started with a conference call with one of our customers. 

They wanted a way to set up some highly available systems that could be used for various beta 
or QA purposes, or production services, or anywhere in between as needed. We also wanted a 
way to maximize the resources. We had 4 servers available to us, all Sun Fire T-2000s. If we 
used them as straight servers, they'd be great at anything they do, right? 

8 cores, 4 threads per core, 32GB of RAM. Nice. Capable of running dozens of zones without 
skipping a beat. Perhaps even hundreds of zones.

Zones make perfect development boxes, right? You can blow them away and re-install them in a 
matter of minutes, or even seconds on ZFS. Zones also do pretty good as production environments 
as well. We're currently using a large number of zones in production, to supply a variety of 
services.

Zones on ZFS make particularly good dev boxes because you can take frequent snapshots and roll 
back as desired.

** Zones with their zoneroot on ZFS do encounter bug #6356600, which relates to how the live 
upgrade scratch zone used for installing packages into local zones can't access ZFS filesystems 
to upgrade zones with a ZFS zoneroot.

Sun Cluster 3.2 introduced support for ZFS as a failover filesystem, and for failover zones as 
well. We decided to make use of both of these features.

We built a 4-node cluster out of the 4 T-2000s, and began exporting individual disks from our 
3Par SAN. Put 3 disks in a ZFS pool as a raidz filesystem, and installed the zone root at a 
ratio of 1 zone per zpool. (We're still doing some testing with our SAN and comparing performance 
of ZFS on individual disks, or ZFS on a RAID5 LUN exported by the SAN, but so far the way we're 
doing it is working nicely.)

So we built the cluster, and got it all configured and running. We then installed the first zone 
onto the ZFS pool. Then I copied the relevant portions of the zones configuration (from /etc/zones/)
 to the other nodes in the cluster.

We then created a resource group in Sun Cluster 3.2, and added the zone into the resource group. 
We also added the ZFS pool into the resource group as an HA-Storage resource, and created a 
quick set of control scripts to start and stop the zone. The zone itself takes care of bringing 
up it's ip addresses, and starting the various applications installed within.

End result: Highly available servers, on a failover basis, that take less than 30 seconds to fail 
over from one host to another.

So far it's working really well. We're already getting more requests to build more of these 
multi-node clusters, with zone/zpool combo's as the resource group. It's been a great solution 
for us, and for our customers.

*this post was migrated from blogs.sun.com/tksunw
