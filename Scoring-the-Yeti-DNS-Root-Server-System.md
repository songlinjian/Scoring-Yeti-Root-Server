
# Scoring the Yeti DNS Root Server System

### 1. Introduction

Yeti DNS Project is a live DNS root server system testbed which is designed and operated for experiments and testing. It is curious for many people how it works under a environment which is a pure IPv6, with more than 13 NS servers, with multiple signers, and testing ZSK/KSK rolling. One Key issue here is about the large DNS response, because many Yeti experiments lead to produces large DNS response (even more than 1600 octets).


Why Large DNS response is relevant and How DNS handles the large response is introduced comprehensively by Geoff's recent article in APNIC blog [ref TBD]. (Thanks Geoff!:) In additional in his article, a scoring model is proposed and applied it to evaluate current 13 DNS root servers how they handle large response. Triggers by Geoff's work, an intuitive idea is formed to score the Yeti DNS root server system using the same testing and metrics. The results are summarized in this document. 


### 2. Reply the testing on IANA DNS Root Server

We firstly repeat the testing on IANA DNS root server. It is expected to confirm the test result and looking for any new findings in different vantage points and metrics. Using the same approach we dig queries against each root servers with a long query name for a non-existent domain name like this to get a response of 1268 octets:

    dig aaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa +dnssec @198.41.0.4    +bufsize=4096

And what we see from each root server is shown in Table 1.


| Root | Response size(v4) | Truncate (v4)| Fragment (v4) | TCP MSS (v4) |Response size(v6) | Truncate (v6) | Fragment (v6) | TCP MSS (v6)|
|---|---|---|---|---|---|---|---|---|
|A|1268|N|N|1460/1380|1268|Y(900)|N|1440/1380|
|B|1265|Y|N|1460/1380|1265|Y|N|1440/1380|
|C|1265|N|N|1460/1380|-|-|-|-|
|D|1265|N|N|1460/1380|1265|N|N|1440/1380|
|E|1268|N|N|1460/1380|280|ServFail|ServFail|ServFail|
|F|1265|N|N|1460/1380|1265|N|UDP|1440/1380|
|G|1265|Y|N|1460/1380|1265|Y|N|1440/1380|
|H|1268|N|N|1460/1380|1268|N|N|1440/1220|
|I|1265|N|N|1460/1380|1265|N|N|1440/1380|
|J|1268|N|N|1460/1380|1268|Y(900)|N|1440/1380|
|K|1268|N|N|1460/1380|1268|N|N|1440/1380|
|L|1268|N|N|1460/1380|1268|N|N|1440/1380|
|M|-|-|-|-|1268|N|TCP|1440/1380|
Table 1 – IANA Root Server Response Profile to a large DNS response

The structure of Table 1 is a slightly different from the table in APNIC's blog article. There are additional columns show the exact size of DNS reponse massage from each root server. We list them because we found them are different in 3 octets. A,E,H,J,K,L,M response 1268 octets and others 1265 octets. After some digging, it is found that different root servers behave differently due to the name compression in NSEC record.  In the case of that non-existent name query, 'aaa' is a common label in both NSEC records of root and 'aaa'. Saving 3 bytes by careful name compression is not a huge optimization but in certain cases it can sharply avoid truncation or fragmentation.   

There is another difference when we display the TCP MSS information in the table. In APNIC's article it said inn IPv4 all the root servers offer a TCP MSS of 1,460 octets and the value is 1,440 octets in IPv6. It maybe true if there is no middle-box or firewall edit the TCP MSS intentionally. By observation, the TCP SYN initiated by testing resolver carries the TCP MSS of 1460 octets in IPv4 and 1440 octets in IPv6, but most TCP SYN/ACK responded by root servers are all 1380 octets. As far as we know, TCP MSS of 1380 octets is a special value edited by some firewalls on the path for security reasons (CISCO ASA firewall). 

Let's look at Fragment first! 

Similar with APNIC's finding, F and M are founded do IPv6 fragmentation. It is worthwhile to mention that in our test F only fragments UDP IPv6 packets and M only fragments TCP segment. For F its fragmenting only UDP can be explained that F's IPV6_USE_MIN_MTU option equals 1 and TCP implementation respects the IPV6_USE_MIN_MTU  not sending TCP segments larger than 1280 octets. 

