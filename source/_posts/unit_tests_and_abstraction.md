---
title: "Unit Tests & Abstraction"
date: 2020-04-08T18:05:30+08:00
showDate: true
draft: false
---

Been a while wondering if embedded program can be unit-tested...  

One day, I was about to rewrite IR_Manager, which was actually a driver that managed IR data transmission and reception. It supported various main stream IR protocols and few carrier frequencies. It can dynamically re-mount the desire IR protocol on runtime such as IR remote re-pair or differenct level of device reset. 

It then re-initialise the required MCU peripherals such as Clock, GPIOs, Timers and interrupts. 

Since it didn`t have a state control (states definitions found but not implemented), different type of IR process were putted in a single huge loop, making it not easy to read and maintain or test. 

![](logic.png)
## Transmission
When the clouds emit a transmit IR command to the hardware, it transmits the stream is valid.

## Receiving
1. When IR stream is captured, IR_Manager will decode it if it matches any protocols of initialised drivers.  
2. Once the stream successfully went through filters and checks, it will be feeded into an encoder to generate commpressed notation of the stream, and send to the cloud.

## The Plan
![alt](unit_test.png)


When I was studying this, a quiestion came to mind: How should a embedded engineer test all the edge cases among all these protocols and checks?  

By abstracting the peripherals drivers from embedded software would be a way, but how? I found Ceedling, it uses GCC instead of the target compiler. We can use it to mock the interfaces so we don't need to repeat the classic loop: flash, run, and debug on the target hardware.

This allows us to run our unit-tests rapidly without conducting the hardware might be even available.  

In this scenario, expected peripherals need to be mocked would be: one timer for IR transmit and one timer for IR receive. However, Ceedling provide a improved experience for automatically generating the CMocks we need.

## The result  
So, basically I finished IR rewriting by this pattern, let say for IR decoding:  

1. find out all the possible success/failure paths in decoding an IR stream  
2. group and write test cases by their purpose  
3. rewrite it, debug it by only Ceedling rather than flashing the program to the hardwear  

## The tricky part you can't mock
The IR transmiting carrier ticker and the IR receiving, they relies on their corresponding timer interrupt.  

So when the development is totally off-the-hardware, how would the hardware interrupt vectors work?

Ceedling provides some form of mock functions to allow you manually control the 'expected', 'unexpected', 'ignore' and 'return' of functions.
Example of IR receive a pair of mark/space data on stm32f1x mcu would be completed by below sequence:  
1. setup the read-out of input capture 1 equal to mark value
2. setup the read-out of input capture 2 equal to (mark+space) value
3. call the interrupt function manually

test_IR.c
```
        // cond1 & cond2 are the arbitrary generated start conditions
        TIM_GetITStatus_ExpectAndReturn(IR_RX_TIM, TIM_IT_CC1, RESET);
        TIM_GetITStatus_ExpectAndReturn(IR_RX_TIM, TIM_IT_CC2, !RESET);
        TIM_ClearITPendingBit_Expect(IR_RX_TIM, TIM_IT_CC2);
        TIM_GetCapture1_ExpectAndReturn(IR_RX_TIM, cond1);
        TIM_GetCapture2_ExpectAndReturn(IR_RX_TIM, cond1 + cond2);
        IR_RX_TIM_IRQHandler();
```
IR.c
```
void IR_RX_TIM_IRQHandler(void)
{
        /* Get the Input Capture value */
        if (TIM_GetITStatus(IR_RX_TIM, TIM_IT_CC1)) {
                TIM_ClearITPendingBit(IR_RX_TIM, TIM_IT_CC1);
                capture_duty = TIM_GetCapture2(IR_RX_TIM) << IR_CLOCK_DIV;
                capture_period = TIM_GetCapture1(IR_RX_TIM) << IR_CLOCK_DIV;
                
        }
        if (TIM_GetITStatus(IR_RX_TIM, TIM_IT_CC2)) {
                TIM_ClearITPendingBit(IR_RX_TIM, TIM_IT_CC2);
                capture_duty = TIM_GetCapture1(IR_RX_TIM) << IR_CLOCK_DIV;
                capture_period = TIM_GetCapture2(IR_RX_TIM) << IR_CLOCK_DIV;
        }

        switch (ir_state) {
        case IR_STATE_RX:
                carrier_ticker = carrier_rx_timeout_tick;
                break;
        case IR_STATE_IDLE:
                if (capture_start > 0) {
                        break;
                }
                ... // some start condition checks
                capture_start = 1;
                capture_stream_ptr = 0;
                ir_state = IR_STATE_RX;
...
```

## Thoughts
TDD is a good way for software development. Ceedling is also a good tool even for embedded developer, it helps us to reduce the waste of development time. By abstracting the application layer, developer can purely focus on the program itself for the moment.  

## Abstraction

To abstract the application level is the ultimate goal. From experiences, the includes of header files increase from time to time as the application grow. The more connections between source files, the harder we can easily change or test them in module by module.

Imagine a situation: **Build a menu module with serial LCD driver:**

Normally we would start from the bottom: 
initialise serial interface peripherals -> follow the datasheet -> make reasonable prototypes for the future

After we have the driver, we would continue building menu and animations. Sounds straight forward right?

Picture it, at the end of the menu system, at the very top of menu.c we would have something like:
```
#include "global.h"
#include "interrupts.h"
#include "GPIO.h"
#include "menuGraphics.h"       // animations
#include "ILI9341_driver.h"     // SPI driver
```

To make the application level abstracted, at some point, we might need to think: which calls which?
In this case, the menu should own the driver object, menu can initiliase the driver by passing the object in for example. Whenever the menu needs driver to execute or calculate, it calls the prototype, the prototype checks the if object exists.

The driver is as simple as a container of the SPI chip's **logics** that only use the flash memory. When the memory usages are bounded and keep as much as application level, developer can control memory usage easiler.