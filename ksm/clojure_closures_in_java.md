title: Clojure closures in Java
date: 2013-08-19

As part of a team conversation this morning, I worked up a quick Java
translation of some more-interesting-than-it-looks Clojure code. It’s
a good example of how lexical closures map to objects.

The code we started out with was this:

```clojure
(defn make-adder [x]
  (let [y x]
    (fn [z] (+ y z))))
 
(def add2 (make-adder 2))
 
(add2 4)
```

What this code does is define and return a new type of function that
adds values to the constant `x`. In Java, it looks like this:

```clojure
// Not needed in Clojure... the Clojure runtime implicitly gives types
// that look similar to this to all Clojure functions.
interface Function
{
   Integer invoke(Integer z);
}
 
public class Foo
{
   // (defn make-adder [x]
   //   (let [y x]
   //     (fn [z] (+ y z))))
   public static Function makeAdder(Integer x)
   {
       final Integer y = x;
 
       return new Function() {
           public Integer invoke(Integer z) {
               return z + y;
           }
       }
   }
 
   public static int main(String[] args)
   {
       // (def add2 (make-adder 2))
       Function add2 = Foo.makeAdder(2);
 
       // (add2 4)
       System.out.println(add2.invoke(4));
   }
}
```

Five lines of Clojure code translate into 30 lines of Java: a function
definition becomes a class definition, with state.

This idiom is powerful, but coming from Java, the power is hidden
behind unusual syntax and terse notation. There are good reasons for
both the syntax and the notation, but if you’re not used to either,
it’s very easy to look at a page of Clojure code and be completely
lost. Getting past this barrier by developing an intuitive feel for
the language is a major challenge faced by teams transitioning to
Clojure and Scala. One of the goals I have for my posts in this blog
is help fellow developers along the way. It should be a fun ride.
