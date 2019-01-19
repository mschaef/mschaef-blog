title: RPN Calc Part 4 – A Noun for State
date: 2013-12-10
sponsor: ksm
tags: ksm rpncalc java

In the [last installment](/ksm/rpncalc_03) of this series, I built a
basic undo facility on top of the command pattern. One of the problems
with that implementation is that the `Command` class has to know too
much about how to save and restore the overall state of the
calculator. In this post, I’ll introduce a way around this problem.


To see the problem, note that every instance of `Commandq has a
private member variable for each global state variable maintained by
the calculator:

```java
private Deque<Double> oldStack;
private Double[] oldRegs;
```

Commands also have methods to save and restore the global state from
the calculator instance itself:

```java
void saveState()
{
    oldStack = new LinkedList<Double>(stack);
    oldRegs = Arrays.copyOf(regs, regs.length);
}
 
void restoreState()
{
    stack = new LinkedList<Double>(oldStack);
    regs = Arrays.copyOf(oldRegs, oldRegs.length);
}
```

Aside from the fact that `Command`s are directly updating the global
state of the calculator, this design makes it more difficult to extend
or modify the calculator’s notion of state. Any change to the
calculator’s design that that adds a global state variable forces the
Command class to be extended to manage the new global state. This
means at least adding new private fields to Command and updating
saveState and restoreState implementations. The stateobject version of
RpnCalc addresses this issue by introducing a new class for the
purpose of managing calculator state: State.

As you might have guessed, `State` has the two fields we need to store
our current notion of state:

```java
Deque<Double> stack = null;
 Double[] regs = null;
```

There are also two constructors, one for constructing a new, blank
state, and a copy constructor for duplicating an existing state:

```java
State()
{
    stack = new LinkedList<Double>();
    regs = new Double[NUM_REGISTERS];
}
 
State(State original)
{
    stack = new LinkedList<Double>(original.stack);
    regs = Arrays.copyOf(original.regs, original.regs.length);
}
```

The `State` class simplifies the calculator’s state variable
declaration down to a single member variable:

```java
private State state = null;
```

Along with that simplification, `Command` no longer needs to worry about
state management, so it’s back down to a single method:

```java
abstract class Command
{
    abstract State execute(State in);
}
```

The biggest change here is that the execute method now returns an
instance of `State`. With state objects, the behavior of a command is
now that it accepts a ‘before’ state, and then returns the state that
results from applying the command to the output state. Every
implementation of `Command`, save for two, first makes a copy of the
state using the copy constructor, and then updates the copied state
and returns it to the caller. The command doesn’t touch the input
state, aside from the initial copy operation (which isn’t an update).

At this point, you might want to re-read that last paragraph - it’s
kind of a big deal. With this latest change to the calculator, we’ve
really made two big improvements. The first is the change that
motivated the post in the first place: `Command`s no longer need to know
how to save and restore state and the no longer need to be aware of
the contents of that state. The second change is that Commands are no
longer modifying global state at all. Each command is now a Function,
mapping from state to state.

This second aspect of the design change has many positive
implications, but the first is that it makes it easier to implement
undo. Because the input state passed into `Command.execute` isn’t
touched, the caller can save off a copy of the previous state in its
entirety. The calculator’s main command loop does just that:

```java
Command cmd = parseCommandString(cmdLine);
 
State initialState = state;
 
state = cmd.execute(state);
 
LastState = initialState; // (lastState is an instance variable alongside state)
```

The undo command itself just ignores its input state and returns
whatever the `lastState` was:

```
cmds.put("undo", new Command() {
        public State execute(State in) {
            return lastState;
        }
    });
```

This version of the calculator gets us most of the way to a
mathematically functional implementation. The next installment will
take us further by removing `state` and `lastState`.



