---
title: "Unit Tests for Embedded"
date: 2020-05-15T18:05:30+08:00
draft: false
---

Been a while wondering if embedded program can be unit-tested...  

I was about to rewrite IR_Manager. It was actually a set of 'driver' that to handle all IR transmit and receive among all the IR protocols or as raw as plain IR profiles.  

The basic logic is: when the device on boot, the firmware picks and initialises the protocols the hardware is needed.  

# Transmission:  
When the clouds emit a transmit IR command to the hardware, it transmits the stream if it passed some restrictions, basically simple as that.  

# Receiving:  
1. When IR stream is captured, IR_Manager will decode it if it matches any of initialised drivers.  
2. Once the stream successfully went through some filters and some conditions, it will be feeded into a encoder to generate a IR commpressed notation and send to the cloud.    

When I was studying this, a quiestion came to mind: How should a embedded engineer test all the edge cases among all these drivers set?  

# The Plan  
To abstract the peripherals drivers from embedded software, the key is to mock.  
I found Ceedling. We can use it to mock the interfaces to our hardware so we don't need to 'flash, run, and debug' on the target hardware.  

This allows us to run our unit-tests rapidly without conducting the hardware might be even available.  

In this scenario, expected peripherals need to be mocked would be: one timer for IR transmit and one timer for IR receive. However, Ceedling provide a improved experience for automatically generating the CMocks we need.

# The result  
So, basically I finished IR rewriting by kind of this pattern, let sat for IR decoding:  

1. find out all the possible success/failure path in decoding an IR stream  
2. group and write test cases by their purpose  
3. rewrite it, debug it by only Ceedling rather than flashing the program to the hardwear  

# The funniest part, Interrupts  
The IR transmiting carrier ticker and the IR receiving, they relies on their corresponding timer interrupt.  

So when the development is totally off-the-hardware, how would the hardware interrupt vectors work, right?  

Ceedling provides some form of mock functions to allow you manually control the 'expected', 'unexpected', 'ignore' and 'return' of functions.  
Example of IR receive a pair of mark/space data on stm32f1x mcu would be completed by below sequence:  
1. setup the read-out of input capture 1 of timerX equal to the mark value
2. setup the read-out of input capture 2 of timerX equal to the mark+space  
3. call the interrupt function manually

# Thoughts  
Unit-test is a good way of embedded development. Ceedling is also a good tool even for embedded developer, it helps us to reduce the waste of development time. By abstracting the peripheral layer, developer can purely focus on the program itself for the moment.  

![](/posts/images/unit_test1.png)