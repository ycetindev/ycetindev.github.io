---
layout: post
title:  "Supercharging ARM Debug Output: ITM"
date:   2025-02-02 13:30:00 +0000
categories: embedded-systems debugging ARM ITM STM32 
---

We often rely on UART for serial output when it comes to debugging in embedded systems. While it's probably the most common method which gets the job done, it's not the fastest method available. Particularly if you are working on recent microcontrollers that support ITM (Instrumentation Trace Macrocell). ITM offers faster debugging and better performance monitoring, and in this post, I'll compare ITM with the traditional UART debugging method, which I covered in my [previous post](https://ycetindev.github.io/posts/2024-11-01-Redirecting-printf-to-UART.html).

## UART: A Classic Debugging Companion

As I said, this has always been our first choice; it has been around for ages for a good reason: it's quite simple and easy to implement. Since almost every microcontroller supports UART through the use of an onboard serial port, the output can be captured with any serial terminal. All you need is a USB-to-UART converter, and you're good to go with this simple and low-cost solution. On the other hand, it's mostly slower than new methods, and also brings additional processing time. 

## ITM: A Faster Alternative to UART

On modern ARM-based systems (like STM32), ITM is an advanced debugging feature that works in real-time to send output directly from the MCU to the debugger, which allows us to bypass the limitations (transfer rate or latency) of UART. As it uses one-wire interface with SWO (Serial Wire Output), it's a decent solution for performance profiling or real-time debugging. One major drawback of ITM is that it cannot handle user commands like UART does, so it's less interactive. However, it's still my first choice when debugging time-critical software. Even though we limit ourselves sometime due to space constraints in the latest design of the PCBs (limiting ourselves too much that we don't want to add an extra pin - SWO) - it's still quite good choice having this in early prototypes; where we spend most of our debugging time. 

## Performance Comparison: ITM vs UART

To compare the performance of ITM and UART, I set up a test using DWT Cycle Counter to measure the elapsed time for sending the lyrics of Bohemian Rhapsody from memory via both output methods.

I also enabled DWT to measure the number of clock cycles elapsed during the transmission. You can find details regarding DWT in my [previous post](https://ycetindev.github.io/posts/2024-10-30-code-execution-time-on-arm-cortex-m-mcus.html).

```c
/* Retargets the C library printf function */
PUTCHAR_PROTOTYPE
{
#ifdef ITM_ON
    ITM_SendChar(ch);
#else
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, 0xFFFF);
#endif
    return ch;
}
```

In the while loop, I sent the buffer (~300 bytes) to explicitly compare performance.

```c
while (1) 
{
    count_start = DWT->CYCCNT;

    printf("%s", buffer);  // Send data via ITM or UART

    count_stop = DWT->CYCCNT;
    count_elapsed = count_stop - count_start;

    printf("\r\nSize: %d, Elapsed count: %lu, Elapsed time: %.2f ms.\r\n", sizeof(buffer)-1, count_elapsed, count_elapsed/170000.0f);

    HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
    HAL_Delay(2000);
}
```

Here are the results: while my ST G4 was running at 170MHz, I configured the SWO clock to 12MHz, which is a sweet spot for the onboard ST-LINK V2.1 debugger on nucleo boards. This provides reliable operation while still delivering significantly better performance than UART. My UART setup used the most common baud rate—115200 bps.

![SWV ITM Data Console](/assets/SWV.png)

Sending the same buffer using ITM took 97470 clock cycles, which is equivalent to 0.57ms in my setup, while sending it over UART took 626930 clock cycles, or 3.69ms. 

![UART Serial Terminal](/assets/SerialTerminal.png)

## Conclusion

The performance difference between ITM and UART is significant—my tests show ITM is approximately 6.5 times faster than UART running at 115200 baud. This significant improvement, combined with minimal CPU overhead, makes ITM an excellent choice for real-time debugging and performance monitoring.

However, choosing between ITM and UART isn't always about raw speed. UART remains valuable when you need interactive debugging with user input, while ITM shines in scenarios requiring high-speed debug output with minimal overhead. For the best of both worlds, you can even use them together in your development setup.

One practical tip: if PCB space allows, always keep the SWO pin available in your prototypes. These enhanced debugging capabilities can save you countless hours during development and troubleshooting.

The complete source code for both methods is available in this [repository](https://github.com/ycetindev/stm32g4/tree/main/G431_ITMvsUART).