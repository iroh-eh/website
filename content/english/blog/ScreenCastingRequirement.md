---
title: "Comercial Linux Screen Casting"
meta_title: "Linux Screen Casting for Windows 10"
description: "Roadmap to Linux Screen Casting"
date: 2023-11-22T05:00:00Z
image: "/images/blog/screnshare.jpg"
categories: ["Linux Development"]
author: "Anurag Pandey"
tags: ["problem-solving", "miracast", "windows", "linux"]
draft: false
---

<br>

### Problem Statement
There is a debian based Operating System (called VingOS) runnning on Intel NUC. Any Apple Device could screen cast onto VingOS seamlessly, Ving Team wanted that we enable  the support for Windows operating devices too. This being a comercial product had to be exteremly user friendly and integrate with existing applications of VingOS seamlessly. Following were the features Ving was looking for : 
- Connection (Time taken for screen to appear on sink after hitting connect button in Windows) time should < 4 secs.
- Disconnection shoule be instant. 
- Should be able to stream Video (at 1080p) and Audio on large screens. No jitter and lag less that 100ms.
- Support Screen Extention (ability to use VingOS as other screen rather than casting).
- Support Multiple Screens being indiferrent to Aiplay or Miracast. 
- Gracefull disconnect in all the cases. 
- Should be able to connect to internet using wifi and ethernet while screen casting.


### Screencasting Interface in Windows ?

Windows has an out of box support for casting screen. It implements [Miracast](https://www.wi-fi.org/discover-wi-fi/miracast) (Open Specification). Great! So we need to have Miracast complaint sink (Sink is where screen is casted, Source device whose screen is being casted). 

There are two opensource repository (with suitable license) which allows Linux to namely : 
- [Miraclecast](https://github.com/albfan/miraclecast) -- Written completely in C, more rigorous and well written of two.
- [Lazycast](https://github.com/homeworkc/lazycast) -- Written in Python and C, easier to change and understand but made for Raspbian.

Windows user to screen cast has to do following : 
- Press (Win + k)
- Select the screen with cursor or arrow keys
- Click on "Connect" button

After a successful connection, Windows 11 shows option for following modes : 
- Duplicate (Default)
- Extend
- Second Screen only


Windows as per Miracast Specification supports complete Wifid connection. Additionally also supports [MS-MICE](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-mice/9598ca72-d937-466c-95f6-70401bb10bdb) (Over Infrastructure). User can't see any difference between two as the interface to connect and after connection remains the same. Miraclecast had no support for MICE but Lazycast has. To understand what Miraclecast and Lazycast had to offer lets explore Linux components which are important. 


### How does screencasting work ft. Miracast Specification

Screencasting can be broken down into following stages : 
- **Establishing connection** : Includes discovery of Linux device on windows Miracast Interface (done through P2P protocol) followed by connection between devices (either P2P or MS-MICE). After this phase is over two devie have a duplex socket connection therefore can exchange the data. An [RTSP Session](https://en.wikipedia.org/wiki/Real_Time_Streaming_Protocol) is started.
- **RTSP Session (Capability Negotiation)**: Source and Sink tell each other their capablitites e.g Bitrate, Resolution, Video and Audio Codecs, Exptected Latency etc. Full list  can be found in Miracast specification. After this phase is over, either sesion is terminated because of incompatiblity or [RTP Streaming](https://en.wikipedia.org/wiki/Real-time_Transport_Protocol) has started on agreed (7236 by default) port.
- **RTP Session** : This implies Windows has started streaming MPEG2 packets. Many media players support capturing and displaying RTP stream over a port e.g VLC. Adjustment to improve the decoding latency, pixelation, network buffering has to be done on Media Player level. Gstreamer is a popular choice (either write piplines using gst-launch or write you own code to custom build the player).

Only first stage varies based on whether we employ P2P or MS-MICE. We will delve deeper into first stage in another post.

To offer multiple screen casting session concurrently we should be able eshtablish multiple connections without interfering with the Airplay module. The connection we are interested has to happen over WiFi (otherwise it would not be user friendly). Most modern WiFi cards come with support for P2P connection -- allowing device to connect with each other over WiFi Securely without the need of router, that is in sigle hop. 

To findout the support from the wificard check the output of `iw phy` (linux terminal). Following is one excerpt of output from my terminal :

```python
        Supported interface modes:
                 * IBSS
                 * managed
                 * AP
                 * AP/VLAN
                 * monitor
                 * P2P-client
                 * P2P-GO
                 * P2P-device

```
Above tells us the supported modes. Normally when we are connected to WiFi network we are in `managed` mode. What we would require for MS-MICE of WiFid is P2P-client or P2P-GO.  But can wifi cards be in two modes simultaneously ? Yes in most cases. Following excerpt from `iw phy` shows which (and how many) modes are supported together.

```python
        valid interface combinations:
                 * #{ managed } <= 1, #{ AP, P2P-client, P2P-GO } <= 1, #{ P2P-device } <= 1,
                   total <= 3, #channels <= 2

```
Above tells us that we can be in `managed` mode and `(P2P-client or P2-GO)` and `P2P device` at the same time. But how the hell exactly ? That happnes using `Network Namespace`. 

But wait, what if Airplay intereferes with our work ? How do we make sure it is unaffected and can work in parallel ? How can we connect multiple Windows devices ? And P2P what ? We will cover all of it in our next blog.
