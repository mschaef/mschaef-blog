title: Still not tested... still not working... sort of...
date: 2008-01-21
filename: ./tech/programming/still_not_tested.txt


Another one along the lines of <a 
href="http://www.mschaef.com/blog/tech/programming/not_tested_not_working.html">My 
last post</a>. I tried to compile this source file today, using the 
compiler in my little Lisp:

<pre class="syntax">
(define (values . args) (%panic "roh roh"))

(define (test x) (+ x 1))
</pre>

I got the following result:

<pre class="syntax">
d:\test>vcsh -c test.scm
;;;; VCSH, Debug Build (SCAN 0.99 - Dec 17 2007 16:47:30)

; Info: Loading Internal File: fasl-compiler
; Info: Package 'fasl-compiler' created
; Info: Loading Internal File: fasl-write
; Info: Package 'fasl-write' created
; Info: Loading Internal File: fasl-compiler-run
; Info: Package 'fasl-compiler-run' created
; Info: stack limit disabled!
Fatal Error: roh roh @ (error.cpp:168)
</pre>

Needless to say, fatal errors still aren't any good. However, this one is 
a bit more interesting than a simple type checking problem. The function 
<tt>%panic</tt> is the internal function used to signal fatal errors from 
Lisp code. The first definition above redefines <tt>values</tt>, the 
function to return multiple return values, so that it always panics with a 
fatal error. This is the kind of thing that, if done in a running 
environment, would break things almost immediately.

<br><Br>

But, the compiler is slightly different.... it isolates the program being 
compiled from the compiler itself. This is done to keep redefinitions that 
might break the currently running compiler from doing just that. 
Redefinitions by the compiled program are only supposed to be visible to 
the compiled program. Since the above program never itself invokes 
<tt>values</tt>, it should never hit the call to <tt>%panic</tt>... except 
that it does.

<br><br>

What's happening here lies in the processing of the second definition. The 
definition itself is transformed a couple times by macroexpansion, first 
to this:

<pre class="syntax">
(%define test (named-lambda test (x) (+ x 1)))
</pre>

And then, basically, to this:

<pre class="syntax">
(%define test (%lambda ((name . test) (lambda-list x)) (x) (+ x 1)))
</pre>

The second macroexpansion step is the step that looks for optional 
arguments, and the internal function that parses lambda lists for optional 
arguments returns three values using <tt>values</tt>. This invocation of 
<tt>values</tt> happens in the environment of the program being compiled, 
so it hits the new <tt>%panic</tt>-invoking definition and the whole show 
grinds to a halt. The 'easy' fix, ensuring that macro expansion is 
isolated from potentially harmful redefinitions, won't work. Macro 
expansion has to happen in the user environment, so that macros can see 
function definitions that they might rely upon.

<br><br>

I don't have a unit test for the user/compiler seperation logic, so I 
thought when I started this blog post I was going to say something like: 
'look, something else fundamentally broken, and without a test case'. 
That's interesting, but if you need convincing to write unit tests, you're 
probably already lost. What I actually learned while researching this post 
is a bit more subtle: it's a fundamental problem, but it's more about the 
design than the code itself.  While the design I have for user/compiler 
seperation seems to work most of the time, it's not adequate to solve this 
kind of problem. I'm not yet exactly sure what the solution is, but it 
won't necessarily involve a missing unit test.