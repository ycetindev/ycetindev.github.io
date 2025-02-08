---
layout: post
title:  "Building a real-time motor emulator"
date:   2025-02-08 13:30:00 +0000
categories: embedded-systems control-theory motor-control simulink stm32
---

Developing motor control algorithms directly on real hardware can be challenging due to hardware constraints and debugging complexity. In this series, I'll be building a motor emulator that enables to test and refine control algorithms before moving to physical implementation. In this post, I'll start with Simulink and then dive into the embedded systems. 

I'll start with one of the simplest problem - DC motor control using H-Bridge.

My plan is to model this using Simscape library in continuous domain first, then implement and compare it with mathematical modelling in Simulink. Next, I'll be implementing this into the microcontroller. Simscape offers a physical modeling approach that directly represents electrical and mechanical components, making it intuitive to simulate system dynamics. However, mathematical modeling in Simulink provides more control over the equations and enables direct implementation in embedded software. Comparing the two ensures that our mathematical model accurately represents real-world behavior.

## Simscape model

![Simscape model](/assets/DCMotor_SS1.png)

In this part, I'm just providing 10kHz PWM signal as an input with fixed duty cycle to provide 1V to the motor.

![PWM signal](/assets/SimulinkPWM.bmp)

## Mathematical model

![Mathematical model](/assets/DCMotor_MM1.png)

Here's the comparison result for Simscape model vs Mathematical model.

![Simulink SS vs MM comparison](/assets/SimulinkComparison.bmp)

They're exactly same! Since I've been running mathematical model quite fast (at 100kHz interval), the obtained results show mathematical model accurate enough.

## Embedded implementation

Now, I'll be implementing this into the microcontroller and I'll be using STM32F3 Discovery board for this. The aim is to make it kind of DC motor system emulator, which enables us to develop and test our controller later. Basics of the implementation:

- The differential equations are quite obvious, it'll be running at 100kHz update rate.
- There will be single input and two outputs of the system (as in the simulations above).
  - Input: Voltage (this will be an PWM input capture interface) as in the simulation (frequency: 10kHz)
  - Outputs: Current and speed (DAC outputs). To make them realistic as much as possible, emulator will be updating the outputs at the PWM frequency rate, 10kHz.

Since ST's F303 already has FPU, I've implemented the model with floating point variables quite straightforward:

```c
void update_motor_model(float applied_voltage)
{
    voltage = applied_voltage;

    // compute derivatives
    float di_dt = (voltage - R * current - Ke * omega) * INV_L;
    float domega_dt = (Kt * current - B * omega) * INV_J;

    // calculate next states using forward Euler approx.
    current = current + Ts * di_dt;
    omega = omega + Ts * domega_dt;

    // to plot
    dbg_current = current;
    dbg_omega = omega;
}
```

As I said, this function will be called at each 100kHz (10us). So, before configuring the additional peripherals, I wanted to check the implementation using SWV interface by logging the states. (You can find details regarding logging using SWV in my [previous post](https://ycetindev.github.io/posts/2025-02-02-Supercharging-ARM-Debug-Output-ITM.html).)

Below you can see obtained step response, note - even though the model runs at 100kHz, I logged the variables at 10kHz rate.

![TBD](/assets/DCMotor_F3Emulator_SWV.png)

As can be seen clearly, obtained responses perfectly match!

Now, as a last step, I'm gonna configure DAC channels to provide current and speed as outputs. We have 12-bit DAC with two channels in this MCU, so we can provide output quite precisely. For initial evaluation, I'm going to assume their maximum values can reach to "1", so I'll be scaling the state variables with 2^12-1 = 4095, and update the outputs at 10kHz interval. Lastly, for the initial test, I'll be connecting button as if "applied_voltage = 1", so when it rises, it means applied_voltage will be set to 1 in the software to check the output of the model.

Finally, here are the captured scope waveforms from DAC outputs. They're also perfectly match with the simulation results and DAC peripheral performed quite well for the sake of the emulator.

![TBD](/assets/DCMotor_F3DiscoEmulator_ScopeWaveform.png)

The complete source code, simulations, logs, scope waveforms and scripts is available in the [repository](https://github.com/ycetindev/stm32f3/tree/main/F3Disco_MotorEmulator).

That's all for this part. Next up, we'll be diving into control system implementation to regulate the motor speed.

Of course, watching numbers and waveforms can't match the satisfaction of hearing a real motor whirring to life. 
But being able to start developing and testing control algorithms without wrestling with cables, power supplies, or that one encoder that just won't behave... that's a pretty good feeling too! And if you disagree, well, just wait a bit more...