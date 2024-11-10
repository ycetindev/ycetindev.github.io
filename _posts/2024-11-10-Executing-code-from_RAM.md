---
layout: post
title:  "Executing code from RAM on STM32"
date:   2024-11-10 13:00:00 +0000
categories: embedded stm32 arm cortex-m optimisation ram
---

In time critical algorithms, we might need to do some tasks or calculations quite fast. When dealing with them in resource limited embedded platforms, we are always looking into ways to optimise our algorithms. Placing some bits of the code to be executed into RAM is one of them and in this post we will dive into details of it, again I'll be using Clarke and Parke transformations to demonstrate its effectiveness which is very common in three phase motor control.

{%- include MathJax.html -%}
## Introduction

In motor control applications, we frequently need to perform coordinate transformations, such as the Clarke and Park transformations, to convert between the ABC and dq reference frames. These transformations involve calculating sine and cosine functions, which can be computationally intensive, particularly when the control loop runs at high frequencies (typically 10-100 kHz). For details, please check my [previous post](https://ycetindev.github.io/posts/2024-11-09-STM32G4-Cordic.html).
In that post, we have used G4's CORDIC coprocessor to accelerate our trigonometric calculations, but in F3 series we don't have any CORDIC or similar hardware. So, instead of  that, I'm gonna use standard math lib functions for the calculation of cosine and sine terms, however, I'll be placing Clarke and Park transformations in the RAM to speed up the calculations.

I'm just gonna copy here the simplified version which is ready to implement in C from previous post:

$$i_d = 0.6667[cos(\theta)i_a+(0.866sin(\theta)-0.5cos(\theta)i_b+(-0.866sin(\theta)-0.5cos(\theta))i_c] $$ 

$$i_q = 0.6667[-sin(\theta)i_a-(-0.866cos(\theta)-0.5sin(\theta))i_b-(0.866cos(\theta)-0.5sin(\theta))i_c] $$

Since fetching data from RAM is faster than from flash, we may want to place some time-critical code into the RAM which we want to execute a bit faster. Most of the time, we don't have enough RAM to place all code into that, so we only prefer to use that limited area for time-critical tasks. For example, I'm going to demonstrate it for the STM32F303VC microcontroller which has 48kB of RAM and 256kB of FLASH. 

To place a function into RAM, we just need to define it with the relevant section attribute to tell the linker to put it in the right place. Since .RamFunc has already been defined in the linker description file, nothing else we need to do. 

```c
#define __RAM_FUNC __attribute__((section(".RamFunc")))
```

For example, here's the relevant part from my ld file. The `>RAM AT> FLASH` command is also another key point because the linker copies the flash memory into RAM at startup during initialization which exists in these sections.

```c
  /* Initialized data sections into "RAM" Ram type memory */
  .data :
  {
    . = ALIGN(4);
    _sdata = .;        /* create a global symbol at data start */
    *(.data)           /* .data sections */
    *(.data*)          /* .data* sections */
    *(.RamFunc)        /* .RamFunc sections */
    *(.RamFunc*)       /* .RamFunc* sections */

    . = ALIGN(4);
    _edata = .;        /* define a global symbol at data end */

  } >RAM AT> FLASH
```

Next, I've used the same Clarke and Park transformations from my previous post again. Now we have two identical functions, one placed into RAM and the second one staying in Flash:

```c
__RAM_FUNC Idq_t ABC2DQ_fRAM(Iabc_t Iabc, trg_t trg){
  ...
}
```

```c
Idq_t ABC2DQ_fFLASH(Iabc_t Iabc, trg_t trg){
  ...
}
```

After doing this, the easiest way to check if your functions are placed properly is putting two breakpoints to each function and then having a look at your PC register.

The function placed in Flash has to be at Flash addressess, which is started from 0x08000000:
![ABC2DQ_fFLASH](/assets/ABC2DQ_fFLASH.png)

While the function executing from RAM has to be located to the RAM, which starts from 0x20000000: 
![ABC2DQ_fRAM](/assets/ABC2DQ_fRAM.png)

Again, we had a look to memory maps when creating the linker script in my [previous post about getting started with STM32](https://ycetindev.github.io/posts/2024-10-27-hello-stm32.html). 

## Performance Comparison

To evaluate the performance, I'm using the DWT cycle counter (covered in my [previous post about execution timing](https://ycetindev.github.io/posts/2024-10-30-code-execution-time-on-arm-cortex-m-mcus.html)) and UART printing (explained in my [post about printf redirection](https://ycetindev.github.io/posts/2024-11-01-Redirecting-printf-to-UART.html)), I measured the execution times simply as follows:

```c
ET_fFLASH.start = *ET_DWT_CYCCNT;
Idq_fFLASH = ABC2DQ_fFLASH(Iabc, trg);
ET_fFLASH.stop = *ET_DWT_CYCCNT;

ET_fRAM.start = *ET_DWT_CYCCNT;
Idq_fRAM = ABC2DQ_fRAM(Iabc, trg);
ET_fRAM.stop = *ET_DWT_CYCCNT;
```

The results look promising:

![Executing code from RAM vs Flash](/assets/FlashvsRAM.PNG)

*Comparison of execution times running from RAM vs Flash*

![abc2dq RAM vs Flash](/assets/abc2dq_RAMvsFlash.bmp)

*Transformed currents and execution times visualised through serial plotting*

And here's a real-time visualisation of the system in action:

![Real-time serial terminal](/assets/ExecFromRAM.gif)

*Real-time visualisation of execution time*

## Conclusion


As we've seen, executing code from RAM instead of Flash can provide a significant performance boost. This technique is particularly valuable for time-critical operations like motor control algorithms where every microsecond counts.

However, like everything in embedded systems, it's a trade-off. While RAM execution is faster, we're also using up our precious RAM space which is usually much more limited than Flash. On our STM32F303VC, we're trading some of our 48kB RAM for speed. It's like choosing between a small city apartment with everything at your fingertips versus a larger house in the suburbs - sure, the house has more space, but getting anywhere takes longer!
