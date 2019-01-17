title: RPN Calc Part 6 – Refactoring the REPL with an Iterator
date: 2014-01-02
sponsor: ksm

Up to now, the calculator’s main command loop has been a
straightforward implementation of a REPL, or
‘read-eval-print-loop’. If you’re unfamiliar with the term, REPLs are
the traditional means that interactive programming languages use to
provide their interactivity. REPL’s provide a command prompt that a
user can use to explore and manipulate the programming environment. In
this way, a REPL makes it possible to work more quickly than
traditional environments that require a program to be recompiled and
restarted to test code changes.

While REPLs can become very complex in the details, the core idea is
quite simple. As the name implies, REPL’s read a command from the
user, evaluate that command, print the result of that evaluation, and
loop back to start again. In rpncalc, all four of these steps are
clearly evident in the code of the REPL. This is useful for
explanatory purposes, but it closely couples the REPL to specific
implementations of ‘read’, ‘evaluate’ and ‘print’. For this post,
we’ll look into another way to model a REPL in code that offers a way
to break this coupling.

The main command loop of rpncalc contains explicit code for each of
the steps in an REPL:

```java

// Set initial state
State state = new State();
 
// Loop until we no longer have a state.
while(state != null) {
 
    // Print the current state
    System.out.println();
    showStack(state);
 
    // Print a prompt, and read the next command from the user.
    System.out.print("> ");
    String cmdLine = System.console().readLine();
 
    if (cmdLine == null)
        break;
 
    Command cmd = parseCommandString(cmdLine);
 
    // Evaluate the command and produce the next state.
    state = cmd.execute(state);
}
```

This code is easy to read and explicit in intent, but it totally
breaks down if commands can’t be read from the console. In the case of
a REPL running on a server, it may be the case that a REPL needs to
print and read over a (secured!) network connection. What would be
useful is a way to decouple the mechanism for reading command from the
loop itself.

In functionally oriented languages, this problem can be addressed by
extending the REPL function with function arguments. These function
arguments allow different implementations of read and print to be
plugged into the same basic loop structure. Default implementations
can be provided that connect to the console, with other
implementations that might read and print using a network connection,
or some other command transport. In Java, a similar effect can be
achieved using functional interfaces (aka SAM types) to provide the
pluggable alternative implementations. In fact, Java 8’s syntax for
anonymous functions will make this approach syntactically
convenient. Java also provides ways to achieve this extensiblity via
class derivation.

Another way to view this problem can be seen by slightly changing your
perspective on the REPL. It may not be completely obvious, but as with
many loops, the REPL is iterating over a sequence of values. In the
case of the REPL, the sequence is the sequence of commands that the
user enters in response to prompts. For each command in the sequence,
the REPL updates the current state and advances to the next sequence
element. This isn’t as concrete as iterating over an in-memory data
structure (and it isn’t necessarily bounded) but the semantics of the
iteration are the same. The key to implementing this design is to
provide a version of Iterable that implements iteration over a command
stream. Given an iterable command stream, the REPL takes on a slightly
different character:

```java
// Set initial state
State state = new State();
 
// Loop over all input commands
for(Command cmd : new ConsoleCommandStream()) {
 
    // Evaluate the command and produce the next state.
    state = cmd.execute(state);
 
    if (state == null)
        break;
 
    // Print the current state
    showStack(state);
}
```

Compared to the initial loop implementation, this version is
completely detached from the mechanisms used to prompt the user for
input and read incoming commands. The termination criteria is also
simpler: there isn’t an explicit check for the end of the command
stream. The implicit termination check within the foreach loop
captures that requirement.

The other component of this implementation is the implementation of
the `CommandStream`. Unfortunately, this is where Java extracts its
tax in lines of code for the additional modularity of this
design. Like all iterable objects, the console command stream
implements the `Iterable` interface. The iterator itself is defined as
an anonymous inner class:

```java
class ConsoleCommandStream implements Iterable<Command>
{
    public Iterator<Command> iterator()
    {
        return new Iterator<Command> ()
        {
```

One of the complexities of implementing Java’s `Iterator` interface is
that callers must be able to call `hasNext` any number of times (zero
to n) before each call to next. It’s not possible to assume one and
only one call to `hasNext` for each call to next, despite the fact
that the foreach does make that guarantee. Without going into the
details, this implies that the actual advance operation can occur
within either next or hasNext. While there are several ways to
implement this, the approach I like to use is to have a separate
method that advances the iterator, but only if it needs to be
advanced. (Calls to next put the iterator into ‘requires advance’
state.) The `advanceIfNecessary` method is where the bulk of the work
of the command stream takes place, including prompting the user, and
reading and parsing the command.

```java
Command nextCmd = null;
 
private void advanceIfNecessary()
{
    if (nextCmd != null)
        return;
 
    System.out.println();
    System.out.print("> ");
 
    String cmdLine = System.console().readLine();
 
    if (cmdLine == null)
        return;
 
    try {
        nextCmd = parseCommandString(cmdLine);
    } catch (Exception ex) {
        throw new RuntimeException("Error while parsing command: " + cmdLine, ex);
    }
}
```

In this way, Java’s built in support for iteration can be used to
break the REPL apart into sub-compoments for handling the stages of
command processing. The REPL is still clearly a REPL, but it no longer
has explicit dependencies on the means used to acquire input
commands. Unfortunately, the REPL still has explicit coupling for
command evaluation and printing the result. As it stands now, we could
modify the REPL to read commands from a network port, but we couldn’t
redirect the output away from the local console. In the next post in
the series, we’ll use the idea of reduce from functional programming
to break the REPL into a pipeline of iterators. This will bring the
rest of the flexibility we need.


