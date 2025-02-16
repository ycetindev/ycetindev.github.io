---
layout: post
title: "Taking the motor emulator for a spin: interface testing"
date: 2025-02-15 10:30:00 +0000
categories: embedded-systems motor-control emulator stm32 instrumentation testing
---

In my [previous post](https://ycetindev.github.io/posts/2025-02-08-Building-a-real-time-motor-emulator.html), I detailed the implementation of a real-time motor emulator on a microcontroller. Before diving into control system development, I thought it's a good idea to validate the emulator with real inputs and outputs first. This intermediate step will ensure our platform is ready for control algorithm implementation while demonstrating the power of proper instrumentation.

## System Enhancements

To make the emulator fully self-contained and testable, I've made several improvements:

### PWM Input Capture

Added TIM2 in input capture mode to measure the incoming 10kHz PWM signal's duty cycle. This allows our emulator to respond to external PWM commands just like a real motor driver would. Simple trick here: keeping PSC register at minimum (0) will allow us to spec ARR (or PRD) register at maximum - hence gives us better resolution when capturing the signal.  

![Emulator Simple Block Diagram](/assets/Emulator_BlockDiagram.PNG)

```c
// TIM2 ISR Callback
if((TIM2->SR) & (TIM_SR_CC1IF))
{
  TIM2->SR = ~(TIM_SR_CC1IF);

  if(IsFirstCaptured == 0 && (GPIOA->IDR & GPIO_PIN_15)) //rising edge
  {
	IsFirstCaptured = 1;
	ICVal1 = TIM2->CCR1;
  }
  else if(IsFirstCaptured) //falling edge
  {
	ICVal2 = TIM2->CCR1;
	if(ICVal2 > ICVal1)
	{
	  Difference = ICVal2 - ICVal1;
	}
	else
	{
	  Difference = (TIM2->ARR - ICVal1) + ICVal2;
	}
	IsFirstCaptured = 0;
  }
}
```

### Output scaling
After evaluating the maximum values of our state variables, I've implemented appropriate scaling factors for more realistic output representation:

```c
#define SCALE_CURRENT 250U
#define SCALE_OMEGA   1000U
```

These scaling factors are used to convert emulator outputs to real-world values before sending them to the DAC.

### Test results
Using a signal generator and oscilloscope, I tested the complete system by:

- Generating 10kHz PWM signals with various duty cycles
- Measuring the DAC outputs for current and speed
- Verifying the system response matches our Simulink simulations

![Emulator GIF](/assets/Emulator_Scope.gif)

As we can see, the emulator responds accurately to different PWM duty cycles. The current and speed outputs scale appropriately and match our theoretical expectations.

Within this testing phase, I've validated end-to-end emulator functionality while establishing clear input/output interfaces. Now, we're ready to implement control algorithms in the next post. 

Again, the complete source code is available in the [repository](https://github.com/ycetindev/stm32f3/tree/main/F3Disco_MotorEmulator).
