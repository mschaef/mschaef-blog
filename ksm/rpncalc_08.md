title: RPN Calc Part 8 – Moving to Clojure
date: 2014-12-02

So far in this series, I’ve taken a basic calculator written in Java
and transformed it from a command-oriented procedural design into a
more functional style. In some ways, this has made for simpler code:
calculator state is better encapsulated in value objects, and explicit
control flow structures have been replaced with domain-specific higher
order functions. Unfortunately, Java wasn’t designed to be a
functional language, so the notation has become progressively more
cumbersome and lengthy. 151 lines of idiomatic Java is now 327 lines
of inner classes, custom iterators, and inverted control flow
patterns. It should be difficult to get this kind of code through a
serious Java code review.

Despite this difficulty, there is value in the functional design
approach; What we need is a new notation. To show what I mean, this
article switches gears and ports the latest version of the calculator
from Java to Clojure. This reduces the size of the code from 327 lines
down to a more reasonable-for-the-functionality 82. More importantly,
the new notation opens up new opportunities for better expressiveness
and further optimization. Building on the Clojure port, I’ll
ultimately build out a version of the calculator that uses eval for
legitimate purposes, and compiles calculator macros and can run them
almost as fast as code written directly in Java.


The first step to understanding the Clojure port is to understand how
it’s built from source. For the Java versions of the code, I used
Apache Maven to automate the build process. Maven provides standard
access to dependencies, a standard project directory structure, and a
standard set of verbs for building, installing, and running the
project. In the Clojure world, the equivalent tool is called
Leiningen. It provides the same structure and services for a Clojure
project as Maven does for a Java project, including the ability to
pull in Maven dependencies. While it’s possible to build Clojure code
with Maven, Leiningen is a better choice for new work, largely because
it’s more well integrated into the Clojure ecosystem out of the box.

For the RPN calculator project, the project definition file looks like this:

```clojure
(defproject rpn-calc "0.1.0-SNAPSHOT"
  :description "KSM Partners - RPN Calculator"
 
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
 
  :dependencies [[org.clojure/clojure "1.5.0"]]
 
  :repl-options {
                 :host "0.0.0.0"
                 :port 53095
                 }
 
  :main rpn-calc.main)
```

This file contains the same sorts of information as the equivalent POM
file for the Java version of the project. (In fact, Leiningen provides
way to take a Leiningen project definition file and translate it into
an equivalent Maven `pom.xml`.) Rather than XML, the Leiningen project
file is written in an S-expression, and it contains a few additional
settings. Notably, the last line is the name of the project’s entry
point: the function that gets called when Leiningen runs the
project. For this project, `rpn-calc.main` is a function that ultimately
delegates to one of three entry points for the three Clojure versions
of the calculator. For this post, the implementation specific entry
point looks like this:

```clojure
(defn main []
  (loop [ state (make-initial-state) ]
    (let [command (parse-command-string (read-command-string state))]
      (if-let [new-state (apply-command state command)]
        (recur new-state)
        nil))))
```

```java
public void main() throws Exception
{
    State state = new State();
 
    while(state != null) {
        System.out.println();
        showStack(state);
        System.out.print("> ");
 
        String cmdLine = System.console().readLine();
 
        if (cmdLine == null)
            break;
 
        Command cmd = parseCommandString(cmdLine);
 
        state = cmd.execute(state);
    }
}
```

Unwrapping the code, both function definitions include construction of
the initial state and then the body of the Read-Eval-Print-Loop. These
two lines of code include both elements.

```clojure
(loop [ state (make-initial-state) ]
    ...
    (recur new-state))
```

The `loop` form, surrounded by parentheses, is the body of the loop. Any
loop iteration variables are defined and initialized within the
bracketed form at the beginning of the loop. In this case, a variable
state is initialized to hold the value returned by a call to
make-initial-state. Within the body of the loop, there can be one or
more recur forms that jump back to the beginning of the loop and
provide new values for all the iteration variables defined for the
loop. This gives a bit more flexibility than Java’s while loop: there
can be multiple jumps to the beginning of a loop.

The body of this `loop` form is entirely composed of a `let` form. A
`let` form establishes local variable bindings over a block of source
code and provides initial values for those variables. If this sounds a
lot like a loop form without the looping, that’s exactly what it is.

```clojure
(let [command (parse-command-string (read-command-string state))]
   ...)
```
   
This code calls `read-command-string`, passing in the current state and
then passes the returned command string into a call to
`parse-command-string`. The result of this two step read process is the
Clojure equivalent of a command object, which is modeled as a function
from a calculator state to a state.

Digressing a moment, there are several attributes of the Clojure
syntax that are worth pointing out. The most important is that, as
with most Lisps, parenthesis play a major role in the syntax of the
language. Parenthesis (and braces and brackets) delimit all statements
and expressions, group statements into logical blocks, delimit
function definitions, and serve as the syntax for composite object
literals. In contrast, a language like Java uses a combination of
semicolons, braces, and parsing grammar to serve the same
purposes. This gives Clojure a more homogeneous syntax, but a syntax
with fewer rules that’s easier to parse and analyze. Explicit
statement delimiters also allow Lisp more freedom to pick symbol
names. Symbols in Lisp can include characters (‘-‘, ‘<', '&', etc.)
that infix languages can't use for the purpose, because the explicit
statement grouping makes it easier to distinguish a symbol from its
context. The topic of Lisp syntax is really interesting enough for its
own lengthy series of posts and articles. Going back to the Clojure
calculator's main loop, the next statement in the loop is yet another
binding form. Like loop, this binding form also includes an element of
control flow.

```clojure
(if-let [new-state (apply-command state command)]
   (recur new-state)
   nil)
```

It may be easiest to see the meaning of this block of code by
paraphrasing it into Java:

```java
State newState = applyCommand(state, command);
 
if (newState != null)
    return recur(newState);
else
    return null;
```

What `if-let` does is to establish a new local variable and then
conditionally pick between two alternative control flow paths based on
the value of the new variable. It’s a common pattern within code, so
it’s good to have a specific syntax for the purpose. What’s
interesting about Clojure, though, is that if the language didn’t have
it built in, a programmer working in Clojure could add it with a macro
and you couldn’t tell the difference from built-in features. (In fact,
the default Clojure implementation of `if-let` is itself a macro.)

At this point, I’ve covered the basic structure of the Clojure
project, as well as the project’s main entry point. Subsequent posts
will cover modeling of application state within Clojure, as well as
the command parser, and the commands themselves. Once I’ve covered the
basic functionality of the calculator, I’ll use that as a starting
point to discuss custom syntax for command definitions, and ultimately
a compiler for the calculator.


