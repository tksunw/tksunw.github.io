---
layout: post
title: "A generic perl script to scan a CIDR subnet for listeners on a specific port."
categories: misc, scripting, perl
---

Ever had a customer ask you where *some process* was running on *some port* in their network?

I have.  And usually this involves an environment that doesn't have any common port scanning 
tools.  Fortunately these days, almost every Unix/Linux OS comes with Perl, even Solaris. 

Since I work for a managed services company, and we manage a multitude of different environments, 
each with it's own set of restrictions and requirements, I try to write the most portable code 
that I can, so that it has the best chance of actually working in any given environment. 

This script uses a couple of standard Perl modules that are included as part of the default 
installation, and don't require any CPAN-Fu, and it takes a couple of options, such as a switch 
for verbosity, and IP address, with or without a CIDR mask, and a TCP port.  The CIDR mask 
defaults to /32, and the port defaults to 22.  Here's an example of the output.

```shell
tcsh-[101]$ ./scan-port.pl 208.64.63.39/30 80
================================================================================
Request to scan 208.64.63.39/30 on Port 80 (http)
Scanning 4 IP Addresses:
--------------------------------------------------------------------------------
208.64.63.36    : rats.entic.net                   : listening on 80 (http)
208.64.63.37    : corn.entic.net                   : listening on 80 (http)
208.64.63.38    : ice.hostsonfire.co.uk            : listening on 80 (http)
208.64.63.39    : jeep.sugarat.net                 : listening on 80 (http)
--------------------------------------------------------------------------------
Found 4 hosts listenening on port 80.
================================================================================
```

And here's the script itself. 

