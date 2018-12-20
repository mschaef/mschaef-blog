title: RPN Calc Part 2 – Composite Commands
date: 2013-10-28

If you’ve played around with the basic version of the Java calculator,
you may have tried to enter multiple commands on the same prompt line:

```clojure
Begin: class com.ksmpartners.rpncalc.basic.RpnCalc

> 1 2
Uncaught Exception: For input string: "1 2"
java.lang.NumberFormatException: For input string: "1 2"
```

This is a convenient way to enter calculator commands, but it doesn’t
work for two reasons. This post discusses why it fails, and how to fix
it:


The immediate failure is because the parser considers the entire input
string as a single command, without regard to whitespace. The parser
looks for a command named `"1 2"`, fails to find it, and then attempts
to parse the entire string as a number. The number parse then fails
and throws an exception that kills the process. This part of the
problem can be fixed by altering the parser to split the input string
into tokens, and parsing each token separately as its own
command. Unfortunately, this shows the second more structural
problem. The way the REPL is currently written, it’s built to read and
execute a single `Command` for each prompt:

```java
Command cmd = parseCommandString(cmdLine.trim());
 
cmd.execute();
```

To correct the second problem, the data flow between
`parseCommandString` and `execute` needs to be widened so that the parser
can return multiple commands for a single input string. The easiest
way to achieve this is to have the parser return a list of `Command`s.

```java
List<Command> cmds = parseCommandString(cmdLine.trim());
 
for(Command cmd : cmds)
    cmd.execute();
```

This approach works well because it’s simple. It doesn’t add any
additional class definitions, there’s still a single call site for
`execute()`, and `java.util.List` is well-understood by developers that
use Java. However, if you’re willing to trade some of that simplicity
away, there’s an alternative approach to the problem. The alternative
approach allows the original REPL to go unchanged, and still process
groups of commands. It does this by introducing a new and more
powerful kind of command object that can be used to compose lists of
commands into single `Command` instances. The parser then uses these
`CompositeCommand`s to signal to the REPL that there are multiple
commands to be executed:

```java
private Command parseCommandString(String cmdStr)
    throws Exception
{
    List<Command> subCmds = new LinkedList<Command>();
 
    for (String subCmdStr : cmdStr.split("\\s+"))
        subCmds.add(parseSingleCommand(subCmdStr));
 
    return new CompositeCommand(subCmds);
}
```

There are a few things to notice about this code. The first is that
the type signature of `parseCommandString` is unchanged: it’s still a
function from `String` to `Command`. This is why the REPL doesn’t need to
be updated. The second aspect of the code is that there’s still a list
of command objects: The list of commands we introduced when we
modified the REPL example is still present, even with the
`CompositeCommand`. What the composite command object does is hide the
list from the REPL; It wraps the list up, abstracts it away, and lets
the REPL pretend that there aren’t any lists.

The internal machinery that lets composite command achieve this isn’t
too involved. All `CompositeCommand` has to do is remember the `Command`
objects it’s composing, and execute each of those commands as if they
were entered individually:

```java
private class CompositeCommand implements Command
{
    private List<Command> subCmds = new LinkedList<Command>();
 
    CompositeCommand(Collection<Command> subCmds)
    {
        this.subCmds.addAll(subCmds);
    }
 
    public void execute()
    {
        for(Command subCmd : subCmds)
            subCmd.execute();
    }
}
```

And that’s all there is to it. While this approach doesn’t have the
surface simplicity of our first attempt, what it has done for us is
give us a new abstraction for sequences of commands. This is fully a
third of the ‘sequence’, ‘selection’, and ‘iteration’ that are
necessary to express any computable program. That’s not a bad result
for a single class definition.

