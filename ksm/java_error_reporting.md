title: Details in Java Code: Error Reporting and Loop Control Variables
date: 2014-03-31
sponsor: ksm
tags: ksm java

Sometimes, it’s easy to focus so much on the architecture of a system
that the details of its implementation get lost. While it’s true that
inattention to architectural concerns can cause a system to fail, it’s
also true that poor attention to the details can undermine even the
best overall system design. This post covers a few minor details of
code structure that I’ve found to be useful in my work:

It’s a small thing, but one of my favorite utility methods is a short
way to throw run-time exceptions.

```java
public static void FAIL(String message)
{
    throw new RuntimeException(message);
}
```

Defining this method accomplishes a few useful goals. The first is
that (with an import static) it makes it possible to throw a
RuntimeException with 22 fewer characters of source text per call
site. If you’re writing usefully descriptive error messages (which you
should be), this can significantly improve the readability of the
code. The text `FAIL` tends to stand out in source code listings, and
bringing the error message closer to the left margin of the source
text makes it more obvious. The symbol FAIL is also easy to identify
with tools like grep, ack, and M-x occur.

To handle re-throw scenarios, it's also useful to have another definition that lets you specify a cause for the failure.

```java
public static void FAIL(String message, Throwable cause)
{
    throw new RuntimeException(message, cause);
}
```

Related to this is a useful naming convention for loop control
variables. Thanks in large part to FORTRAN, and its mathematical
heritage, it's very common to use the names i, j, and k for loop
control variables. These names aren't very descriptive, but they're
short and for small loop bodies, there's usually enough context that a
longer name would be superfluous. (If your loop spans pages of text,
you should use a more descriptive variable name... but first, you
should try to break up your loop into sensible, testable functions.)
One technique I've found useful for making loop control variables more
obvious (and searchable) without going to fully descriptive variable
names is to double up the letters, giving `ii`, `jj`, and `kk`.

These are both small changes, but they both can improve the
readability of the code. Try them out and see if you like them. If you
disagree that they are improvements, it's easy to switch back.
