---
title: "Unit Tests for Embedded"
date: 2020-04-08T18:05:30+08:00
draft: false
---

Been a while wondering if embedded program can be unit-tested...  

One day, I was about to rewrite IR_Manager, which was actually a driver that managed IR data transmission and reception. It supported various main stream IR protocols and few carrier frequencies. It can dynamically re-mount the desire IR protocol on runtime such as IR remote re-pair or differenct level of device reset. 

It then re-initialise the required MCU peripherals such as Clock, GPIOs, Timers and interrupts. 

Since it didn`t have a state control (states definitions found but not implemented), different type of IR process were putted in a single huge loop, making it not easy to read and maintain or test. 

# Transmission:  
When the clouds emit a transmit IR command to the hardware, it transmits the stream if it passed all the checks.

# Receiving:  
1. When IR stream is captured, IR_Manager will decode it if it matches any of initialised drivers.  
2. Once the stream successfully went through some filters and some conditions, it will be feeded into an encoder to generate commpressed notation of the stream, and send to the cloud.

# The Plan  
When I was studying this, a quiestion came to mind: How should a embedded engineer test all the edge cases among all these protocols and checks?  

By abstracting the peripherals drivers from embedded software would be a way, but how? I found Ceedling, it uses GCC instead of the target compiler. We can use it to mock the interfaces so we don't need to repeat the classic loop: flash, run, and debug on the target hardware.

This allows us to run our unit-tests rapidly without conducting the hardware might be even available.  

In this scenario, expected peripherals need to be mocked would be: one timer for IR transmit and one timer for IR receive. However, Ceedling provide a improved experience for automatically generating the CMocks we need.

# The result  
So, basically I finished IR rewriting by kind of this pattern, let sat for IR decoding:  

1. find out all the possible success/failure path in decoding an IR stream  
2. group and write test cases by their purpose  
3. rewrite it, debug it by only Ceedling rather than flashing the program to the hardwear  

# The tricky part you can't mock
The IR transmiting carrier ticker and the IR receiving, they relies on their corresponding timer interrupt.  

So when the development is totally off-the-hardware, how would the hardware interrupt vectors work?

Ceedling provides some form of mock functions to allow you manually control the 'expected', 'unexpected', 'ignore' and 'return' of functions.
Example of IR receive a pair of mark/space data on stm32f1x mcu would be completed by below sequence:  
1. setup the read-out of input capture 1 = mark value
2. setup the read-out of input capture 2 = (mark+space) value
3. call the interrupt function manually

test_IR.c
```
1       // cond1 & cond2 are the arbitrary generated start conditions
2       TIM_GetITStatus_ExpectAndReturn(IR_RX_TIM, TIM_IT_CC1, RESET);
3       TIM_GetITStatus_ExpectAndReturn(IR_RX_TIM, TIM_IT_CC2, !RESET);
4       TIM_ClearITPendingBit_Expect(IR_RX_TIM, TIM_IT_CC2);
5       TIM_GetCapture1_ExpectAndReturn(IR_RX_TIM, cond1);
6       TIM_GetCapture2_ExpectAndReturn(IR_RX_TIM, cond1 + cond2);
7       IR_RX_TIM_IRQHandler();
```
IR.c
```
1       void IR_RX_TIM_IRQHandler(void)
2       {
3               /* Get the Input Capture value */
4               if (TIM_GetITStatus(IR_RX_TIM, TIM_IT_CC1)) {
5                       TIM_ClearITPendingBit(IR_RX_TIM, TIM_IT_CC1);
6                       capture_duty = TIM_GetCapture2(IR_RX_TIM) << IR_CLOCK_DIV;
7                       capture_period = TIM_GetCapture1(IR_RX_TIM) << IR_CLOCK_DIV;
8                       
9               }
10              if (TIM_GetITStatus(IR_RX_TIM, TIM_IT_CC2)) {
11                      TIM_ClearITPendingBit(IR_RX_TIM, TIM_IT_CC2);
12                      capture_duty = TIM_GetCapture1(IR_RX_TIM) << IR_CLOCK_DIV;
13                      capture_period = TIM_GetCapture2(IR_RX_TIM) << IR_CLOCK_DIV;
14              }
15      
16              switch (ir_state) {
17              case IR_STATE_RX:
18                      carrier_ticker = carrier_rx_timeout_tick;
19                      break;
20              case IR_STATE_IDLE:
21                      if (capture_start > 0) {
22                              break;
23                      }
24                      ... // some start condition checks
25                      capture_start = 1;
26                      capture_stream_ptr = 0;
27                      ir_state = IR_STATE_RX;
28              ...
```

# Thoughts  
Unit-test is a good way of embedded development. Ceedling is also a good tool even for embedded developer, it helps us to reduce the waste of development time. By abstracting the peripheral layer, developer can purely focus on the program itself for the moment.  

# TBC...

![](/posts/images/unit_test1.png)