---
layout: post
title: "System identification simplified: take the red pill - trust the data"
date: 2025-02-22 10:30:00 +0000
categories: control-systems, system-identification
---

System identification is a fundamental concept in control engineering that bridges the gap between theoretical models and real-world applications. Not only in engineering but also from practical life experiences, I've learned that effective control of any system requires a deep understanding of its behavior. In this post, I'll demonstrate a practical approach to system identification using a motor emulator I developed in my previous posts.

{%- include MathJax.html -%}
## Why, Mr. Anderson? Why? Why do you persist?

Because, Agent Smith, in industrial applications, understanding system behavior is essential to design robust controllers, or to predict system behavior and optimise control system performance. So, system identification is your red pill to reality.

While black-box identification methods are useful when system knowledge is limited, having basic system insights – even rough ones – can significantly improve our approach. Let's revisit our DC motor model from the previous post.

In the electrical domain, our DC motor is governed by:

$$ L \frac{di}{dt} = -Ri + V - e \rightarrow \frac{di}{dt} = \frac{1}{L}(-Ri + V - K\omega_e) $$

Under steady-state conditions and assuming negligible electrical speed:

$$ V = L\frac{di}{dt} + Ri $$

Applying Laplace transform gives us:

$$ V(s) = I(s)(Ls + R) $$

Leading to our transfer function:

$$ \frac{I(s)}{V(s)} = \frac{1}{Ls + R} = \frac{1/R}{(L/R)s + 1} $$

Where:
- $$1/R$$ is our feedforward gain
- $$L/R$$ is our time constant

### Experimental Setup and Results

I conducted step response tests using our emulator's DAC output, measuring current through a scope. The measurements were scaled using the factor [(4095/3.3)/250] to obtain actual current values.

![Step Response Test](/assets/stepResponseTest.png)
*Step Response Test with 50% Duty Cycle*

From this initial test, we observed:
- Steady-state value: 5.27A
- Time to reach 95% (≈5.0A): 1.442s

Using these values:
- $$6 * 1/R = 5.27A \rightarrow R = 1.139$$ Ω
- $$T_s = 4\tau = 1.442s \rightarrow \tau = 0.3605$$s
- $$L = R * \tau = 1.139 * 0.3605 = 0.4106$$ H

To validate our findings and account for system nonlinearities, I repeated the tests at different input levels.

![Step Response Test (Multiple Duty Cycles)](/assets/stepResponseAll.png)
*Step Response Tests at 25%, 50%, 75%, and 100% Duty Cycles*

The results across different operating points:

| Duty Cycle | Resistance (Ω) | Inductance (H) |
|------------|---------------|----------------|
| 25%        | 1.178         | 0.415         |
| 50%        | 1.139         | 0.411         |
| 75%        | 1.124         | 0.414         |
| 100%       | 1.189         | 0.218         |

Average values: R = 1.1575 Ω, L = 0.3645 H

## Conclusion

While this simplest approach worked (our estimated parameters were quite close to the actual values), there's a beautiful lesson here: sometimes you just need to trust your data and keep things simple. The system will tell you its story - you just need to know how to listen.

Remember:
- More data = more confidence
- Simple methods can yield powerful results
- Real systems aren't perfect, and that's okay

*and, you've taken the red pill - there's no going back.*

