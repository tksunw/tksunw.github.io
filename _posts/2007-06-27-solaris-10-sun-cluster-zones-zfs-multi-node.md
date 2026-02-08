---
title: "Using Solaris 10 Update 3, Sun Cluster 3.2, Zones, & ZFS in a Multi-Node Cluster of Sun Fire T-2000s"
date: 2007-06-27 12:00:00 -0500
categories: [Solaris]
tags: [solaris, sun-cluster, zfs, zones, t2000, virtualization]
---

I worked with a customer to create highly available systems that could be used for various beta or QA purposes, or production services.

## Hardware Setup

Four Sun Fire T-2000 servers were available, each equipped with 8 cores, 4 threads per core, and 32GB of RAM. We recognized these machines could potentially support dozens of zones without skipping a beat. Perhaps even hundreds of zones.

## Architecture Decision

We built a 4-node cluster and configured it with ZFS-backed Solaris zones. We put 3 disks in a ZFS pool as a raidz filesystem, and installed the zone root at a ratio of 1 zone per zpool.

## Zone Configuration

Zone configurations were copied from `/etc/zones/*` to other cluster nodes. We created resource groups in Sun Cluster 3.2 and added zones with corresponding ZFS pools as HA-Storage resources, developing control scripts to manage zone startup and shutdown.

## Results

The implementation achieved failover in under 30 seconds. Highly available servers, on a failover basis, that take less than 30 seconds to fail over from one host to another.

> **Note:** Bug #6356600 affects zones with ZFS zoneroots during live upgrades.

The solution generated positive reception, with increasing requests to deploy additional multi-node clusters using this zone/zpool architecture.
