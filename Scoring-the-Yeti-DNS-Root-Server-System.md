
# Scoring the Yeti DNS Root Server System

### 1. Introduction

[Yeti DNS Project](https://yeti-dns.org) <sup>[1]</sup> is a live DNS root server system testbed which is designed and operated for experiments and testing. Many people are curious how it works under an environment which is pure IPv6, with more than 13 NS servers, with multiple signers, and testing ZSK/KSK rolling. One key issue here is large DNS responses, because many Yeti experiments lead to large DNS responses - even more than 1600 octets.


Why large DNS responses are relevant and how DNS handles large responses is introduced comprehensively by Geoff's articles published in the APNIC blog [_Evaluating IPv4 and IPv6 packet fragmentation_](https://blog.apnic.net/2016/01/28/evaluating-ipv4-and-ipv6-packet-frangmentation/)  <sup>[2]</sup>  and [_Fragmenting IPv6_](https://blog.apnic.net/2016/05/19/fragmenting-ipv6/)  <sup>[3]</sup>  (thanks Geoff!). More recently, Geoff published a pair of articles examining the root servers system behavior, [_Scoring the Root Server System_](https://labs.apnic.net/?p=915)  <sup>[4]</sup>  and [_Scoring the DNS Root Server System, Pt 2 - A Sixth Star?_](https://labs.apnic.net/?p=924)  <sup>[5]</sup> . In these articles, a scoring model is proposed and applied it to evaluate current 13 DNS root servers how they handle large response.

Triggered by Geoff's work [4] and [5], we formed the idea of scoring the Yeti DNS root server system using the same testing and metrics. The results are summarized in this document. 


### 2. Repeat the Tests on the IANA DNS Root Servers

We first repeat the testing on the IANA DNS root servers. It is expected to confirm the test results, but may show some new findings from different vantage points and metrics. Using the same approach we use the [<tt>dig</tt>](https://en.wikipedia.org/wiki/Dig_(command)) command to send queries against each root servers with a long query name for a non-existent domain name like this to get a response of 1268 octets:

    dig aaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa +dnssec @198.41.0.4    +bufsize=4096

And what we see from each root server is shown in **Table 1**.


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
**Table 1** – IANA Root Server Response Profile with large DNS responses

The structure of **Table 1** is a slightly different from the table in APNIC's blog article. There are additional columns to show the exact size of DNS reponse messages from each root server. We list them because we found them are different by 3 octets. A,E,H,J,K,L,M have responses of 1268 octets and others 1265 octets. After some digging, we found that different root servers behave differently due to the name compression of the NSEC record.  In the case of our non-existent name query, 'aaa' is a common label in both NSEC records of root and 'aaa'. Saving 3 bytes by careful name compression is not a huge optimization but in certain cases even 3 bytes could avoid truncation or fragmentation.   

There is another difference when we display the TCP MSS information in the table. In APNIC's article it said that in IPv4 all the root servers offer a TCP MSS of 1460 octets and the value is 1440 octets in IPv6. It may be true if there is no middle-box or firewall which changes the TCP MSS intentionally. In our observation, the TCP SYN initiated by testing resolver carries the TCP MSS of 1460 octets in IPv4 and 1440 octets in IPv6, but the TCP SYN/ACK responded by root servers are all 1380 octets. As far as we know, TCP MSS of 1380 octets is a special value used by some firewalls on the path for security reasons (CISCO ASA firewall). 

Let's look at fragments! 

As with APNIC's finding, we found that F and M fragment IPv6. It is worthwhile to mention that in our test F only fragments UDP IPv6 packets and M only fragments TCP segment. F fragmenting only UDP can be explained that F's DNS software sets <tt>IPV6_USE_MIN_MTU</tt> option to 1 and the TCP implementation respects the <tt>IPV6_USE_MIN_MTU</tt>, so does not send TCP segments larger than 1280 octets.

TCP MSS setting,in APNIC's blog article it is suggested that TCP MSS should be set 1220 octets. It is observed that H already accepted Geoff's suggestion as the time of writing. TCP MSS setting in root server is relevant because there are mainly two risks if TCP MSS is not set properly. One introduced in APNIC's blog article is Path MTU (PMTU) Black Hole in which large TCP segment may be dropped in the middle but ICMP6 PTB message is lost or filtered. Another risk is not covered by that article but is also relevant. That is the behavior where the IP packets TCP sent for segments are fragmented by root servers if TCP does not respect the <tt>IPV6_USE_MIN_MTU</tt> socket option and <tt>IPV6_USE_MIN_MTU=1</tt> in that server. Please see [TCP and MTU in IPv6](https://yeti-dns.org/resource/2016workshop/5.tcp-mtu.pdf) <sup>[6]</sup> , presented by Akira Kato at the 2016 Yeti workshop. 

M is fragmenting the TCP segment in this way! It is odd that large UDP responses of M are not fragmented, and unknown why M fragments only TCP segments.

As for the truncation behavior of root servers, our test shows the same pattern that in IPv4 B and G truncate the response, in IPv6 A and J join B and G to truncate the response. By definition truncation happens when DNS message length is greater than that permitted on the transmission channel. The truncation behavior of A, B, G and J looks odd because they truncate the response even when both query and response specifies a large EDNS0 buffer size. There must be a separate method to determine whether to truncate or not. Obviously for these root operators they prefer to  truncate the response and fall back to TCP rather than fragmenting a larger response. This preference is actually stronger in IPv6 and IPv4, as A and J send truncated packets of 900 octets, possibly because the potential problem of fragments missing or filtering in IPv6 is worse than in IPv4 (as explained in detailed in APNIC blog article).

We observe that there is a Server Failure response (<tt>SERVFAIL</tt>) from E in IPv6 which happens only in China. We also observed C in IPv6 and M in IPv4 are unreachable as the time of testing. By trace-route, the problem is spotted by upstream network provides. It reminds us that DNS system - even the root server system - is vulnerable to network attacks or routing problems.

### 3. Tests on the Yeti DNS Root Servers

The next step is to <tt>dig</tt> the same query against Yeti DNS root servers. And what we see from each Yeti root server is shown in **Table 2**.

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
**Table 2** – Yeti Root Server Response Profile to a large DNS response

We see no truncation for any Yeti root server. That's means none of the Yeti servers have truncation logic apart from the server default truncation, based on the EDNS buffer sizes.

We notice that #2 and #14 accept Geoff's suggestion to change TCP MSS to 1220 and reduce the risk for TCP segment fragmentation (although perhaps they were configured this way before this recommendation).

For #23 (running Microsoft DNS), the response is bigger than others because it does not apply name compression for the <tt>mname</tt> and <tt>rname</tt> fields in the SOA record - leading to an increase of 12 octets, and it also name compression on the root label itself, resulting in a bigger packet.

Note that currently Yeti implements [MZSK](https://github.com/BII-Lab/Yeti-Project/blob/master/doc/Experiment-MZSK.md)  <sup>[7]</sup>  which produces large DNS responses due to multiple ZSKs. By querying for DNSKEY records with their DNSSEC signature, all Yeti servers response with a DNS message size up to 1689 octets and fragment the UDP response. When the <tt>+tcp</tt> option is added to <tt>dig</tt> - performing the DNS query via TCP - the result in the "Fragment" column is the same as that in Table 2 (#1, #3, #4, #7 fragment TCP segments). So in Yeti's case there is a trade-off between whether to truncate the large responses or to fragment them. There is no way to avoid the cost brought by the large response (1500+ octets) with the existing DNS protocol and implementations. However, some proposals are made to address the problem by [DNS message fragments](https://tools.ietf.org/html/draft-muks-dns-message-fragments-00) <sup>[8]</sup>  or always transmitting the large DNS response with connection-oriented protocols like [TCP](https://tools.ietf.org/html/draft-song-dnsop-tcp-primingexchange-00) <sup>[9]</sup> or [HTTP](https://tools.ietf.org/html/draft-ietf-dnsop-dns-wireformat-http-00) <sup>[10]</sup> .


### 4. Metrics for Scoring

Like Geoff, we can use a similar five star rating system for Yeti root
operators. Of course, we won't use the IPv4 ratings, since Yeti is
IPv6-only.  We can "borrow" 3 of Geoff's stars:

* If the IPv6 UDP packet is sent without fragmentation for packets up
  to 1,500 octets in size, then let’s give the server a star.
* If the IPv6 UDP packet is sent without truncation for IPv6 packet
  sizes up to 1,500 octets, then let’s give the server a star.
* If the offered IPv6 TCP MSS value is no larger than 1,220 octets,
  then let’s give the server another star.

We can suggest two more stars:

* If the IPv6 TCP packets are sent without IP fragmentation, we will
  give the server a star.
* If the server compresses all labels in the packet, we give the
  server a star. (Not technically the "penalize Microsoft" star, but
  in practice it is.)

### 5. Scoring Yeti Root Servers

Using this system we rate the Yeti servers.

| Num  | Stars | Comments |
|------|-------|----------|
|#1    | &#9733;&#9733; | Fragments both UDP and TCP, using TCP MSS of 1380 |
|#2    | &#9733;&#9733;&#9733;  | Fragments UDP, do not compresses all labels in the packet |
|#3    | &#9733;&#9733; | Fragments both UDP and TCP, using TCP MSS of 1440 |
|#4    | &#9733; | Fragments both UDP and TCP, using TCP MSS of 1380,do not compresses all labels in the packet |
|#5    | &#9733;&#9733;&#9733; | Using TCP MSS of 1440,do not compresses all labels in the packet |
|#6    | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1440 |
|#7    | &#9733;&#9733; | Fragments both UDP and TCP, using TCP MSS of 1440 |
|#8    | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1440 |
|#9    | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1440 |
|#10   | &#9733;&#9733;&#9733; | Using TCP MSS of 1440,do not compresses all labels in the packet |
|#11   | [_nul points_](https://en.wiktionary.org/wiki/nul_points) | Server not responding during the test. |
|#12   | &#9733;&#9733;&#9733; | Using TCP MSS of 1440,do not compresses all labels in the packet |
|#13   | &#9733;&#9733;&#9733;| Using TCP MSS of 1380,do not compresses all labels in the packet |
|#14   | &#9733;&#9733;&#9733;&#9733;&#9733; | Our only 5-point server! |
|#15   | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1440 |
|#16   | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1380 |
|#17   | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1380 |
|#18   | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1380 |
|#19   | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1440 |
|#20   | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1440 |
|#21   | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1380 |
|#22   | &#9733;&#9733;&#9733;&#9733; | Using TCP MSS of 1440 |
|#23   | &#9733;&#9733;&#9733; | Using TCP MSS of 1440, doesn't compress SOA fully. |
|#24   | &#9733;&#9733;&#9733; | Using TCP MSS of 1440,do not compresses all labels in the packet |
|#25   | &#9733;&#9733;&#9733; | Using TCP MSS of 1380,do not compresses all labels in the packet |
**Table 3** – Starry View of Yeti Servers

If we can make setting the TCP MSS at 1220 a common practice then we
should have a lot of "5 star" Yeti servers.

### 6. Conclusion

We found it interesting to replicate APNIC's results, and were happy
to be able to use similar ratings across the Yeti servers. 

Regarding the DNS large response issue in general, operator can adopt Geoff's suggestions on the `IPV6_USE_MIN_MTU` option, TCP MSS settings, and DNS message compression. However, good solutions are still unknown for issues like IPv6 DNS fragmentation in UDP for very large response. It is still unknown whether is it better to truncate or fragment the large response... more attention is needed in the network community because we are going to build the whole Internet on IPv6 infrastructure.       

As always,lease let us know what you think and what you'd like for us to look
at in either the IANA or Yeti root servers.

### 7. Reference

[1] [_Yeti DNS Project_](https://yeti-dns.org), www.yeti-dns.org 

[2] Geoff Huston, [_Evaluating IPv4 and IPv6 packet fragmentation_](https://blog.apnic.net/2016/01/28/evaluating-ipv4-and-ipv6-packet-frangmentation/), APNIC blog, January 2016.

[3] Geoff Huston, [_Fragmenting IPv6_](https://blog.apnic.net/2016/05/19/fragmenting-ipv6/)， APNIC blog, May 2016.

[4] Geoff Huston, [_Scoring the Root Server System_](https://blog.apnic.net/2016/11/15/scoring-dns-root-server-system/),APNIC blog, November 2016.

[5] Geoff Huston, [_Scoring the DNS Root Server System, Pt 2 - A Sixth Star?_](https://blog.apnic.net/2016/12/12/scoring-dns-root-server-system-pt-2-sixth-star/), APNIC blog, December 2016.

[6] Akira Kato, [_TCP and MTU in IPv6_](https://yeti-dns.org/resource/2016workshop/5.tcp-mtu.pdf) , Yeti DNS Workshop, November 2016

[7] Shane Kerr and Linjian Song, [_Multi-ZSK Experiment_](https://github.com/BII-Lab/Yeti-Project/blob/master/doc/Experiment-MZSK.md), Yeti DNS Project

[8] Mukund Sivaraman, Shane Kerr and Linjian Song, [_DNS message fragments_](https://tools.ietf.org/html/draft-muks-dns-message-fragments-00), IETF Draft, July 20, 2015

[9] Linjian Song,Di Ma [_Using TCP by Default in Root Priming Exchange_](https://tools.ietf.org/html/draft-song-dnsop-tcp-primingexchange-00),IETF Draft, November 26, 2014

[10] Linjian Song, Paul Vixie, Shane Kerr and Runxia Wan [_DNS wire-format over HTTP_](https://tools.ietf.org/html/draft-ietf-dnsop-dns-wireformat-http-00), IETF draft,September 15, 2016 
