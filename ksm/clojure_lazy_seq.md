title: Clojure: lazy-seq and the StackOverflowException
date: 2014-01-28
sponsor: ksm

I’ll get back to rpncalc shortly, but before I do, I wanted to take a
post to talk about a surprising problem I recently had with lazy
sequences. As part of my day job, I am developing a Clojure based
system for accumulating and displaying time series data on a web
page. One of the core algorithms in my implementation is a incremental
merge sort. I have a function that takes two seq’s, both ordered by
time, and produces a lazy result seq with all values from both inputs,
also in time order. Every few seconds, as new input values are read
from their sources, the program uses the ordered merge function to
integrate the new values into a seq that contains a complete history
of all values. It’s a straightforward and flexible design, and
initially, it appeared to work quite well. The problems only started
to arise after several hours of run time: traversing the history list
would then immediately result in stack overflow exceptions.

If you’re familiar with lazy sequences, this may seem like an odd
result. After all, one of the benefits of lazy sequences (aside from
their laziness) is that they can eliminate recursion and reduce
pressure on the stack. Lazy sequences might require more heap
allocation, but they shouldn’t require all that much stack. To explore
this idea a bit further, I’m going to use a simpler example than my
merge function, I’m going to start with a recursive version of map:

```clojure
(defn my-map [ fn xs ]
  (if (empty? xs)
     ()
     (cons (fn (first xs)) (my-map fn (rest xs)))))
```

For simple use cases, `my-map` has the same interface as the built-in
function map:

```clojure
user> (my-map #(+ 1 %) (range 3))
(1 2 3)
user> (map #(+ 1 %) (range 3))
(1 2 3)
```

The limitations of `my-map` start to become apparent for larger
sequences:

```clojure
user> (map #(+ 1 %) (range 10000))
(1 2 3 4 5 6 7 8 9 10 ...)
user> (my-map #(+ 1 %) (range 10000))
StackOverflowError   clojure.lang.Numbers.add (Numbers.java:1685)
```

What’s happening here is that the recursive call to `my-map` causes the
function to allocate a stack frame for each element of the input
sequence. The stack has a fixed size limit, so this places a fixed
limit on the size of the sequence that `my-map` can manipulate. Any
input sequences beyond that length limit will cause the function to
overflow the stack. map gets around this through laziness, which is
something that we can also use:

```clojure
(defn my-map-lazy [ fn xs ]
  (lazy-seq
    (if-let [ xss (seq xs) ]
     (cons (fn (first xs)) (my-map-lazy fn (rest xs)))
     ())))
```

With this definition, all is right with the world:

```clojure
user> (my-map-lazy #(+ 1 %) (range 10000))
(1 2 3 4 5 6 7 8 9 10 ...)
```

While the text of `my-map` and `my-map-lazy` is similar, the functions
internally are quite different in operation. `my-map` completely
computes the result of the mapping before it returns: it eagerly
evaluates and returns the fully calculated result. In contrast,
`my-map-lazy` doesn’t compute any of the mapping before it returns: it
lazily defers the calculation until later and returns a promise to
compute the result later on. The difference may be more clear, looking
at a slightly macro-expanded form of `my-map-lazy`:

```clojure
(defn my-map-lazy [ fn xs ]
  (new clojure.lang.LazySeq
     (fn* []
       (if-let [xss (seq xs)]
          (cons (fn (first xs)) (my-map-lazy fn (rest xs)))
         ()))))
```

The only computations that happen between the entry and exit of
`my-map-lazy` are the allocation of a new lexical closure and then the
instantiation of a new instance of `LazySeq`. While the body `my-map-lazy`
still contains a call to itself, the call doesn’t happen until after
`my-map-lazy` returns and the `LazySeq` invokes the closure. There is no
recursive call, and there is no risk of overflowing the stack. (The
traversal state that was stored on the stack in the recursive version
is stored on the heap in the lazy version.)

So why was my merge sort overflowing the stack? To see why, I’m going
to introduce a new function, using Clojure’s internal map
function. This function serves no purpose, other than to introduce a
layer of laziness. It is only useful for the purposes of this
discussion:

```clojure
(defn lazify [ xs ]
  (map identity xs))
```

Because the evaluation of `map` is lazy, we can predict that what
lazify returns is a LazySeq. This turns out to be true:

```clojure
user> (.getClass [1 2 3 4 5])
clojure.lang.PersistentVector
user> (.getClass (lazify [1 2 3 4 5]))
clojure.lang.LazySeq
```

Calling `lazify` on the result of lazify produces another LazySeq,
distinct from the first.

```clojure
user> (def a (lazify [1 2 3 4 5]))
#'user/a
user> (.getClass a)
clojure.lang.LazySeq
user> (def b (lazify a))
#'user/b
user> (.getClass b)
clojure.lang.LazySeq
user> (identical? a b)
false
```

Due to the way `lazify` is defined, the results of the sequences a and
b are identical to each other - they both result in `(1 2 3 4 5)`.
However, despite the similarity in the results they produce, the
two sequences are distinct and produce their results with different
code paths. Sequence a computes the identity of each element of the
vector `[1 2 3 4 5]` and sequence `b` computes the identity of each
element of sequence `a`. Sequence `b` has to go through sequence a to
get the value from the vector that underlies both. Even in lazy
sequences, this process is still eager, still recursive, and it still
consumes stack.

To confirm this theory, I’ll use another function that applies `lazify`
to a sequence any number of times.

```clojure
(defn lazify-n [ n seq ]
  (loop [n n seq seq]
    (if (> n 0)
      (recur (- n 1) (lazify seq))
      seq))
```

This function builds a tower of lazy sequences n sequences
tall. Computing even the first element of the result sequence,
involves recursively computing every element of each sequence in this
tower, down to the original input seq to `lazify-n`. The depth of the
stack required to maintain this recursive stack is proprortional to
n. High values of n should produce sequences that can’t be traversed
without throwing a stack overflow error. This turns out to be true:

```clojure
user> (lazify-n 1 [1 2 3 4 5])
(1 2 3 4 5)
user> (lazify-n 4000 [1 2 3 4 5])
StackOverflowError   clojure.core/seq (core.clj:133)
```

Going back to my original merge sort stack overflow, it is caused by
the same issue that we see in `lazify-n`. The calls to merge two lists
don’t merge the lists at the time of the call. Rather, the calls
produce promises to merge the lists at some later point in time. Every
call to merge increases the number of lists to merge, and increases
the depth of the stack that the merge process needs to use during the
merge operation. After a while, the number of lists to be merged gets
high enough that they can’t be merged without overflowing the
stack. This is the cause of my initial stack overflow.

So what’s the solution? One easy solution is to give up some amount of
laziness.

```clojure
(defn lazify-n! [ n seq ]
  (loop [n n seq seq]
    (if (> n 0)
      (recur (- n 1) (doall (lazify seq)))
      seq))
```

The only difference between this new version of `lazify-n` and the
previous is the call to doall on the fourth line. What `doall` does is
force the full evaluation of a lazy sequence. So, while `lazify-n!`
still produces an n high tower of lazy sequences, they’re all been
fully traversed. Because `LazySeq` caches values the first time it’s
traversed traversal, there’s no need to recursively call up the tower
of sequences to traverse the final output sequence. This gives up some
laziness, but it avoids both stack overflow issues we’ve discussed in
this blog post: the overflow on long input sequence lengths and the
overflow on deeply nested lazy sequences. The cost (there’s always a
cost) is that this requires more heap storage than many alternative
structures.
