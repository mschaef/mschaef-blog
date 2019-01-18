title: Middle School Math, Reverse Polish Notation, and Software Design
date: 2013-10-03
sponsor: ksm
tags: ksm rpncalc

One of the things I remember most about middle school math class is
that I went through it in a perpetual state of disorganization. During
one particularly bad spell, I lost two calculators within a week. The
loss, and the reaction of my parents, drove me to try to fix the
problem once and for all. My plan was simple: buy an expensive
calculator with the hope that it’d serve as an incentive to keep track
of my stuff. The next weekend, I took weeks of allowance money to the
local [Service Merchandise](https://en.wikipedia.org/wiki/Service_Merchandise) 
and bought a new [HP-11C](http://www.hpmuseum.org/hp11c.htm)
pocket calculator. Almost 30 years later, I still have both the calculator
and a fascination for its unusual
[Reverse Polish Notation (RPN)](https://en.wikipedia.org/wiki/Reverse_Polish_notation)
user interface. Over those decades, I’ve also found out that RPN provides a
good way to explore a number of fundamental ideas in the field of
software design.


If you haven’t used an RPN calculator, your first attempt will
probably be a confusing experience. Unlike most calculators, The HP
didn’t have an ‘=’ key. Instead, it had a large button down the center
labeled ‘Enter’, which pushed the most recently keyed number onto an
internal four-level stack. The mathematical operations then worked
against this stack. The ‘+’ key popped off two numbers, added them,
and pushed the result back onto the stack. To add 2 and 3, you’d make
the following keystrokes: `[2][ENTER][3][+]`.

In detail:

* `[2]` – Begin entering the number ‘2’ into the top level of the stack.
* `[ENTER]` – Duplicate the number ‘2’ on the top of the stack so it’s in the top two levels.
* `[3]` – Begin entering the number ‘3’ into the top level of the stack, replacing one of the copies of the number ‘2’.
* `[+]` – Pop off the top two levels of the stack, pushing back the sum of the two numbers.

Because the display shows the top level of the stack, it shows the
answer (5) as soon as the user presses `[+]`. Because the answer, 5,
is on the top of the stack, it’s also immediately ready to be used as
an input to another calculation. This last bit is why RPN calculators
can be so compelling to use: they make it easy to start with a small
calculation and extend it into something larger. As long as a number
is on the stack, it can be used in a calculation; The origin of the
number doesn’t matter to the way it can be used. In computer science
terms, this is the beginning of Referential Transparency. The FORTH
programming language builds on this foundation, extending the basic
tenets of RPN into a complete programming language. RPN combined with
functional decomposition gives the language a great deal of expressive
power, but due to the simplicity of stacks a small FORTH can be
implemented in a very small amount of memory.

As this series of blog posts continues, I intend to explore some of
these ideas using a set of RPN calculator implementations written in
Java and in Clojure. We’ll start off with a simple implementation in
Java, spend a bit of time exploring the command pattern, and then move
into more functional approaches to the problem.




