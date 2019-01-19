title: RPN Calc Part 10 – Macros and the Intent of the Code
date: 2014-12-20
sponsor: ksm
tags: ksm rpncalc java clojure

One of the key attributes I look for when writing and reviewing code
is that code should express the intent of the developer more than the
mechanism used to achieve that intent. In other words, code should
read as much as possible as if it were a description of the end goal
to be achieved. The mechanism used to achieve that goal is secondary.

Over the years, I’ve found this emphasis improves the quality of a
system by making it easier to write correct code. By removing the
distraction of the mechanism underneath the code: it’s easier for the
author of that code to stay in the mindset of the business process
they’re implementing. To see what I mean, consider how hard it would
be to query a SQL database if every query was forced to specify the
details of each table scan, index lookup, sort, join, and filter. The
power of SQL is that it eliminates the mechanism of the query from
consideration and lets a developer focus on the logic itself. The
computer handles the details. Compilers do the same sort of thing for
high level languages: coding in Java means not worrying about register
allocation, machine instruction ordering, or the details of free
memory reclamation. In the short-term, these abstractions make it
easier to think about the problem I’m being paid to solve. Over a
longer time scale, the increased distance between the intent and
mechanism makes it easier to improve the performance or reliability of
a system. Adding an index can transparently change a SQL query plan
and Java seamlessly made the jump from an interpreter to a compiler.

One of the unique sources of power in the Lisp family of languages is
a combination of features that makes it easier build the abstractions
necessary to elevate code from mechanism to intent. The combination of
dynamic typing, higher order functions, good data structures, and
macros can make it possible to develop abstractions that allow
developers to focus more on what matters, the intent of the paying
customer, and less on what doesn’t. In this article, I’ll talk about
what that looks like for the calculator example and how Clojure brings
the tools needed to focus on the intent of the code.


To level set, I’m going to go back to the calculator’s addition command
defined in the [last installment](/ksm/rpncalc_09) of this series.:

```clojure
(fn [ { [x y & more] :stack } ]
   { :stack (cons (+ y x) more)})
```

Given a stack, this command removes the top two arguments from the
stack, adds them, and pushes the result back on top of the stack. This
stack:

```clojure
1 2 5 7
```

becomes this stack:

```clojure
3 5 7
```

While the Clojure addition command is shorter than the Java version,
the Clojure version still includes a number of assumptions about the
machinery used in the underlying implementation:

* Calculator state is passed to the command as a map with a key `:stack` that holds the stack.
* The input stack can be destructured as a sequence.
* The output state is represented in a map allocated at the end of the command’s execution.
* The output stack is a sequence of cons cells and the output of this command is stored in a newly allocated cell.
* The command has a single point in time at which it begins execution.
* The command has a single point in time at which it ends execution.
* The execution of this command cannot overlap with other commands that manipulate the stack.

Truth be told, there isn’t a single item on this list that’s essential
to the semantics of our addition command. Particularly in the case
where a sequence of commands is linked together to make a composite
command, every item on that list might be incorrect. This is because
the state of the stack between elements of a composite command might
not ever be directly visible to the user. Keeping that in mind, what
would be nice is some kind of shorthand notation for stack operations
that hides these implementation details. This type of notation would
make it possible to express the intent of a command without the
machinery. Fortunately, the programming language Forth has a stack
effect notation often used in comments that might do the trick.

Forth is an interesting and unique language with a heavy dependency on
stack manipulation. One of the coding conventions sometimes used in
Forth development is that every ‘composite command’ (‘word’, in Forth
terminology) is associated with a comment that shows a picture of the
stack at the beginning and end of the command’s execution. For
addition, such a comment might look like this:

```forth
: add ( x y -- x+y ) .... ;
```

This comment shows that the command takes two arguments off the top of
the stack, ‘x’ and ‘x’, and returns a single value ‘x+y’. None of the
details regarding how the stack is implemented are included in the
comment. The only thing that’s left in the comment are the semantics
of the operation. This is more or less perfect for defining a
calculator command. Mapped into Clojure code, it might look something
like this:

```clojure
(stack-op [x y] [(+ x y)])
```

This Clojure form indicates a stack operation and has stack pictures
that show the top of the stack both before and after the evaluation of
the command. The notation is short, yes, but it’s particularly useful
because it doesn’t overspecify the command by including the details of
the mechanics. All that’s left in this notation is the intent of the
command.

Of course, the mechanics of the command still need to be there for the
command to work. The magic of macros in Clojure is that they make it
easier to bridge the gap from the notation you want to the mechanism
you need. Fortunately, all it takes in this case is a short three line
macro that tells Clojure how to reconstitute a function definition
from our new stack-op notation:

```clojure
(defmacro stack-op [ before after ]
  `(fn [ { [ ~@before & more# ] :stack } ]
     { :stack (concat ~after more# ) } ) )
```

Squint your eyes, and the structure of the original Clojure add
command function should be visible within the macro definition. That’s
because this macro really serves as a kind of IDE snippet hosted by
the compiler, providing blanks to be filled in with the macro
parameters. Multiple calls to a macro are like expanding the same
snippet multiple times with different parameters. The only difference
is that when you expand a snippet within an IDE, it only helps you
when you’re entering the code into the editor; the relationship
between a block of code in the editor and the snippet from which it
came is immediately lost. Macros preserve that relationship, and
thanks to Lisp’s syntax, do so in a way that avoids some of the worst
issues that plague C macros. This gives us both the more ‘intentional’
notation, as well as the ability to later change the underlying
implementation in more profound ways.

Before I close the post, I should mention that there are ways to
approach this type of design in other languages. In C, the
preprocessor provides access to compile-time macro expansion, and for
Java and C#, code generation techniques are well accepted. For
JavaScript, any of the higher level languages that compile into
JavaScript can be viewed as particularly heavy-weight forms of this
technique. Where Lisp and Clojure shine is that they make it easy by
building it into the language directly. This post only scratches the
surface, but the next post will continue the theme by exploring how we
can improve the calculator now that we have a syntax that more
accurately expresses our intent.

