title: RPN Calc Part 9 – State and Commands in Clojure
date: 2014-12-15

In my last post, I started porting the RPN calculator example from
Java to Clojure, moving a functional program into a functional
language. In this post, I finish the work and show how the Clojure
calculator models both state and calculator commands.

Going back to the last post, the Clojure version of the
Read-Eval-Print-Loop (REPL) has the following code.

```clojure
(defn main []
  (loop [ state (make-initial-state) ]
    (let [command (parse-command-string (read-command-string state))]
      (if-let [new-state (apply-command state command)]
        (recur new-state)
        nil))))
```

As with the Java REPL, this function continually loops, gathering
commands to evaluate, evaluating them against the current state, and
printing the state after each command is executed. The REPL function
controls the lifecycle of the calculator state from beginning to end,
starting by invoking the state constructor function:

```clojure
(defn make-initial-state []
  {
   :stack ()
   :regs (vec (take 20 (repeat 0)))
   })
```

Like `main`, the empty brackets signify that this is a 0-arity function,
a function that takes 0 arguments. Looking back at the call site, this
is why the name of the function appears by itself within the
parenthesis:

```clojure
(make-initial-state)
```

If the function required arguments, they’d be to the right of the
function name at the call site:

```clojure
(make-initial-state <arg-0> ... <arg-n>)
```

This is the way that Lisp like languages represent function and macro
call sites. Every function or macro call is syntactically a list
delimited by parenthesis. The first element of that list identifies
the function or macro being invoked, and the arguments to that
function or macro are in the second list position and beyond. This is
the rule, and it is essentially universal, even including the syntax
used to define functions. In this form, `defn` is the name of the
function definition macro, and it takes the function name, argument
list, and body as arguments:

```clojure
(defn make-initial-state []
  {
   :stack ()
   :regs (vec (take 20 (repeat 0)))
   })
```

For this function, the body of the function is a single statement, a
literal for a two element hash map. In Clojure, whenever run time
control flow passes to an object literal, a new instance of that
literal is constructed and populated.

```clojure
{
 :stack ()
 :regs (vec (take 20 (repeat 0)))
 }
 ```
 
This one statement is thus the rough equivalent of calling a
constructor and then a series of calls to populate the new
object. Paraphrasing into faux-Java:

```java
Mapping m = new Mapping();
 
m.put("stack", Sequence.EMPTY);
m.put("regs", vec(take(20, repeat(0)));
```

Once the state object is constructed, the first thing the REPL has to
do is prompt the user for a command. The function to read a new
command takes a state as an argument. This is so it can print out the
state prior to prompting the user and reading the command string:

```clojure
(defn read-command-string [ state ]
  (show-state state)
  (print "> ")
  (flush)
  (.readLine *in*))
```

This code should be fairly understandable, but the last line is worthy
of an explicit comment. `*in*` is a reference to the usual
`java.lang.System.in`, and the leading dot is Clojure syntax for
invoking a method on that object. That last line is almost exactly
equivalent to this Java code:

```java
System.in.readLine();
```

There’s more use of Clojure/Java interoperability in the command parser:

```clojure
(defn parse-command-string [ str ]
  (make-composite-command
   (map parse-single-command (.split (.trim str) "\\s+"))))
```

The Java-interop part is in this bit here:

```clojure
(.split (.trim str) "\\s+")
```

Translating into Java:

```java
str.trim().split("\\s+")
```

Because `str` is a `java.lang.String`, all the usual string methods are
available. This makes it easy to use standard Java facilities to trim
the leading and trailing white space from a string and then split it
into space-delimited tokens. Going back to part 2 of this series, this
is the original algorithm I used to handle multiple calculator
commands entered at the same prompt.

The rest of `parse-command-string` also follows the original part-2
design: each token is parsed individually as a command, and the list
of all commands is then assembled into a single composite command. The
difference is that there’s less notation in the Clojure version,
mainly due to the use of the higher-order function `map`. `map` applies a
function to each element of an input sequence and returns a new
sequence containing the results. This one function encapsulates a
loop, two variable declarations, a constructor call, and the method
call needed to populate the output sequence:

```java
List<Command> subCmds = new LinkedList<Command>();
  
for (String subCmdStr : cmdStr.split("\\s+"))
    subCmds.add(parseSingleCommand(subCmdStr));
```

What’s nice about this is that eliminating the code eliminates the
possibility of making certain kinds of errors. It also makes the code
more about the intent of the logic, and less about the mechanism used
to achieve that intent. This opens up optimization opportunities like
Clojure’s lazy evaluation of mapping functions.

The final bit of new notation I’d like to point out is the way the
Clojure version represents commands. Commands in the Clojure version
of the calculator are functions on calculator state, represented as
Clojure functions:

```clojure
(fn [ { [x y & more] :stack } ]
    { :stack (cons (+ y x) more)})
```

This function, the addition command, accepts a state object and uses
argument list destructuring to extract out the stack portion of the
state. It then assembles a new state object that contains a version of
the stack that contains the sum of the top two previous stack
elements. Rather than focusing on the machinery used to gather and
manipulate stack arguments, Clojure’s notation makes it easier for the
code behind the command to match the intent. As before, this helps
reduce the chance for errors, and it also opens up new optimization
opportunities.

(If you’ve read closely and are wondering what happened to `regs`,
commands in the Clojure version of the calculator can actually return
a partial state. If a command doesn’t return a state element, then the
previous value for that state element is used in the next
state. Because add doesn’t change `regs`, it doesn’t bother to return
it.)


