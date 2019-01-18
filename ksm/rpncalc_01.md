title: RPN Calc Part 1 – A simple calculator in Java, and the Command Pattern
date: 2013-10-21
sponsor: ksm
tags: ksm rpncalc java

The first implementation of the [RPN calculator](/ksm/rpncalc_00) I’m
going to look at is the basic Java implementation. The link takes you
directly a a view of the source file for this version, but if you’d
like to play with the code, you can also clone the whole project with
[git](https://github.com/ksmpartners/blog-rpncalc).  Cloning the git
project will bring down all the versions of the calculator, and also
let you compile and run the software locally. I’ll discuss more about
how the implementation works, after the jump:

When you run the calculator, it works a lot like an interactive
programming language with a read-eval-print-loop (REPL). The
calculator presents a prompt, reads a command, executes it, prints the
result, and then loops back around until the user types quit. A user
session that computes the sum of two and three looks like this (user
entry in bold):

```clojure
Begin: class com.ksmpartners.rpncalc.basic.RpnCalc

> 2

1> 2.0
> 3

1> 2.0
2> 3.0
> +

1> 5.0
> quit
end run.
```

The implementation has to address several problems:

* The stack has to be stored.
* The current state of the calculator needs to be displayed to the user.
* Commands need to be accepted from the user.
* User commands need to be dispatched to code that performs the requested action.

Storage of the stack is simple. The basic version of the calculator
uses a Java collection stored as an instance variable of the
calculator class:

```java
public class RpnCalc extends Calculator
{
    //...
 
    private Deque<Double> stack = new LinkedList<Double>();
```

Printing the stack is also simple, although it’s made more verbose by
the fact that the Java foreach loop requires an `Iterable`, rather than
an `Iterator`:


```java
private void showStack()
{
    int ii = 0;
 
    for (Iterator<Double> it = stack.descendingIterator(); it.hasNext(); ) {
        Double val = it.next();
 
        System.out.println((ii + 1) + ">" + val);
        ii++;
    }
}
```

Similarly, the read, eval, print loop looks almost exactly like you’d
expect. The most interesting bits are these:

```java
Command cmd = parseCommandString(cmdLine.trim());
 
cmd.execute();
```

The last two statements translate whatever the user entered into a
Command, an object that encapsulates the user’s intent into something
that can be stored and directly executed at a later point in time. It
is a command in the sense of the GoF Command Pattern:

```java
interface Command
{
    void execute();
}
```

The command themselves do their work via direct mutation of the state
stored in the stack instance variable:

```java
cmds.put("+", new Command() {
    public void execute() {
        Double x = stack.pop();
        Double y = stack.pop();
 
        stack.push(x + y);
    }});
```

One tricky detail of this design is hidden in the fact that every
command string entered by the user ultimately goes through
`Command.execute()`. There is no special case for entering numbers,
which therefore must also be done via a command. What’s different
about the numeric entry command is that it contains a bit of instance
data representing the number to be entered. The command parser creates
a separate instance of `PushNumberCommand` for each number that’s
entered on the command line.

```java
private class PushNumberCommand implements Command
    {
        Double number;
 
        PushNumberCommand(Double number) { this.number = number; }
        public void execute() { stack.push(number); }
    }
```

In this design, you can make a strong argument that `parseCommandString`
serves as a rudimentary compiler. Rather than translating from Java
into classfiles, it translates from text into command instances. While
this specific implementation only translates from strings containing a
single command into command objects that perform a single operation,
there’s nothing inherent in the design that prevents it from parsing
complex command strings into complex command objects:

```java
private Command parseCommandString(String cmdStr)
    throws Exception
{
    Command cmd = cmds.get(cmdStr);
 
    if (cmd != null)
        return cmd;
    else
        return new PushNumberCommand(Double.parseDouble(cmdStr));
}
```

Next time, I’ll talk a bit more about generalizing `parseCommandString`
into a more powerful kind of compiler. This will enrich both the user
command entry syntax, and give us a more powerful example of the
command pattern within the calculator.

