---
layout: post
title: My First Arduino Library
date: 2018-04-25 12:00:00
tags: arduino
author: vminute
---

<amp-img src="{{ site.baseurl }}assets/images/blog/2018-04-25-screenshot.png" width="800" height="450" layout="responsive" alt="" class="mb3"></amp-img>

I’ve been playing with Arduino for quite a long time. I used the AVR-based UNO (was called “diecimila” or “duemilanove” at that time), but also some of the variants released over the years by Arduino itself or by other vendors that leveraged their open model to design their own hardware and integrate it with the IDE and ecosystem.

I used it for many hobby projects (most of them still lying "almost finished" on my desk) and to implement quick proofs of concepts and prototypes. I like the easy to use IDE (missing only an interactive debugger…) and how easy it is to quickly integrate many different kinds of sensors, actuators, and devices.

Lately, I used some ESP32 based devices and really appreciated how easy it is to use their connectivity features.

Connecting your device will also require that you care about security. For a project that I am still developing (and hope to document here soon) I needed to download information from different websites using HTTPs. 
This requires certificates and you need to provide a known root certificate to ensure that the one provided by the website you are trying to reach is valid. Having a valid “chain” of certificates will ensure that your connection is encrypted and that you are really connecting to the server you were referencing in your URL. 

If you always connect to the same server you will probably solve the issue by keeping its root certificate as a string in your source code. This may stop working if the certificate expires, is revoked or the owner of the server decides to use a different one for any reason and you may need to check the original certificate and eventually rebuild your sketch with a new one.

If you need to connect to different servers, using different certificates, things start to get a bit more complicated. You will need to store different root certificates and validate the server ones against them.

This is what your browser does when showing you the lock in the URL bar of a secure connection.

Browsers can store certificates on the local filesystem, inside a database or let the operating system manage them. On Arduino you don’t usually have those fancy features, so you need to store the certificates you need inside your sketch as mime-64 encoded strings. 

My small library attempts to make this process simpler by associating commonly used domains to their root certificates. 
Currently, it manages only certificates used to access Google services (this is what I am using in my project) but can be easily extended to support new ones.

You can find the code and documentation on <a href="https://github.com/VMinute/RootCertificates">GitHub</a>.

You can add it to your own sketches using the Arduino library manager, as shown in the screenshot above.

