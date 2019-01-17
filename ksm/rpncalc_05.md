title: RPN Calc Part 5 – Eliminating the Globals
date: 2013-12-17
sponsor: ksm

Throughout the last four parts of this series, the common theme has
been that the state of the calculator program has been managed
globally. This requires the main command loop to directly update
global data after each command, to prepare the state for the next
command. While this works, it would be nice to remove the need for the
global update. This post talks about how that’s done in the functional
version of rpncalc.

The [last installment](/ksm/rpncalc_04) in the series updated the
calculator by adding a Java class to model `State`. This allows the
entire state of the calculator to be managed as a first-class entity
within the program. That’s evident from the fact that the calculator
now models state through a single global reference:

```java
public class RpnCalc extends Calculator
{
    // ...
 
    private State state = null;
```

Before I continue, I should clarify by saying that `state` isn’t truly a
global variable. Obviously `state` is an instance variable within a
single instance of `RpnCalc`. However, because of the fact that the
calculator program only runs with a single instance of `RpnCalc`, any
instance variables within that class are effectively global within the
program, and suffer many of the same problems associated with true
globals. This parallel is also true for instance variables within
singletons managed by Spring. They’re instance variables, but they act
like globals and should be viewed with the same kind of suspicion.

To remove the global state, we’re going to take advantage of the fact
that the last update to rpncalc changed the signature for a command,
so that it returns a new copy of the state:

```java
abstract class Command
    {
        State execute(State in)
```

This implies that the main command loop has direct access to the
updated state, and can manage it as a local variable:

```java
public void main()
        throws Exception
    {
        State state = new State();
 
        while(state != null) {
            // ...read and gather command...
            state = cmd.execute(state);
        }
    }
```

This eliminates the global variable, moving the same data to a single
variable contained within a function definition. This hugely reduces
the variable’s footprint within the code. The global variable can be
read and updated by 200 lines of code, compared to 20 lines of code
for the local variable. For a developer, this makes the management of
state more evident within the code, and makes it easier to write and
maintain correct code.

Unfortunately, there’s a problem. For most of the commands, storing
state in a local variable works well: a command like addition only
needs access to the current state to produce the next state. Where it
breaks down is in the undo command. In contrast to every other
command, undo doesn’t need access to the current state, it needs
access to the previous state. Now that we’ve moved the state
management entirely within the command loop function, there’s no way
for the undo command to get access to the history of states. Given
that undo is a requirement, we need to find a solution.

One approach would be to extend the command loop to maintain a full
history of states, and then broaden the command function signature to
accept both the current and previous states. Each command would then
be a function on a history of states. This approach has some appeal,
but a simpler way to achieve the same result is to acknowledge the
fact that each state is a function of the previous state, and store a
back reference within each state:

```java
private class State
{
    State prev;
    // ...
 
    State()
    {
        prev = null;
        // ...
    }
 
    State(State original)
    {
        prev = original;
        // ...
    }
}
```

This makes undo easy:

```java
cmds.put("undo", new Command() {
        public State execute(State in) {
            // skip past the copy made for 'undo', and get to the previous.
            return in.prev.prev;
        }
    });
```

This design is also closely related to how git stores commits: each
commit stores a back link to the previous commit in the history. It
does have potentially unbounded memory usage, but our state in rpncalc
is small enough in size and our memory is large enough that this
shouldn’t present a problem. If it ever did present a memory
consumption problem, adding a bound on the length number of retained
states is as simple as walking down the previous links and nulling the
previous link of the last history element to retain.

Next up, we’ll start taking a bit more advantage of the newly
functional nature of rpncalc, and start using functional techniques to
decompose the main command loop into component modules.