```perl
#!/usr/bin/env perl
#===============================================================================#
#
# scan_port.pl (c) tkennedy - 2011 November 19 Version 1.1
#
#-------------------------------------------------------------------------------#
#
# Description: Allows scanning a IP Subnet for TCP listeners on designated port.
# Syntax: $0 [-v] IPADDRESS/CIDR-NETMASK [Port]
#
#-------------------------------------------------------------------------------#
#
# Purpose:
# This script provides a means of scanning a subnet for TCP listeners, using 
# only commonly available Perl functions.  This should make the script portable
# to any system with Perl installed.  The script can be passed 3 arguments. 2 of
# the arguments are optional, and include a '-v', which increases the verbosity
# of script output, and 'P', which is a TCP port presented as an integer between
# 1 - 65535.  If a port is not supplied on the command line, the script assumes
# a default of 22, which is the common port for Secure Shell traffic (ssh). The
# mandatory argument is an IP Address, either with, or without, a CIDR netmask.
# If a CIDR netmask is not supplied on the command line, then we assume you only
# intend to scan a single IP (ie, a /32).
#
#-------------------------------------------------------------------------------#
#
# History:
#
# 20101123 1.1 tkennedy - removed Switch module for Sol8/Perl5.003 compatibility
#
# 20101119 1.0 tkennedy - initial revision
#
#===============================================================================#
#
# The modules we're using are standard perl modules, so this script should 
# work on any operating system with Perl installed.
#
use strict;
use IO::Socket;

my ($ip,$cidr,$port,$target);
my $VERBOSE = 0;

# We need a regex to match IP addresses.  This is only used to validate
# the command-line options to verify that one option is an IP.
#
my $ipr = qr/^((?:(?:2(?:5[0-5]|[0-4][0-9])\.)|(?:1[0-9][0-9]\.)|(?:(?:[1-9][0-9]?|[0-9])\.)){3}(?:(?:2(?:5[0-5]|[0-4][0-9]))|(?:1[0-9][0-9])|(?:[1-9][0-9]?|[0-9])).*$)/x;

# I wanted to keep things as free-form as possible, and so opted to just 
# parse the command line to extract our options.  This also gives us some
# leeway to ignore bad options.
#
# In our parser, we will match a '-h' for help info, a '-v' to extend 
# our output a bit, including printing lines for hosts that are scanned
# but not listening.  We'll also match a single number, which we'll 
# interpret as a TCP port number, and lastly, we'll match an IP address
# or CIDR-masked sub-net.
#
foreach my $arg (@ARGV) {
 for ($arg) {
  if ( $arg eq '-h' )  { &usage; }
  elsif ( $arg eq '-v' )  { $VERBOSE = 1; }
  elsif ( $arg =~ m/^\d+$/ )  { $port  = $arg; }
  elsif ( $arg =~ m/$ipr/ )  { $target = $arg; } 
 }  
}

# if the user forgot to put in a target subnet, goto usage();
#
&usage("Error: no target supplied!") if ("$target" == "");

# Here we'll set the default port to "22", which is SSH, and then we'll
# check to see if a port was submitted on the command line.  If it was,
# we'll override the default with the user submitted value.  We'll do 
# the same for CIDR mask, assuming that if a mask was not passed in, then
# the user wants a single host scanned and use /32 as the default.
#
($ip,$cidr) = split( /\//, $target);
$cidr  = ( $cidr ? $cidr : "32" );
$port  = ( $port ? $port : "22" );

# die if we get an invalid CIDR mask!
#
&usage("Error: Invalid CIDR mask [$cidr]") if ( $cidr > 32 );

# Get the TCP Service name for our port...
#
my $svcname = getservbyport($port,"tcp");

# Convert the CIDR mask into a hex netmask, and calculate the number
# of addresses in the supplied target.
#
# my $mask = 0xffffffff >> $cidr;  # this fails on sol8/perl-5.003
my $size = 1 << ( 32 - $cidr );
my $mask = $size - 1;

#===============================================================================#
# script body below here.
#===============================================================================#

&hr2();
print "Request to scan ${ip}/${cidr} on Port $port ($svcname)","\n";
print "Scanning $size IP Addresses:","\n";

# calculate the lowest IP in the range, by AND-ing the supplied IP 
# address and the netmask, and convert to dotted quad notation.
#
my $lowest      = unpack('N', pack('C4', split '\.', $ip)) & ~$mask;
my $count = 0;
my $scanned = 0;

# based on the $lowest IP in the subnet, let's enumerate all of the
# IP addresses in the supplied $target subnet.
#
my @ips  = map {$lowest++} 0 .. $mask;

# a simple loop to scan each of the ips we mapped.
#
foreach my $addr(@ips) {
 scan_ip($addr);
}

&hr();
print "Found $count hosts listenening on port $port.","\n";
&hr2();

#===============================================================================#
# only subroutines are below here.
#===============================================================================#

sub scan_ip {
 #
 # from the foreach loop above, we've passed in our address.
 # this address is an integer, so we'll need to convert it to
 # a dotted-quad format IP address. then try hostname lookup,
 # and then try opening a socket.
 #
 my $addr = shift;
 my $ipaddr = join( '.', unpack( "C4", pack( "N", $addr ) ) );

 my $hostname = gethostbyaddr(inet_aton("$ipaddr"), AF_INET);
 chomp $hostname;

 if ("$hostname" eq "") {
  $hostname = ( $VERBOSE ? "NXDOMAIN" : " " );
 }

 my $sock = IO::Socket::INET->new(
  PeerAddr => "$ipaddr",
  PeerPort => "$port",
  Timeout => "1",
  Proto => "tcp",
  ) or nosocket("$ipaddr","$hostname") && next;

 if($scanned < 1) { &hr(); }
 
 my $txt = "listening on $port ($svcname)";
 printf('%-15s : %-32s : %-15s', $ipaddr, $hostname, "$txt\n") if($sock);

 close($sock) if($sock);
 $count++;
 $scanned++;
}

sub nosocket { 
 # we reach this sub on unsuccessful socket attempts.
 #
 if($scanned < 1) { &hr(); }
 my $ipaddr = shift;
 my $hostname = shift;

 if($VERBOSE > 0) {
  my $txt = "closed on $port ($svcname)";
  printf('%-15s : %-32s : %-15s', $ipaddr, $hostname, "$txt\n");
 }

 $scanned++;
 next;
}

sub usage { 
 if (@_) {
  print "\n@_\n";
 }
 # this sub() just prints the standard usage information.
 #
 print < [port]
 -h  this message
 -v  verbose output
  format IP/CIDR: 1.1.1.1/32, /32 is default CIDR mask
 [port]  a tcp port between 1 and 65535, 22 is default 

Example: $0 -v 192.168.0.1/24 80
 will scan the 255 addresses in the 192.168.0.0 subnet on port 80

EOT
exit(1);
}

sub hr { 
 # print a row of ---s
 print "-" x 80, "\n";
}

sub hr2 { 
 # print a row of ===s
 print "=" x 80, "\n";
}

# EOF
```

So far this has worked successfully on Solaris 10 Sparc & x64, CentOS 5.5 x64, Solaris 8 Sparc, and Solaris 9 Sparc, and Mac OS X 10.6.


*this post was migrated from blogs.sun.com/tksunw
