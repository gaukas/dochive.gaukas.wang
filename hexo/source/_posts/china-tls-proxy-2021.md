---
title: How China Telecom incapacitated TLS Proxies
categories:
- reports
- security
tags: 
- china
- tls
- proxy
- censorship 
date: 07/28/2021 18:50
---
_Prev title: How China Detects and Blocks TLS Proxies_

## Abstract
Starting in 2021 June, the Great Firewall, operated by the GOV of CHN, started blocking and interfering with the widely used TLS proxies used by CHN internet users to bypass the well-known internet censorship in CHN. We have found evidence suggesting a hidden reputation system against overseas IPs and exposed a common discrepancy of modern TLS proxy software implementations.

## Introduction
Starting June 2021, we have seen massive reports indicating TLS proxies started to malfunction in CHN. While many proxy servers got their 443 port timed out in CHN, some other impacted servers still have their accessible HTTPS decoy websites reachable from CHN. This unexpected behavior may indicate that CHN is testing new technology to sabotage TLS proxy without harming normal TLS-based web browsing.

In this report, we summarized the efforts we put into the investigation of this incident, including our guesses, experiments with their data, verifications tool(s), and our final conclusion.

### Proxy usage in CHN
It is well known that CHN has the most censored Internet in the world. While most of the Internet users in CHN are still satisfied with their domestic internet ecosystem, there are still many people who try to access various services offered in only other countries. The activity to circumvent the infrastructure-level internet censorship of CHN called the Great Firewall(GFW) is known as Wall-breaking, or Fan-Qiang (翻墙, literally, climb over the wall) in CHN.

There have been many popular traffic proxying tools or protocols used mostly/only for wall-breaking, ranked according to their age:
- OpenVPN
- Shadowsocks/ShadowsocksR
- Project V
- XTLS
- Trojan-GFW/Trojan-Go

And just as new wall-breaking tools emerges in China, GFW adapts to attack the latest popular circumventions.

## Background
The attack started with being observable regionally and quickly spreads to nationwide in CHN. Only Internet users routing via **AS4134 CHINANET-BACKBONE**, which are majorly China Telecom users, were impacted by this incident. Hosts that does not route through AS4134 were not impacted.

Many of self-reported cases claimed that the attack is targetting only the TLS Proxying behavior, leaving the normal HTTPS web browsing unharmed. If this is true, then such discriminating behavior could be an indication of the GFW being able to identify TLS proxy traffic from HTTP traffic, which directly threatens the safety of TLS proxy users. 

