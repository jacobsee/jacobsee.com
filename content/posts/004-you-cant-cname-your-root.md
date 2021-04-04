---
title: "You can't CNAME your root domain (but also, you can!)"
description: "A reflection on reactions to pandemic restrictions"
date: 2021-04-04
---

I ran across an interesting situation recently where I wanted to take a friend's domain (that they had given me access to on their Google Domains account) and host a landing page on the same server that the rest of my sites run on - with one small catch:

I didn't want to have to maintain a separate `A` record for their domain pointing to my server's IP address. I already maintain an `A` record for my own domain. I thought that I should be able to `CNAME` their domain to mine, and have the IP address of my own `A` record simply set the course for this new domain as well.

_It turns out that this breaks some rules under normal DNS conventions._

Specifically, under the authoritative RFCs[^0] [^1]:

- `CNAME` records, when defined, must be the only record for a given hostname.
- `SOA` and `NS` records _must_ exist at the domain root.

These two constraints would contradict each other - so `CNAME` records are not technically allowed at the domain root!

The thing is - This is not a technical limitation. This is merely a limitation in the specification, written during a time when the modern Internet (and the infrastructure that we use to drive it) simply did not look anything like what it does today.

This leaves us with an interesting situation. Google Domains has not implemented any sort of workaround for this, so using Google as the nameserver for my friend's domain was not an option. _However_, some other providers have implemented what they call `ALIAS` records, `ANAME` records, or virtual `CNAME` records!

These pseudo-records are a fantastic piece of DNS server trickery. Instead of replying to a DNS request for the root hostname with a `CNAME` record (which, remember, breaks the rules) - they have the DNS server perform its own lookup of the `ALIAS`ed domain on-the-fly and return that IP to the requesting client as a normal `A` record, indistinguishable from any other _totally normal_ `A` record on a domain root!

Out of the public DNS providers that support this "virtual `CNAME`" service, [CloudFlare](https://cloudflare.com) is the one that I decided to use (since I use CloudFlare for other things as well). They call this trickery "`CNAME` Flattening", and they have [written their own detailed post about it here](https://blog.cloudflare.com/introducing-cname-flattening-rfc-compliant-cnames-at-a-domains-root/). In the CloudFlare DNS panel, once you enable `CNAME` flattening, you can treat `CNAME` records as if they _can_ exist at the root. Create the `CNAME` record at the root, and it will automatically handle the flattening (i.e. conversion to an `A` record) for you!

![CloudFlare's CNAME Flattening option in their DNS control panel](/images/cloudflare-cname-flattening.png)

This made for an elegant implementation of my desired DNS configuration. I used the Google Domains panel to point the `NS` records for my friend's domain at CloudFlare, and from there was able to virtually `CNAME` that domain to point at my main domain (and thus be hosted by my same web server), without breaking any DNS rules!

[^0]: [RFC1912](https://tools.ietf.org/html/rfc1912)
[^1]: [RFC2181](https://tools.ietf.org/html/rfc2181)
