# Find Your IP Nameserver

The nameserver works for a given domain, and answers for any defined NS entries.

The CLI only takes one argument, the path to the config file. Defaults to `/usr/local/etc/ns.conf`

It'll answer for the following records:

* some.domain
  * A - from config file (rrA)
  * AAAA - from config file (rrAAAA)
  * NS - defaults to a single NS, ns.some.domain where A=rrA and AAAA=rrAAAA
  * DNSKEY - from config file - if this and 'ksk' are present, we will sign queries with DO set
  * TXT - replies with a TXT record containing "query source" and the source IP the query came from
    * Adds a TXT record containing "client subnet" and the EDNS CLIENT-SUBNET details
    * Adds a TXT record containing "edns size" and the EDNS size details
* <any ns>.some.domain - from the config file
  * A - the A record
  * AAAA - the AAAA records
* <random>.some.domain
  * A - rrA, as above
  * AAAA - rrAAAA as above
  * TXT - as above, but also sets
    * ipdns_<random> to source IP in memcache
    * ipdns_client_<random> to EDNS CLIENT-SUBNET, if set
    * ipdns_edns_size_<RANDOM> to EDNS Size, if set

It will also respond for hostname.bind, version.bind and author.bind TXT in CHAOS class.

## MemCache

At the moment, it's hard coded to use memcache on the local server on the default port - this will get pulled out into a config file item, but network latency *could* result in a race condition between the NS sticking the source IP in memcache and the PHP looking in memcache for the entry a split second later.

## DNSSEC

If you've given it a ksk (private key) and a rrDNSKEY to publish, it'll sign replies, generating relevant RRSIG and NSEC entries.

## Performance

No idea. Currently untested. Unlikely to be properly fast.

## Config File

The config file is JSON format, look at the supplied example.

* debug : sets the debug level
    * 0 : off
    * 1 : on
    * 2 : verbose
* domain : the domain that is delegated to the IP the code is listening on
* ksk : the path to the KSK private key
* rrDNSKEY : the DNSKEY data. No sanity checking is done on format, but won't work unless correctly formatted.
* localAddr : the address we should listen on. defaults to :: (all of any protocol)
* localPort : the port we should listen to. defaults to 53, which of course needs root (<1024)
* logFile : the file to log to. blank for no logging.
* rrNS : an array of hashes of nameservers. name is minimum, A and/or AAAA for in bailiwick / glue
  defaults to ns.$domain with rrA and/or rrAAAA
* mname : the SOA master nameserver name. defaults to the first rrNS entry
* rname : the SOA responsible person contact. defaults to hostmaster.$domain
* ttlStatic : the TTL for static entries (A, AAAA, NS, SOA, DNSKEY). Defaults to 30s.
* ttlDynamic : the TTL for dynamic entries (TXT). Defaults to 5s.
* ttlSoaMin : the SOA Minimum (TTL for negative caching). Defaults to $ttlStatic
* memcacheServer : the memcache server's IP. Defaults to 127.0.0.1
* memcachePort : the memcache server's port. Defaults to 11211

## Notes

### ANY QTYPE

We don't like ANY queries. So, ANY received via UDP gets NOERROR with TC set. ANY received via TCP gets REFUSED.

## Find Your IP Webserver

The nameserver is an additional component to the [Find Your IP Webpage](https://bitbucket.org/karldyson/find-your-ip-webpage) but can also work on it's own.

The Find Your IP Webpage makes up a 16 character random string and looks it up <string>.some.domain

If this nameserver is serving some.domain, it will respond for <random>.some.domain but also make relevant entries in memcache.

So, the process works like this:

* Browser runs JS, makes up 16 character string.
* Tries to fetch <string>.some.domain/<some/path>?json
* Looks up <string>.some.domain in order to do that
* Nameserver gets request, stores ID and source IP in memcache, responds with A and/or AAAA
* Browser requests <string>.some.domain/<some/path>?json from resolved IP
* PHP looks in memcache and generates JSON content for AJAX request and returns content including source IP for DNS lookup
* Browser renders in relevant <DIV>

## License / Warranty

This code is Copyright 2017 Karl Dyson.

You can use it for personal use if it's useful.

You must contact me for permission if you wish to use it for commercial use.

You must contact me for permission to use it for non-profit use.

Any use must be credited.