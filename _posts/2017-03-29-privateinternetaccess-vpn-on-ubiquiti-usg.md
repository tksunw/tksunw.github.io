---
title: "PrivateInternetAccess VPN on a Ubiquiti USG"
date: 2017-03-29 12:00:00 -0500
categories: [Networking]
tags: [vpn, pia, ubiquiti, unifi, usg, privacy]
---

So Congress scrapped an FCC rule called the Broadband Consumer Privacy Proposal, which previously required broadband providers to obtain permission before collecting and selling subscriber data. In response to that, I upgraded my home router to a Unifi Security Gateway from Ubiquiti Networks and wanted to configure it with Private Internet Access (PIA) VPN.

Existing posts in the UBNT Community Forums contained confusion, or are just outdated regarding USG VPN configuration with PIA.

The setup for a PIA VPN configuration is very easy.

## Route Configuration

The primary challenge involved calculating routes for all subnets outside the home network. Using RFC1918 private address space, you need to add these routes via the USG settings app "subnets" menu:

- `0.0.0.0/1`
- `192.169.0.0/16`
- `192.170.0.0/15`
- `192.172.0.0/14`
- `192.176.0.0/12`
- `193.0.0.0/8`
- `194.0.0.0/7`
- `196.0.0.0/6`
- `200.0.0.0/5`
- `208.0.0.0/4`
- `224.0.0.0/3`

## Routing Logic

Because hosts have a default route to the USG (192.168.1.1), all traffic reaches it. The USG maintains a default ISP route and a local route to 192.168.0.0/22 (internal network). More specific routes always win in routing tables, so the subnet list above provides more specific routes than the default for all non-local IPs, forcing external traffic through the VPN while remaining easily overrideable if desired.

## USG Configuration Settings

The configuration uses these specific settings:

- **Purpose:** VPN Client
- **VPN Client:** PPTP
- **Enabled:** checkbox (enable when VPN should be active)
- **Remote Subnets:** individual entries for each subnet listed above
- **Server IP:** obtained from PIA (example: `nslookup us-east.privateinternetaccess.com`)
- **Username:** PIA credentials
- **Password:** PIA credentials
- **MPPE:** Yes (for encryption)

## Notes

Netflix definitely blocks PIA. For devices requiring direct internet access, you can move the VPN tunnel off the USG onto a computer behind it, then configure specific devices to use that computer as their default gateway.

Enjoy your ISP not selling your internet activities to advertisers.
