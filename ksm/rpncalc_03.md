title: RPN Calc Part 3 – Undo
date: 2013-11-19
sponsor: ksm
tags: ksm rpncalc java

One of the reasons given in the GoF book for the use of the command
pattern is to support undo. By recording the commands executed by a
user, and giving the commands the ability to reverse themselves, a
user interface can be designed to allow users to undo mistakes they
make. For this post, I’ll talk about how this is done by extending the
[last version of the calculator](/ksm/rpncalc_02) to support undo.

There are three main extensions to rpncalc that enable it to support
undo. The first is that I’ve broadened the `Command` interface into an
abstract class that supports operations for managing state. There is a
new method, `saveState`, that saves a copy of the globally shared state
within the command instance itself:

```java
abstract class Command
{
    private Deque<Double> oldStack;
    private Double[] oldRegs;
 
    void saveState()
    {
        oldStack = new LinkedList<Double>(stack);
        oldRegs = Arrays.copyOf(regs, regs.length);
    }
```

There is also the inverse of `saveState`, `restoreState`:

```java
void restoreState()
{
    stack = new LinkedList<Double>(oldStack);
    regs = Arrays.copyOf(oldRegs, oldRegs.length);
}
```

Taken together, these changes provide the two new fundamental
operations necessary to support undo: Saving global state, in case an
undo operation requires that it be restored at a later point in
time. If the definition of global state ever broadens to include other
variables, the change can be confined to these two methods. If there’s
ever a more efficient way for a command to save state than a full
copy, the change can be confined to these two methods. In this way the
two methods provide an abstraction over the idea of ‘global state’. By
adding methods (verbs) to manipulate global state, we’ve extended the
language to allow direct statements about global state.

The next step is to use this new abstraction to implement undo. The
first step is easy: each command needs to be modified to save the
current global state before it executes. There are other ways to do
this, but what I’ve done here is add a call to `saveState` at the
beginning of each command implementation. This is simple, and for a
small program like rpncalc, it works well:


```java
cmds.put("+", new Command() {
        public void execute() {
            saveState();
 
            Double x = stack.pop();
            Double y = stack.pop();
 
            stack.push(x + y);
        }
    });
```

Implementing undo is just a bit trickier. One way to think about undo
is that it restores the state at the beginning of the last command
that was executed. To implement undo in these terms requires that we
keep a record of the last executed command.

```java
Command cmd = parseCommandString(cmdLine);
 
cmd.execute();
 
lastCmd = cmd;
```

From here, ‘undo’ is a new command that restores the state at the
beginning of `lastCmd`:

```
cmds.put("undo", new Command() {
        public void execute() {
            saveState();
 
            lastCmd.restoreState();
        }
    });
```

Note that undo itself saves its own state. This makes undo a
reversible operation: ‘undo’ can be ‘undone’.

To this point, we’ve developed the basic calculator idea around the
idea of state contained in a collection of global variables. The
methods `saveState` and `restoreStateq provide a nice, verb-oriented
way to hide the details behind state management.

This approach has a great deal of headroom, but it doesn’t take
advantage of the capabilities of the language to logically group data
into tuples. The next installation of this series will look into a way
we can use a Java object to package our entire calculator state into a
single object that can be manipulated as a whole. If this post
introduced verbs for managing state, the next post will introduce the
noun.
