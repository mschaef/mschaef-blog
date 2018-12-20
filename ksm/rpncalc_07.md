title: RPN Calc Part 7 – Refactoring Loops with Reduce
date: 2014-06-01

In the last installation of this series, we started using Java
iterators to decompose the monolithic REPL (read-eval-print-loop) into
modular compoments. This let us start decoupling the semantics of the
REPL from the mechanisms that it uses to implement read, evaluate, and
print. Unfortunately, the last version of rpncalc only modularized the
command prompt itself: the ‘R’ in REPL. The evaluator and printer are
still tightly bound to the main command loop. In this post I’ll use
another kind of custom iterator to further decompose the main loop,
breaking out the evaluator and leaving only the printer itself in the
loop.

Going back to the original command loop from the stateobject version
of rpncalc, the loop traverses two sequences of values in parallel.

```java
state = new State();
 
while(running) {
    System.out.println();
    showStack();
    System.out.print("> ");
 
    String cmdLine = System.console().readLine();
 
    if (cmdLine == null)
        break;
 
    Command cmd = parseCommandString(cmdLine);
 
    State initialState = state;
 
    state = cmd.execute(state);
 
    lastState = initialState;
}
```

Neither of the two sequences this loop traverses are made
explicit within the code, both are implicit in the sequence of values
taken on by variables managed by the loop. The first sequence the loop
traverses is the sequence of commands that the user enters at the
console. This sequence manifests in the code as the sequence of values
taken on by cmd through each iteration of the loop. The second
sequence is similarly implicit: the sequence of states that state
takes on through each iteration. Last post, when we added the
`CommandStateIterator`, the key to that refactoring was that we took one
of the implicit loop sequences and made it explicitly a sequence witin
the code. Having an explicit structure within the code for the
sequence of commands provided a place for the loop to invoke the
reader that wasn’t in the body of the loop itself.

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

Looking forward, the next refactoring for the REPL is to make explicit
the implicit sequence of result states in the same way we transformed
the sequence of input commands. This will let us take our current
loop, which loops over input commands, and turn it into a loop over
states. The call to evaluate will be pushed into an iterator in the
same way that we pushed the reader into an iterator in the last
post. This leaves us with a main loop that simply loops over states
and prints them out:

```java
for(State state : new CommandStateReduction(new State(), new CommandStream()))
    showStack(state);
```

This code is short, but it’s dense: most of the logic is now outside
the text of the loop, and within `CommandStateReduction` and
`CommandStream`. The command stream is the same stream of commands used
in the last version of rpncalc. The ‘command state reduction’ stream
is the stream that invokes the commands to produce the sequence of
states. I’ve given it the name ‘reduction’ because of the way it
relates to reduce in funcional programming. To see why, look back at
abstract class we’re using to model a command:

```
abstract class Command
{
    abstract State execute(State in);
}
```

Given a state, applying a command results in a new state, returned
from the execute method. A second command can then be applied to the
new state giving an even newer state, and there’s no inherent bound on
the number of times this can happen. In this way, a sequence of
commands applied to an initial state produces a corresponding sequence
of output states. The sequence of output states is the sequence of
command results that the REPL needs to print for each entered
command. Each time a command is executed, the result state needs to be
printed and stored for the next command.

The relationship between this and reduction comes from the fact that
reduction combines the elements of a sequence into an aggregate
result. Reducing + over a list of numbers gives the sum of those
numbers. Applying a sequence of commands combines the effects of those
commands into a single final result. The initial value that gets
passed into the reduction is the initial state. The sequence over
which the reduction is applied is the sequence of commands from the
console. The combining operator is command application. The most
significant difference between this and traditional reduce is that we
need more than just the final result, we also need each intermediate
result. (This makes our reduction more like Haskell’s scan operator.)

Practically speaking `CommandStateReduction` is implemented as an
`Iterable`. The constructor takes two arguments: the initial state
before any commands are executed, and a sequence of commands to be
executed.

```java
class CommandStateReduction implements Iterable<State>
{
    CommandStateReduction(State initialState, Iterable<Command> cmds)
```

Note that the only property that the command state reduction requires
of the sequence of commands is that it be `Iterable` and produce
`Command`s. There’s nothing about the signature of the reduction
iterator that requires the sequence of commands to be concrete and
known ahead of time. This is useful, because our current command
source is `CommandStream`, which lazily produces commands. Both the
command stream and the command state reduction are lazily evaluated,
and only operate when a caller makes a request. The command stream
doesn’t read until the evaluator requests a command, the evaluator
doesn’t evaluate until the printer makes a request for a
value. Despite the fact that it’s hidden behind a pipeline of iterable
object, the REPL still operates as it did before: first it reads, then
it evaluates, then it prints, and then it loops back.

As with the command state iterator, most of the logic in command state
reduction is handled with a single `advanceIfNecessary` method. The
instance variable state is used to maintain the state between command
applications:

```java
private State state = initialState;
 
private boolean needsAdvance = true;
 
Iterator<Command> cmdIterator = cmds.iterator();
 
private void advanceIfNecessary()
{
    if (!needsAdvance)
        return;
 
    needsAdvance = false;
 
    if (cmdIterator.hasNext())
        state = cmdIterator.next().execute(state);
    else
        state = null;
}
```

Looking back at the code, the Java version of the RPN calculator has
come a long way. From heavily procedural origins, we’ve added command
pattern based undo logic, switched over to a functional style of
implementation, and redesigned our main loop so that it operates via
lazy operations on streams of values. We’ve taken a big step in the
direction of functional programming. The downside has been in the size
of the code. The functional style has many benefits, but it’s not a
style that’s idiomatic to Java (at least before Java 8). Our code side
has more than doubled from 150 to 320 LOC. In the next few entries of
this series, we’ll continue evolving rpncalc, but switch over to
Clojure. This will let us continue this line of development without
getting buried in the syntax of Java.
