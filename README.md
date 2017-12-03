#  IOC2RPZ - turns your threat intelligence into RPZ feeds.
## Overview

## ioc2rpz vs bind:
- ioc2rpz built to handle RPZ distribution only
- ioc2rpz supports as many RPZ as you need. bind supports only 32 zones per DNS view
- live zones
- indicators via REST API calls
- IOC expiration time
- configurable packet size --> zone transfer optimization
- performace

## How to start ioc2rpz service

## Configuration
1. AXFR update time - full Zone update and rebuild if MD5 is different. IOC added as insert_new (zone, ioc) -> to have an ability support IXFR
2. IXFR update time - incremental zone update
### Config file

[:AXFR:] = url,
[:FDateTime:] = "2017-10-13 13:13:13", [:FDateTimeZ:] = "2017-10-13T13:13:13Z", [:FTimestamp:] = 1507946281
[:ToDateTime:] = "2017-10-13 13:13:13", [:ToDateTimeZ:] = "2017-10-13T13:13:13Z", [:ToTimestamp:] = 1507946281


Action can be: single value, list of tuples. Single value defines a single action for an RPZ. A list depending on the param can define multiple results.
- Single value: "nodata", "nxdomain", "passthru", "drop", "tcp-only", "blockns" ("nsdname", "nsip")
- List:
-- Single action: {"redirect_domain", "www.example.com"}, {"redirect_ip", "127.0.0.1"} - as a response CNAME or A/AAAA records will be returned
-- Multiple actions: {"local_a","127.0.0.1"}, {"local_aaaa", "fe80::1"}, {"local_cname","www.example.com"}, {"local_txt","Just TXT record"}
-- [TODO] An action per source: {"",action,locdata} //default action ,{"source_name",action,locdata}

### Constants - ioc2rpz.hrl


### How the Full (AXFR) and Incremental(IXFR) caches are updated
- AXFR cache always contains prebuilt zones without SOA/NS/TSIG records. Prebuilt means all records are splitted by packets and labels were shortened/zipped.
- If server recieve an AXFR request it retrieve packets from the AXFR cache, add SOA/NS records and TSIG if needed.
- AXFR zones update should be considered as a clean up procedure, which should periodicaly take place. Just to be sure that there is no desynchronization between the sources and the cache.
- For large zones, AXFR updates should be scheduled infrequently to minimize impact on server's performance and ammount of transferred data to all clients.
- All changes if it is possible should be done via incremental zone updates. In that case the AXFR cache will be rebuilt only in case if a zone was updated.
- [TODO] Due to an optimization, only last packet will be rebuilt for new IOCs and relevant and accordant packets for the expired IOCs.
- IXFR cache contains only IOCs and expiration dates. [TODO] and packets ID's (to make it possible rebuild the zone fast).
- RPZ record contains current zone Serial and Serial_IXFR. Serial_IXFR serve as a minimum incremental zone serial which is available for an incremental zone transfer.
- IXFR cache is flushed after full zone update (AXFR). Serial_IXFR = Serial. Clients will recieve full zone update in any case, this is why it is important to have AXFR zone transfer infrequently.
- When IXFR is updated, AXFR cache must be rebuilt.
- If a zone does not support IXFR updates -> it doesn't saved in the IXFR table.
- Live zones are not cached in the AXFR, IXFR caches but the sources (IOCs) can be cached in the hot cache.

### Hot cache
IXFR updates are not cached in the hot cache

## TODO features
- [x] IOC individual expiration time - update time will be used as minimum time for full zone refresh + check the SOA
- [x] Logs: Good timestamp, IP into normal format
- [ ] TSIG Found Key ... 2017-10-24 10:58:58 Bad timestamp ... Valid MAC
- [X] Cache for IXFR (save IOC rules with Exp.Time) and rebuild AXFR cache
-- [X] check AXFR zone update with new records
-- [X] check zone update with expired records
- [X] Incremental zone transfer/IXFR
- [x] Support DNS UDP IXFR/SOA request. IXFR - tcp only
- [x] DNS Notify messages
- [x] Local response for the IOCs
- [x] Hot cache for IXFR IOC/sources - таймштамп округляем до минут
- [ ] http/https/ftp errors handling - source status in the record. If a source is not available - work w/o it
- [ ] Source based on files check by mod.date and size -> read by chunks
- [ ] Performance testing vs bind:
-- [ ] 1 core/8Gb RAM: start time, zone transfer time, zone size, CPU, Memory
---[ ] 100k rules
---[ ] 1M rules
---[ ] 10M rules
-- [ ] 4 cores/32 Gb RAM: start time, zone transfer time, zone size
---[ ] 100k rules
---[ ] 1M rules
---[ ] 10M rules
- [ ] By a signal and DNS request
-- [ ][x] Reread CFG
-- [ ][x] Refresh a zone
-- [ ][x] Refresh all zones
-- [ ][x] Terminate processes/Exit
- [ ] Path for DB
- [ ] If Zone not ready - respond NODATA
- [ ] Statistics per zone
- [ ] Fix IPv6 reversing "cleanup"
- [ ] Container
- [ ] Documentation
- [ ] Add source RPZ
- [ ] Add source SQL
- [ ] Mnesia for storage (and auto creation)
- [ ] Distributed configuration based on mnesia

## Other/optimization TODO
- [ ] Check if we can substitute an A/AAAA record because of IP
- [ ] UDP under supervisor
- [ ] EDNS0 RR
- [ ] Check all TODO in the code
- [ ] Clean up the code & add comments
- [ ] IOC to lowercase - check performance impact
- [ ] Memory optimization for huge zones
- [ ] Do not cache expired IOCs if ExpDateTime<Serial_IXFR / update ExpDateTime if exists
- [ ] Check zones IXFR update from multiple sources
- [X] Move connectors to a new file
- [x] Move DB fun to a new file e.g. ioc2rpz_db
- [x] Move additional fun to ioc2rpz_fun
- [x] Do not save in IXFR cache zonex w/o IXFR
- [x] Create dirs: src, bin, cfg
- [ ] Share IOC between the feedx in IXFR table

## Free threat intel
http://www.malwaredomains.com/?page_id=66
http://mirror1.malwaredomains.com/files/spywaredomains.zones
http://data.netlab.360.com

## Bugs
- [ ] saveZones - doesn't correctly save zones if there a lot of updates. Save strategy based on update size and time.
- [x] Clean cache tables if a zone was removed from CFG --- данные удаляются, но таблица особо не читстится

### References
- Domain Name System (DNS) IANA Considerations
https://tools.ietf.org/html/rfc6895
- Domain Names - Implementation and Specification
https://tools.ietf.org/html/rfc1035
- Incremental Zone Transfer in DNS
https://tools.ietf.org/html/rfc1995
- DNS Response Policy Zones (RPZ)
https://tools.ietf.org/html/draft-ietf-dnsop-dns-rpz-00
https://tools.ietf.org/html/draft-vixie-dns-rpz-02
- Secret Key Transaction Authentication for DNS (TSIG)
https://tools.ietf.org/html/rfc2845
- HMAC: Keyed-Hashing for Message Authentication
https://tools.ietf.org/html/rfc2104
- HMAC SHA TSIG Algorithm Identifiers
https://tools.ietf.org/html/rfc4635
- DNS Transport over TCP - Implementation Requirements
https://tools.ietf.org/html/rfc5966
- A Mechanism for Prompt Notification of Zone Changes (DNS NOTIFY)
https://tools.ietf.org/html/rfc1996