Popular TLS proxies are usually designed be detection-resistant with the replay-attack being disabled by TLS itself and a fallback behavior for those who failed to authenticate as a proxy user. However, even TLS connections could be fingerprinted and be discriminated based on that. According to [tlsfingerprint.io](https://tlsfingerprint.io), the TLS ClientHello messages are not random at all. And Censor could be discriminating many other factors, such as negotiated ALPN, SNI value, etc. 

## Threat Model
In the first experiment we attempted to examine the discrimination on TLS proxy servers versus on HTTPS protocol which serves websites over TLS. The client will connects to the same TLS server as a proxy user (with application layer authentication) and as an HTTPS visitor (without application layer authentication), with _tcpdump_ being used to capture packets on wire. 

### The attack against TLS proxy
On the test node simulating a Chinese TLS proxy client, in pcap we observed massive amount TLS handshake failures caused by _RST_ packets. We believe it prevented TLS proxy from working correctly. 

On the TLS proxy server the test node connects to, we observed no abnormal traffic.

### The attack against HTTPS protocol
On the test node simulating a Chinese HTTPS client, in pcap we observed effectively the same amount of TLS handshake failures caused by _RST_ packets. We believe it should effectively prevent HTTPS protocol from working correctly, even though such behavior is not reported by web browser users. 

### Analysis
Thus we have confirmed that the attack is not discriminating TLS proxy traffic among all other HTTPS traffic. But instead, a major design discrepancy in between TLS proxy applications and web browsers caused the inconsistent user experience. Specifically when examining the attack against, on average we observed more than one ClientHello message being transmitted per HTTPS server we connected to. We would explain this behavior as an automatic retry after failed TLS handshake caused by _RST_. On the other hand, TLS proxy clients do not implement this retry mechanism but instead will fail immediately upon _RST_ being received. 

## Other evaluations
In addition we evaluated the potential impact of other fingerprintable TLS factors on TLS handshake successful rates, among many target hosts running TLS proxy/web servers.

### TLS ClientHello fingerprint
| ![clienthello](/images/china-tls/clienthello.png) |
| :--: |
| Fig 1. various targets(hosts) with various popular TLS ClientHello Messages |

The data does not show an indication of ClientHello-fingerprint-based discrimination.

### Negotiated ALPN
| ![alpn](/images/china-tls/alpn.png) |
| :--: |
| Fig 2. various negotiatede ALPN with various popular TLS ClientHello Messages |

The data does not show an indication of negotiated-ALPN-based discrimination.

### SNI
| ![alpn](/images/china-tls/sni.png) |
| :--: |
| Fig 3. various SNI/Hostnames |

The data does not show an indication of SNI or Hostname-based discrimination.

### IP address reputations
Also, according to the data we showed in Fig 1, the attack strength varies across different TLS servers, which might indicate that this attack is based on a hidden IP reputation system. The reputation rating factor is yet known.

## Conclusions
Our research shows:

- GFW does not demonstrate an ability to identify TLS proxy traffics out of normal HTTPS traffics in real-time.
- GFW isn’t utilizing a discrimination based on ClientHello/ALPN/SNI used for TLS handshake in real-time.
- Statistics shows a strong indication towards a hidden IP reputation system. 
- TLS Proxy clients who does not retry on receiving _RST_ is vulnerable to this kind of random _RST_ injection attack. 

## Credits
My advisor, Professor [Eric Wustrow](https://www.colorado.edu/ecee/eric-wustrow) at University of Colorado Boulder

Professor [J. Alex Halderman](https://eecs.engin.umich.edu/people/halderman-j-alex/) at University of Michigan

[Jack Wampler](https://scholar.google.com/citations?user=iHnZESwAAAAJ&hl=en) at University of Colorado Boulder

And everyone else from [Refraction Networking](https://refraction.network/) who withstood my spamming

Software Tool used during this project:
- [refraction-networking/utls](https://github.com/refraction-networking/utls)
- [tlsfingerprint.io](https://tlsfingerprint.io/)
- [tcpdump](https://www.tcpdump.org/)

## Appendix

### Artifact
the source code of the attack verification tool is published on GitHub: https://github.com/Gaukas/GFW-2021Summer-TLS-RST-Incident 
- Due to this attack is now inactive, you may not be able to get observation supporting my data.

### Incident timeline
Updated 7/28/2021 16:00 UTC:

The attack is no longer observable on any server for several days. We are now concluding the case and moving on.

Updated 7/23/2021 16:30 UTC:

The attack is again observable ONLY on one server I have control over. Besides, a minor timeout problem appears on all targets.

And no sign of server-side ALPN discrimination.


Updated 7/23/2021 00:33 UTC:

The attack is still inactive.

Internet discussion indicates this attack is:

- Limited to AS 4134 CHINANET-BACKBONE (in contradictory to CN2 AS4809).
- Bidirectional (the client, no matter a Chinese IP or a foreign IP, receives RST)

The ASN relation matches our data, where our only good IP is also the only server we have enforcing AS4809 routing.

We did not get a chance to verify the statement of Bidirectional behavior.

Updated 7/22/2021 18:36 UTC:
The attack is no longer active. I will keep an eye on it for the next 48 hours.

Updated 7/22/2021 00:00 UTC:
Thanks to Prof. Halderman and Jack, I checked the SNI. Sadly no strong evidence for SNI sniffing.

Updated 7/21/2021 00:00 UTC:
I have published the experimenting tool I used to check/confirm the attack on GitHub:

https://github.com/Gaukas/GFW-2021Summer-TLS-Proxy-Attack


Latency (ping) in ms from test node to all targets, in the same order:
65, 69, 93, 191, 63, 199, 131, 133

Packet loss, in the same order:
8%, 12%, 2%, 0%, 0%, 0%, 0%, 0%

According to the data we collected today, there is no sign of a strong correlation between TLS Fingerprints (from ClientHello) and TLS handshake successful rates. Instead, there is a strong sign of a hidden IP reputation system within the GFW. By utilizing the Reputation System, GFW could selectively attack different servers with different intensities.

Original Post 7/20/2021 00:00 UTC:
Our first guess was TLS Fingerprint-based attack. Thanks to the project TLSFingerprint.io, we exploited that different TLS clients may initialize the TLS connection with noticeable differences which is enough to categorize the users. If a TLS proxy client uses some unpopular fingerprints, the request made by such client would be very easy to be separated from normal browser-based HTTPS requests.

By collecting enough fingerprints from variant TLS proxy client implementations, we found that most of the popular TLS proxy clients use minor TLS fingerprints which is vulnerable to the attack we imagined. The only exception is the XTLS/Xray-core, which in their release v1.4.2, they have noticed and fixed the TLS fingerprinting vulnerability (big thanks to uTLS) by forging the major browsers’ fingerprints. However, even with Xray-core in use, the TLS proxies are still unavailable. But so far at least we can say GFW isn’t utilizing the fingerprint-based attack in real-time.

After I set up a test node in China in order to do some PCAP, I see strange things happening:

Google Chrome also had a difficult time connecting to the decoy sites. It keeps changing the fingerprints used for the handshake and the first time I try to connect, 3 different fingerprints with 6 ClientHello messages were involved.
cURL requests to decoy sites are suffering from SSL handshake failure due to connection reset.
TLS proxy requests are not guaranteed to fail. Still can see some successful requests.
This has been interesting. By expanding the experiments, I found that more than half of the TLS handshake to the proxy server under attack would experience _RST_ before the handshake completes. As a rough conclusion: GFW does not show the ability to identify TLS proxy traffics from normal HTTPS traffics in real-time.