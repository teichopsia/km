km
==
kehlarn's prefix mapping swiss knife

this was a network tool I wrote long ago for dealing with RADB objects
to build acls. it is common to merge such prefixes where possible to
reduce the acl size for some vendors where size is an issue or perf a
concern.

(apologies to google for picking on them. they had the right sized
examples and a duplicate I needed to cite. this project did start
from dealing with your deaggregation so it's only fair I figure)

example, google's v4 list is rather large from deaggregation:

	$ echo '!gas15169'|nc whois.radb.net. 43|sed -ne 2p|tr ' ' '\n'|wc -l
	7417

dump all v4 routes for google from radb and give us the minimal subset

	$ echo '!gas15169'|nc whois.radb.net. 43|km fold -r -|wc -l
	69

do the same in v6 land as it is a shorter example:

	$ echo '!6as15169'|nc whois.radb.net. 43|sed -ne 2p|tr ' ' '\n'|wc -l
	228

and fold it as well:
	
	$ echo '!6as15169'|nc whois.radb.net. 43|km fold -r -
	2001:1900:2292::/48
	2001:4860::/32
	2401:fa00::/32
	2404:6800::/32
	2404:f340::/32
	2600:1900::/28
	2604:31c0::/32
	2606:73c0::/32
	2607:f8b0::/32
	2620:0:1000::/40
	2620:33:c000::/48
	2620:120:e000::/40
	2620:15c::/36
	2800:3f0::/32
	2a00:1450::/32
	2a00:79e0::/31
	2c0f:fb50::/32

17 is much simpler no? granted in both of these you'll want to crosscheck route objects to
confirm that is their entire space. sometimes arin will let you announce more/less depending
on the allocations. ymmv depending on LIR.

sometimes you want these aggregates for other tests though so what if you want to search a
logfile for these without maintaining static lists and tedious greps? let's scan that for
the old bbn /8 but exclude the googledns/24:

	$ echo '!gas15169'|nc whois.radb.net. 43|sed -ne 2p|tr ' ' '\n'|km grep -i 8.0.0.0/8  -x 8.8.8.0/24 -|km fold -
	8.8.4.0/24
	8.15.202.0/24
	8.34.208.0/20
	8.35.192.0/20

same idea but in v6 land:

	$ echo '!6as15169'|nc whois.radb.net. 43|sed -ne 2p|tr ' ' '\n'|km grep -i 2620::/16 -x 2620:120::/32 -
	2620:0:1000::/40
	2620:15c::/36
	2620:33:c000::/48

or maybe you want to make an acl out of it?

	$ echo '!gas15169'|nc whois.radb.net. 43|sed -ne 2p|tr ' ' '\n'|km grep -i 8.0.0.0/8  -x 8.8.8.0/24 -|km fold -|km acl --acl 8675309 -
	/ip firewall address-list remove [find list=8675309]
	/ip firewall address-list add list=8675309 address=8.8.4.0/24
	/ip firewall address-list add list=8675309 address=8.15.202.0/24
	/ip firewall address-list add list=8675309 address=8.34.208.0/20
	/ip firewall address-list add list=8675309 address=8.35.192.0/20

or read rpsl directly for us: (ignored in grep mode)

	$ echo '!gas15169'|nc whois.radb.net. 43|km fold -r -x 0/0 -i 8.0.0.0/8 - |km acl -a google8675309 -i 8.8.0.0/16 -x 0.0.0.0/0 -
	/ip firewall address-list remove [find list=google8675309]
	/ip firewall address-list add list=google8675309 address=8.8.4.0/24
	/ip firewall address-list add list=google8675309 address=8.8.8.0/24

and probably the most useful function still in testing:

	$ echo '!gas15169'|nc whois.radb.net. 43|km grep -i 104.134.0.0/16 |km fold -
	104.134.92.0/24
	104.134.128.0/17

rpsl mode doesn't work in grep since the reply from irrd is a single liner usually
which doesn't make a lot of sense in a grep context since you're probably scanning
for specific prefixes in a range. swizzle how you like though

	echo '!gas15169'|nc whois.radb.net. 43|tr ' ' '\n'|km grep -Hni 104.134.128.0/22 -
	-:2142:104.134.128.0/17
	-:2143:104.134.128.0/23
	-:2144:104.134.128.0/24
	-:2145:104.134.129.0/24
	-:2146:104.134.130.0/23
	-:2147:104.134.130.0/24
	-:2148:104.134.131.0/24
	-:7256:104.134.128.0/20
	-:7257:104.134.128.0/22
	-:7259:104.134.128.0/18
	-:7263:104.134.128.0/21
	-:7292:104.134.128.0/19

some statistics in case you wonder how the folding works:

	$ echo '!gas15169'|nc whois.radb.net. 43|km stat -r -
	warning - filters ignored for now
	before:  7416
	distal:  7316
	prox:    100
	folds:   31
	passes:  5
	dup:     1
	after:   69

distal here means specifics; proximal is aggregates; folds are adjacent bitlens that
were collapsed from a N to N-1 prefix. passes is the number of iterations at depth
to complete. dup is a duplicate in their route objects (woopsie!) and after is the
final output

	$ echo '!gas15169'|nc whois.radb.net. 43|km fold -rD - >/dev/null
	duplicate: 104.132.222.0/24

not atypical - level3 registered a proxy route object in their database
as they may filter radb as a source during acl builds

those needing other vendor acls probably want to use this in conjunction with
[bgpq3](https://github.com/snar/bgpq3) for best fit. i purposely just eject
mikrotik as it isn't supported there and hide my other acl tooling mess
