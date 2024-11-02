---
layout: post
title: "Redirecting printf to UART for STM32 Debugging"
date: 2024-11-01 21:30:00 +0000
categories: embedded stm32 arm cortex-m printf debugging UART comms
---

When developing embedded applications, having a reliable debugging method is crucial. One of the most convenient debugging techniques is redirecting the `printf` function to output through UART.

## Introduction

 In this post, I'll show you how to implement this on an STM32F303VC microcontroller using ST's HAL Library. While I generally prefer bare-metal programming, the HAL library can be useful for quick prototyping. 

Let's dive into the implementation!

## UART Configuration

For this demonstration, I'll use UART4 (though any of the five UART peripherals available on this MCU would work) with the following configuration:
- Baud rate: 115200 bits/s
- Data bits: 8
- Parity: None
- Stop bits: 1
- Pins: PC10 (TX) and PC11 (RX)

## Implementing printf Redirection

The key to redirecting `printf` is implementing the correct function prototype based on your toolchain. Here's how to do it:

```c
#ifdef __GNUC__
#define PUTCHAR_PROTOTYPE int __io_putchar(int ch)
#else
#define PUTCHAR_PROTOTYPE int fputc(int ch, FILE *f)
#endif
```

Next, implement the character transmission function using UART4:

```c
/* Retargets the C library printf function to the USART */
PUTCHAR_PROTOTYPE
{
    // Write a single character to UART4 and wait until transmission is complete
    HAL_UART_Transmit(&huart4, (uint8_t *)&ch, 1, 0xFFFF);
    return ch;
}
```

That's it! 

After implementing these functions, you can use `printf` as you normally would:

```c
printf("still alive\r\n");
```

## Practical Example: Temperature Monitoring

To demonstrate the usefulness of this setup, let's create a simple application that reads the MCU's internal temperature sensor every second and prints the readings over UART:

```c
while (1)
{
    if(flag_1s != 0)
    {
        flag_1s = 0;
        printf("still alive\r\n");

        // Calculate temperature using internal sensor
        mcu_temp = (((float)(ADC1->DR - ADC1_CAL) - TS_CAL1) / 
                    (TS_CAL2 - TS_CAL1)) * (110.0F - 30.0F) + 30.0F;
        printf("Temperature: %.2f°C\r\n", mcu_temp);
    }
}
```

## Visualizing the Data

You can view the output using any serial terminal application. Here are some visualization options:

Basic serial terminal output:

![Temperature readings in terminal](/assets/stillAlive.png)

Real-time plotting using CoolTerm:

![Real-time temperature plot](/assets/coolTerm.png)

Real-time plotting using your own script in MATLAB, which you can tailor based on your needs:

![Real-time temperature plot using MATLAB](/assets/serialportplot.gif)

The complete source code for this project is available at [this link](https://github.com/ycetindev/stm32f3/tree/82fd68d41ff994436eed9804e2ed46c8c9bffcd6/).

## Conclusion

Redirecting `printf` to UART is a powerful debugging technique that allows you to easily monitor variables and system state during development. Combined with serial plotting tools, it becomes an invaluable tool for visualizing sensor data and system behavior in real-time.

Debugging prints are like breadcrumbs in a forest - the more you leave, the easier it is to find your way back. At least now we can make our MCU talk to us through UART, even if it's just to say "still alive" - which, let's be honest, is sometimes all we need to know after a long debugging session!