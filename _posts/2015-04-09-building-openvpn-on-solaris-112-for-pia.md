---
title: "Building OpenVPN on Solaris 11.2 for Use with PIA VPN"
date: 2015-04-09 12:00:00 -0500
categories: [Solaris]
tags: [solaris, openvpn, vpn, pia, privacy]
---

I wanted to establish OpenVPN connectivity on Solaris 11.2 to connect with Private Internet Access (PIA) VPN. This references prior work by Stefan Reuter on OpenSolaris 2008.11. While the basic approach remains similar, some implementation details have evolved for Solaris 11.2.

## Step 1: Install the TAP Driver

```bash
git clone https://github.com/kaizawa/tuntap.git
./configure
gmake
sudo gmake install
```

## Step 2: Install LZO Compression Library

```bash
wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.09.tar.gz
tar -zxvf lzo-2.09.tar.gz
cd lzo-2.09
./configure
gmake
gmake check
sudo gmake install
```

## Step 3: Install OpenVPN

```bash
wget https://swupdate.openvpn.org/community/releases/openvpn-2.3.6.tar.gz
tar -zxvf openvpn-2.3.6.tar.gz
cd openvpn-2.3.6
CFLAGS="-I/usr/local/include" LDFLAGS="-L/usr/local/lib" ./configure --with-gnu-ld --enable-password-save
gmake
sudo gmake install
```

## Configuration

I modified PIA's standard OpenVPN configuration file with custom parameters:

```text
client
dev tun
proto udp
remote us-east.privateinternetaccess.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
tls-client
remote-cert-tls server
auth-user-pass
comp-lzo
verb 1
reneg-sec 0
crl-verify crl.pem
auth-user-pass .pia.login
script-security 2
route-delay 2
route-up route-up.sh
route-noexec
```

The credentials file `.pia.login` contains your PIA username and password, one per line:

```text
yourusername
yourpassword
```

## Route-Up Script

```bash
#!/usr/bin/env ksh

# OpenVPN passes the remote gateway in as $route_vpn_gateway.
/usr/sbin/route add 0.0.0.0/1 $route_vpn_gateway
/usr/sbin/route add 128.0.0.0/1 $route_vpn_gateway
```

The script manages routing by creating two specific routes, allowing traffic to route over the VPN while maintaining access to the local LAN via more specific routes.

## Notes

This has been successfully implemented on Solaris 11.3 using OpenVPN 2.3.x as well. Versions 2.4.0 and later exhibited segmentation faults attributed to OpenSSL compatibility issues on Solaris.
