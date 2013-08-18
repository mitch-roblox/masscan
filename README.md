# MASSCAN: Mass IPv4 port scanner

This is a port scanner. It spews out packets at a high rate, then catches any
responses asynchronously. Because it's asynchronous, it's a lot faster than 
`nmap` -- and a lot less feature rich.

This is a 48-bit scanner: scanning all ports (16-bits) on all
IPv4 addresses (32-bits). It's also useful on smaller problems, such as the
10.x.x.x address space within a company.

It randomizes the IPv4+port combination, whereas `nmap only randomizes the
IPv4 address. This is so that we can send out 10-million packet per second
when scanning the entire Internet, but the owner of a Class C network will
only see 1 packet per second comming in.


# Building

On Debian/Ubuntu, it goes something like this:

	$ git clone https://github.com/robertdavidgraham/masscan
	$ cd masscan
	$ sudo apt-get install build-essential
	$ sudo apt-get install libpcap-dev
	$ make
	$ make regresss

This puts the program in the 'bin' subdirectory.

On Windows, use the VisualStudio 2010 project.

On Mac OS X, once you've installed a developer environment, you
should be able to likewise just "make; make regress". Detecting
the network adapter is currently broken, so you'll get errors 
telling you what to manually configure when running the program.

On BSD's, it oughta be close to working, but I haven't tried it
yet. I'd like to see what 'netmap' can do with it -- in theory
should be a lot faster than Linux.



# Regression testing

The project contains a built-in self-test:

	$ make regress
	bin/masscan --regress
	selftest: success!

If the self-test fails, the program returns an exit code of '1' and an
error message particular to which module and subtest failed.

NOTE: The regression test is completely offline: it doesn't send any packets.
It's just testing the invidual units within the program. I plan to create
an online test, where a second program listens on the network to verify
that what's transmitted is the same thing that was specified to be sent.


# Usage

Usage is similar to `nmap`, such as the following scan:

	# masscan -p80,8000-8100 10.0.0.0/8

This will:
* scan the 10.x.x.x subnet, all 16 million addresses
* scans port 80 and the range 8000 to 8100, or 102 addresses total
* print output to <stdout> that can be redirected to a file

To see the complete list of options, use the `--echo` feature. This
dumps the current configuration and exits. This ouput can be used as input back
into the program:

	# masscan masscan -p80,8000-8100 10.0.0.0/8 --echo > xxx.conf
	# masscan -c xxx.conf --rate 1000


# Comparison with Nmap

Where reasonable, every effort has been taken to make the program familiar
to `nmap` users, even though it's fundamentally different. Two important
differences are:

* no default ports to scan, you must specify `-p <ports>`
* target hosts are IP addresses or simple ranges, not DNS names, nor 
  the funky subnet ranges `nmap` can use.

You can think of `masscan` as having the following settings permanently
enabled:
* `-sS`: this does SYN scan only (currently, will change in future)
* `-Pn`: doesn't ping hosts first, which is fundamental to the async operation
* `-n`: no DNS resolution happens
* `--randomize-hosts`: scan completely randomized
* `--send-eth`: sends using raw `libpcap`

If you want a list of additional `nmap` compatible settings, use the following
command:

	# masscan --nmap

# Tips on reading the code

The file `main.c` contains the `main()` function, as you'd expect. Also,
this file contains the main scanning thread that spews packets, as well
as the catching thread that catches responses. This is the core functionality
of the program, everything else is secondary.


# Transmit rate (IMPORTANT!!)

This program spews out packets very fast. On Windows, or from VMs,
it can do 300,000 packets/second. On a Linux (no virtualization) it'll
do 1.6 million packets-per-second. That's fast enough to melt most networks.

Note that it'll only melt your own network. It randomizes the target
IP addresses so that it shouldn't overwhelm any one network.

By default, the rate is set to 100 packets/second. To increase the rate to
a million use something like "--rate 1000000".


# How it works

Here are some notes on the design.

## Spews out packets asynchronously

This is an **asynchronous** program. That means it has a single thread
that spews out packets indiscriminately without waiting for responses.
Another thread collects the responses.

This has lots of subtle consequences. For example, you can't use this
program to scan the local subnet, because it can't ARP targets and 
wait for responses -- that's synchronous thinking.

## Randomization

Packets are sent in a random order, randomizing simultaneously the IPv4
address and the port.

In other words, if you are scanning the entire Internet at a very fast
rate, somebody owning a Class C network will see a very slow rate of
packets.

The way we do this randomization is that we assign every IP/port combo
a sequence number, then use a function that looks like:

	seqno = translate(seqno);

The `translate()` function uses some quirky math, based on the LCG PRNG
(the basic random number generator we are all familiar with) to do this
translation.

The key property here is that we can completely randomize the order
without keeping any state in memory. In other words, scanning the 
entire Internet for all ports is a 48-bit problem (32-bit address and
16-bit port), but we accomplish this with only a few kilobytes of
memory.

