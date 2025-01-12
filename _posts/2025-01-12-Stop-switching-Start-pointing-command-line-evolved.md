---
layout: post
title:  "Stop switching, start pointing - command line evolved"
date:   2025-01-12 00:00:00 +0000
categories: embedded-systems python GUI Claude
---

In my [previous post](https://ycetindev.github.io/posts/2025-01-11-Stop-switching-Start-pointing.html), I shared how function pointers can simplify command handling in embedded systems. As promised, I wanted to explore how we can make this process more user-friendly with a Python-based GUI. But here's where it gets interesting - instead of writing it myself from scratch, I decided to let AI do the heavy lifting.

## The AI Approach

I used Claude 3.5 Sonnet for this task, and honestly, the prompt was almost embarrassingly simple. I just shared my command structure (as I shown in the previous post) and asked for "a simple Python based user interface." That's it. No complex specifications, no detailed requirements - just a basic request for a GUI that could handle my serial commands.

You know, this whole process made me laugh about my own journey with serial interfaces. Ten years ago, I was that bright-eyed university student who thought using C# for serial interfaces made me a "real programmer". Five years ago, I felt pretty clever about switching to Python! And now? Well, today I am wise, so I am using Claude.

## The Result

The resulting application includes everything I needed:
- Port selection and connection management
- Command history with hex view for debugging
- Line ending options (crucial for cross-platform compatibility)
- A clean, responsive interface that actually looks professional

Here's a quick demo of the interface in action:

![Python-GUI](/assets/python-gui.gif)

## Conclusion

While the core embedded implementation still requires careful consideration (as discussed in Part 1), the testing interface doesn't have to be a time sink. Whether you're using function pointers in your embedded code or creating testing interfaces, sometimes the best approach is knowing when to let AI handle the routine parts of development.

The complete source code for both the embedded implementation and the Python GUI is available in this [repository](https://github.com/ycetindev/stm32g4/tree/main/fcnPointers/scripts).