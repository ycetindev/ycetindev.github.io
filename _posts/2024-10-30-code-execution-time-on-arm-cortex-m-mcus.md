---
layout: post
title:  "Code Execution Time on Arm Cortex-M MCUs"
date:   2024-10-30 13:30:00 +0000
categories: embedded stm32 arm cortex-m
---

Ever tried to measure the execution time of your code block? This is a very common task, especially when working on time-critical applications that need to be computed at a high frequency, such as 100kHz. In such cases, you need to complete various operations like converting an ADC register, conditioning the input, and performing necessary control actions or protections - all within a tight time budget. 

While there are several ways to measure execution time, the most common approach is to toggle an output pin and use an oscilloscope to capture the waveform. However, this method requires an oscilloscope channel and additional setup, so there should be an easier way to do this: Debug Watchpoint and Trace (DWT) module! Thanks to the CoreSight debug port of Arm Cortex-M processors, there's always a 32-bit counter that's freely running to count clock cycles. This counter, known as DWT_CYCCNT, can be used to measure the execution time of your code.

![https://developer.arm.com/Processors/Cortex-M4](/assets/cortex-m4.png)

As an example, I'm going to demonstrate the usage of the DWT module using the STM32F303 microcontroller, which has Cortex-M4 core (based on the Armv7-M architecture). However, the concepts can be easily adapted to any Arm Cortex-M core.

## Using the DWT Module

Here are the main DWT registers we'll be using: 
(*You can have a look to **Armv7-M Architecture Reference Manual** for further details*)

![DWT register summary](/assets/DWT-register-summary.png)

Let's start by defining the register addresses to utilise the DWT module.

```c
volatile uint32_t *DWT_CTRL = (uint32_t *)(0xE0001000); 
volatile uint32_t *DWT_CYCCNT  = (uint32_t *)(0xE0001004);
```

![DWT_CTRL register](/assets/DWT-CTRL-register.png)

To enable the CYCCNT counter, we need to set the CYCCNTENA bit in the DWT_CTRL register:

```c
// Enable CYCCNTENA
*DWT_CTRL |= 1; 
```

However, there's one more control register we need to enable - the Debug Exception and Monitor Control Register (DEMCR). Bit 24 of this register, TRCENA, needs to be set to enable the DWT/ITM features.

```c
volatile uint32_t *DEMCR = (uint32_t *)(0xE000EDFC);

// and then Enable bit[24] TRCENA
*DEMCR |= (1 << 24); 
```

![DEMCR register](/assets/DEMCR-register.png)

Now that we've enabled the CYCCNT counter, we can use it to measure the execution time of our code:

```c
count_start = *DWT_CYCCNT;
// code to measure the execution time
count_stop = *DWT_CYCCNT;
count_elapsed = count_stop - count_start;
```

The simplicity of this method lies in the fact that we're directly reading the free-running CYCCNT counter before and after the code we want to measure. The difference between the start and end values gives us the elapsed clock cycles.

However, there's one thing to keep in mind: interrupts can affect the measured time. Whenever an interrupt condition is triggered, the relevant Interrupt Service Routine (ISR) will run, and the time spent in the ISR will be included in the execution time measurement. To prevent this, you may want to disable all interrupts before taking the measurement.
Alternatively, if you want to capture the minimum and maximum execution times based on different conditions in the code, you can add some additional logic:

```c
uint32_t count_max = 0xFFFFFFFF;
uint32_t count_min = 0x00;

    if(count_elapsed > count_max){
    count_max = count elapsed;
    }
    if(count_elapsed < min){
    count_min = count_elapsed;
    }
```

As a demonstration, I've added a simple LED toggling and button capture function to show the difference in execution time:

![code-exec-time-quick-demo](/assets/code-exec-time-quick-demo-1.gif)

In this example, you can see how the elapsed clock cycle count varies based on the conditional statements in the code.
To calculate the actual time elapsed, you need to know the CPU clock speed. For example, if the clock speed is 72 MHz, then 30 clock cycles would equate to 0.416 microseconds.

```c
// Calculate time in microseconds (assuming 72 MHz clock)
float time_us = (float)count_elapsed / 72.0f;
```

The DWT-based approach provides a straightforward and accurate way to profile the execution time of your Arm Cortex-M embedded code. By leveraging the built-in DWT module, you can easily measure and optimize the performance of your time-critical applications.