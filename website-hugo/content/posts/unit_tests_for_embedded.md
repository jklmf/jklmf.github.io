---
title: "Unit Tests for Embedded"
date: 2020-05-15T18:05:30+08:00
draft: false
---

Been a while wondering if embedded program can be unit-tested...<br>
<br>
<br>
I was about to rewrite IR_Manager. It was actually a set of 'driver' that to handle all IR transmit and receive among all the IR protocols or as raw as plain IR profiles.
<br>
The basic logic is: when the device on boot, the firmware picks and initialises the protocols the hardware is needed.
# Transmission:

When the clouds emit a transmit IR command to the hardware, it transmits the stream if it passed some restrictions, basically simple as that.
# Receiving:
1. When IR stream is captured, IR_Manager will decode it if it matches any of initialised drivers.
2. Once the stream successfully went through some filters and some conditions, it will be feeded into a encoder to generate a IR commpressed notation and send to the cloud.
<br>
When I was studying these, a quiestion came to mind: How should a embedded engineer test all the edge cases among all these drivers set?
<br>
# CMock
tbc...

![](/posts/images/unit_test1.png)