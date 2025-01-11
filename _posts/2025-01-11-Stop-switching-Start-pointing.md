---
layout: post
title:  "Stop switching, start pointing"
date:   2025-01-11 13:00:00 +0000
categories: embedded-systems function-pointers debugging
---

Recently, while reading an embedded systems engineering book by Elecia White, I realised that I was not fully comfortable with one of the chapters on function pointers. So, I decided to put theory into practice using one of my go-to development boards - ST's G4. In this post, I'll walk you through creating a serial command handler using function pointers, a technique that's invaluable when setting up development and debugging environments for embedded systems projects.

## Introduction

During board bring-up, we often need to run numerous tests repeatedly with hardware engineers. This could be due to various reasons: imperfect test implementations, hardware design issues, or even assembly problems. For debugging purposes, most embedded software engineers (myself included) prefer a command-line interface due to resource constraints. When tackling such bring-up tasks, we know we'll likely need to extend tests, modify them, or repeat them multiple times. A simple command-line interface, especially when implemented using function pointers early in development, can make this process much smoother.

## Quick Recap of Function pointers

Function pointers are lifesavers when we need to pass functions around and execute them dynamically based on commands or input data. In embedded systems, we commonly use them for event-driven programming, command processing, and state machine implementations. While many developers default to switch-case mechanisms for handling a small number of commands, I've come to appreciate the function pointer approach for its modularity and easy extensibility, particularly in testing and interface implementations.

## Serial command handler

Let's start with a small list of commands that we need to implement:

- help: *Display list of available commands*
- ping: *Return pong for comms verification*
- version: *Display firmware version information*
- temperature test: *Runs temperature tests, printing temperature data and errors*
- blink LED: *Sets an LED/IO to blink*

So, each command consists of a *name*, a *function* to call, and a *description* string. 

```C
typedef void (*functionPointerType)(void);
struct commandStruct
{
	char const *name;
	functionPointerType execute;
	char const *help;
};
```

And, an array of these will provide our list of commands:

```c
const struct commandStruct commands[]=
{
		{"help", &cmdHelp, "Display list of available commands."},
		{"ver", &cmdVersion, "Display firmware version."},
		{"tempTest", &cmdTempTest, "Run temperature test."},
		{"blinkLed", &cmdBlinkLed, "Set the LED to blink at 1Hz."},
		{"ping", &cmdPing, "Return pong for comms verification."},
		{"",0,""} // End of table indicator. Must be last!
};
```
The implementation is straightforward - we just need to create the command execution functions according to our function pointer definition. Using a character stream handler, we can easily execute the commands.
For simplicity, I'm using the HAL library and standard string functions. Here's the command handler function that manages the command-line interface:

```c
void HandleCommand(void) {
    if (command_ready)
    {
        command_ready = 0;

        // Iterate through the command table
        for (int i = 0; commands[i].execute != NULL; i++) 
        {
            if (strcmp((char *)uart_rx_buffer, commands[i].name) == 0) 
            {
                commands[i].execute(); // Call the corresponding function
                return;
            }
        }

        // If no match, send an error message
        printf("Unknown command: %s\n", uart_rx_buffer);
    }
}
```

If you're new to printf redirection in embedded systems, check out my previous post about [printf redirection](https://ycetindev.github.io/posts/2024-11-01-Redirecting-printf-to-UART.html)

The complete source code is available in this [repository](https://github.com/ycetindev/stm32g4/tree/main/fcnPointers).

## Testing the implementation

With this minimal setup, anyone can run hardware tests using a serial terminal like TeraTerm. Simply entering commands (start with 'help' to see available options) allows you to execute and extend your test suite as needed. Here's a quick demonstration:

![serial command handler demo](/assets/serial_command_handler.gif)

## Conclusion

I hope this post has clearly demonstrated how function pointers can simplify command handling while making your codebase more flexible for future expansions. Sure, you could stick with usual switch-case statement (we've all been there), but why settle for that when you could have a flexible command handler that's easier to extend than your deadline? 

In the next post, I aim to looking at how this process can be more automated using Python scripts. 
