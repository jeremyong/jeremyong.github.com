---
layout: post
title: I do, in fact, use a debugger
date: 2023-08-06 00:00
categories:
- Debugger
- Tools
---

... and frequently at that! Note: this is intended to be a light jab at [this post](https://lemire.me/blog/2016/06/21/i-do-not-use-a-debugger/).
No offense is intended, and I respect that others may find different ways of being productive in their work.
That said, I thought I'd jot down a dissenting opinion to the "I do not use a debugger" piece.

I learned to program with [BASIC](https://en.wikipedia.org/wiki/BASIC) back when I was 11. I shortly graduated
after that to ANSI C, armed with K&R's C programming book, mainly because I was trying to make stick-figure
fighting games and BASIC simply wasn't cutting it (I remember my program being a heinous mess of if statements,
and a single frame took seconds to render the next ASCII drawing). Debuggers weren't a thing then, and even
if they were, I doubt I would have understood how to properly use them.

Well, debuggers are a thing now, and boy do I use them. I use MSVC, LLDB, GDB, the debugger built-in to the web
browser, pretty much whatever gets the job done for the environment I'm working in. Occasionally, I can't
get away from `printf` statements or in-memory logs for some dirty piece of multi-threaded code.

I am not alone. In five minutes, I polled several people I knew who took positions with debuggers and reported
using them daily. There's also this guy named John Carmack that reportedly uses debuggers too, and I'm pretty
sure his code has shipped before.

I should make it clear that I do not think that there is one objective truth regarding tools. But I do think
that learning to use the right tools can make you far more productive. You wouldn't build a skyscraper without
power tools and heavy machinery, right?

Anyhow, so why do I use debuggers?

For what I do, I feel that `printf` does not scale. There is only so much time in a single frame in a game or
application running at 60 Hz, racing against the next vblank interval. There's also only so much time I have
patience for injecting `printf` statements and waiting for the entire damn program to recompile. I also don't
have the patience to kill the app, recompile it, then restart the app, only to find that the crash turned out
to be very difficult to reproduce. Sure I could write tests, but for a phantom crash or bug, how do I know
what test to write? What if the complexity involved in recreating the bug would require a test as complex as
the application itself? Just as electricians measure volts and amps directly with instruments like multimeters
and oscilloscopes, sometimes, as programmers, the right thing to do is inspect registers, memory, variables,
the program counter, and other bits of state directly. Maybe my debugger won't scale to hundreds of processors,
terabytes of data, and trillions of closely related instructions, but I don't feel like this is a risk. If I
needed to deploy to such a system, I would probably be in the business of improving my tools to debug such a
system.

Another important scenario where a debugger is useful is when reasoning about other people's code. Even on
projects that I work on alone, I often interface with third-party code, and in my opinion, it is important
to be able to understand how that code operates when things go wrong. In this case, the debugger is an
exploratory tool, because while I may be fast at reading source code line-by-line, letting the debugger
simply tell me what's happening is far more effective. Plus, debuggers don't lie except in exceptional
circumstances. I certainly trust the debugger more than I would trust my cursory inspection of a massive
multi-million LOC codebase I took no part in writing.

This leads me to the next point about debuggers. Sometimes, things break in a way that no amount of code
reading would predict. You pause the debugger, inspect a variable's value, then step and notice that the
variable changed despite no code visibly altering the variable. Ah, this variable's memory must be aliased
on a different thread. This is one scenario of countless scenarios where memory stomps, bad aliasing,
and other such issues would confound even the most astute reader. After identifying that one of these
issues is at play though, _at that point_, a closer inspection is certainly warranted, armed with clues
from debugger-land. I'm reminded of my time working in an electronics lab, where after hours of debugging
a circuit, I happened to notice that the circuit was malfunctioning because of a faulty breadboard. I
determined this by doing the standard "poke-and-prod with the oscilloscope probes" approach. No amount of
analysis about the particular arrangement of transistors, capacitors, resistors, and wires in front of
me would have determined the problem, and that's just sometimes how it is. The world is messy, and code
is no exception.

There are plenty of other tricks I rely on also. I can pause the debugger in response to a value changing.
I can set a conditional breakpoint faster than I could alter and recompile the program. I can freeze
multiple threads (or all but one thread) if I suspect a concurrency-related data violation. I can emit
data to the debugger only in the event the debugger is attached. I can do all this exploration _while still
thinking critically_ to understand what I should measure and inspect next to diagnose an issue. Using a
debugger doesn't mean I turn my brain off.

That said, even heavy debugger users shouldn't use the debugger at the exclusion of all other approaches though.
Sometimes, a problem warrants a different approach. The chief examples I can imagine are scenarios where a debugger
simply isn't available, for example, a multi-server deployment in the cloud. In addition, sometimes a log of
some sort is more appropriate to establish a chronological sequence of events. Just use what's effective, full stop.

Ben Deane, an experienced game developer who worked on Goldeneye, Medal of Honor, StarCraft, Diablo, World
of Warcraft and so forth, [wrote in 2018](https://www.elbeno.com/blog/?p=1598):

> The principal problem with debugging is that it doesn’t scale. (…) in order to catch bugs, we often need to be able to run with sufficiently large and representative data sets. When we’re at this point, the debugger is usually a crude tool to use (…) Using a debugger doesn’t scale. Types and tools and tests do.

You know what else doesn't scale? Writing a bespoke tool and test for literally every problem that
you might encounter during day-to-day development. Restarting an application repeatedly in order
to recreate a buggy/crashy scenario you have yet to understand. Dogmatically trying to approach
every problem by pretending you can just "read the code" and suddenly grok the entire universe,
despite the fact that most code you likely interact with was authored by someone else, or barring
that, insisting that the only code you ever use or interact with is code authored by you. Not to mention
that sometimes, you're allowed to write code that has a one-off purpose. Perhaps to prototype a feature,
or validate a design. That code doesn't need to scale to infinite complexity and thousands of contributors,
while running perfectly maintained on arbitrary hardware for any future as-yet-to-be-invented operating system.
Over-engineering is as much a sin as under-engineering.

That said, Ben (and Dr. Lemire) raise good points. There _is_ a need for tests to catch regressions, and general
automation, especially as designs and code dependencies materialize. But porque no los dos (why not both)?

Let me end with a quote that sums up my sentiment:

> > "Debuggers don't remove bugs. They only show them in slow motion." - Dr. Lemire

> Uh yes, did anyone claim otherwise?