Regarding TCP MSS setting,in APNIC's blog article it is suggested that TCP MSS should be set 1220 octets. It is observed that H already accepted Geoff's suggestion as the time of writing. TCP MSS setting in root server is relevant because there are mainly two risks if TCP MSS is not set appropriately: one is introduced in APNIC's blog article to avoid Path MTU Black Hole in which large TCP segment may be dropped in the middle but ICMP6 PTB message is lost or filtered. Another risk is not covered by that article but IMHO is more relevant . That is the risk that large TCP segment is to be fragmented into IPv6 fragments by root servers if TCP dose not respect IPV6_USE_MIN_MTU socket option and IPV6_USE_MIN_MTU=1 in that server (ref TBD). 

M is fragmenting the TCP segment in this way! It is odd that large UDP response of M  is not fragmented.

As to truncate behavior of Root server, as expected, our test shows the same pattern that in IPv4 B and G truncate the response, in IPv6 A and J join B and G to truncate the response. By definition truncation happens when DNS massage length is greater than that permitted on the transmission channel. The truncation behavior of A, B, G and J  looks odd because they truncate the response even when both query and response specifies a large EDNS0 buffer size. There must be a independent and paralleled routine to determine truncate or not. Obviously for these root operators they prefer to  truncate the response, fall back to TCP than fragmenting a larger response. This preference is stronger in IPv6 and IPv4 (A and J) because the potential problem of fragments missing or filtering in IPv6 is worse than in IPv4 which is explained in detailed in APNIC blog article. ( 900?)

It is observed that there is a Server Failure from E in IPv6 and it happens only in China. It is also observed C in IPv6 and M in IPv4 are unreachable as the time of testing. By trace-route, the problem is spotted by upstream network provides. It is remind us that DNS system especially the Root server system is vulnerable due to security attack or routing problems.

### Test on Yeti DNS Root Server

The next step is to dig the same query against Yeti DNS root server. And what we see from each Yeti root server is shown in Table 2.

| Num  |Operator|Response size|Truncate| Fragment |TCP MSS|
|------|----|-------------|--------|--------- |-------|
|#1   |BII | 1252      |  N    |UDP+TCP |	1440/1380|
|#2   |WIDE| 1255      |  N    |UDP     | 1440/1220|
|#3   |TISF| 1252      |  N    |UDP+ TCP|	1440/1440|
|#4   |AS59715| 1255    |  N    |UDP+ TCP  | 1440/1380|
|#5   |Dahu Group| 1255      |  N    |N		  |	1440/1440|
|#6   |Bond Internet Systems | 1252      |  N    |N|	1440/1440|
|#7   |MSK-IX | 1252      |  N    |UDP+ TCP|	1440/1440|
|#8   |CERT Austria | 1252      |  N    |N|	1440/1440|
|#9   |ERNET | 1252      |  N    |N|	1440/1440|
|#10   |dnsworkshop/informnis | 1255      |  N    |UDP|	1440/1440|
|#11   |CONIT S.A.S Colombia | -      |  -    |-|	-|
|#12   |Dahu Group | 1255      |  N    |N|	1440/1440|
|#13   |Aqua Ray SAS| 1255      |  N    |N|	1440/1380|
|#14   |SWITCH| 1252      |  N    |N|	1440/1220|
|#15   |CHILE NIC | 1252      |  N    |N|	1440/1440|
|#16   |BII@Cernet2| 1252      |  N    |N|	1440/1380|
|#17   |BII@Cernet2 | 1252      |  N    |N|	1440/1380|
|#18   |BII@CMCC | 1252      |  N    |N|	1440/1380|
|#19   |Yeti@SouthAfrica| 1252      |  N    |N|	1440/1440|
|#20   |Yeti@Australia | 1252      |  N    |N|	1440/1440|
|#21   |ERNET| 1252      |  N    |N|	1440/1380|
|#22   |ERNET| 1252      |  N    |N|	1440/1440|
|#23   |dnsworkshop/informnis | 1269      |  N    |N|	1440/1440|
|#24   |Monshouwer Internet Diensten| 1255      |  N    |N|	1440/1440|
|#25   |DATEV | 1255      |  N    |N|	1440/1380|
Table 1 – Yeti Root Server Response Profile to a large DNS response

No Truncation for all Yeti root server. That's means not additional DNS routine designed for TC in Yeti root servers.

For #23 (Bundy?) The response is bigger than others because it does not apply name compression for mname and rname in SOA record which increase 12 octets , and it also compressed the root from one label to two labels.

### Metrics for scoring

### Scoring Yeti Root Servers




