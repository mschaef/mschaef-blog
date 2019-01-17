title: Threads, Line-Synchronization, and OutputStream: ThreadLocals and Java I/O
date: 2013-07-27
sponsor: ksm

The other day, our team ran across an interesting design problem
related to Java synchronization. Without getting into the details of
why this problem came up, the gist of it was that we had to somehow to
write a synchronization wrapper around `OutputStream`. (Well,
technically, it was a `BufferedWriter`, but the issue is the same.) This
wrapper needed to correctly allows multiple unsynchronized writer
threads to write to an underlying writer, each thread atomically
writing lines of text. We wound up managing to avoid having to do this
by changing our interface, but the solution to the `OutputStream`
problem still provides an interesting look at a lesser-known aspect of
the Java Runtime: `ThreadLocal` variables.


To illustrate the problem, consider this simpler version of the
`OutputStream` interface. (Unlike the full version, this interface
eliminates the two bulk-write forms of the write method, which don’t
matter for this conversation.)

```java
interface SimpleOutputStream {
    void write(int byte);
}
```

The simplest way to writing a synchronized wrapper for this interface
is to wrap each call to the underlying implementation method in a
synchronized block. This wrapper then guarantees that each thread
entering write will have exclusive and atomic access to write its byte
to the underlying implementation.

```java
public void write(int byte)
{
    synchronized(underlying)
    {
        underlying.write(byte);
    }
}
```

The difficulty with this approach is that the locking is too fine
grained to produce a coherent stream of output bytes on the underlying
writer. A context switch in the middle of a line of output text will
cause that line to contain bytes from two source threads, with no way
to distinguish one thread’s data from another. The only thing the
locking has provided is a guarantee that we won’t have multiple
threads reentrantly calling into the underlying stream. What we need
is a way for our wrapper to buffer the lines of text on a per-thread
basis, and then write full lines within a `synchronized` block, and this
is where `ThreadLocal` comes into play.

An instance of `ThreadLocal` is exactly what it sounds like: a container
for a value that is local to a thread. Each thread that gets a value
from an instance of a `ThreadLocal` gets its own, unique copy of that
value. Ignoring how exactly it might work, this is the abstraction
that will enable our implementation of `OutputStream` to buffer lines of
text from each writer thread, prior to writing them out. The key is is
in the specific use of the thread local.

```java
ThreadLocal threadBuf = new ThreadLocal() {
    protected StringBuffer initialValue() {
        return new StringBuffer();
    }
};
```

This code allocates an instance of a local derivation of `ThreadLocal`
specialized to hold a `StringBuffer`. The overrided initialValue method
determines the initial value of the `ThreadLocal` for each thread that
retrieves a value – it is called once for each thread. In this
specific case, it allocates a new `StringBuffer` for each thread that
requests a value from `threadBuf`. This provides us with a line buffer
per thread. From here on out, the implementation is largely connecting
the dots.

First, we need an implementation of write that buffers into `threadBuf`:

```java
public void write(int b)
      throws IOException
{
    threadBuf.get().append((char)b);
 
    if (b == (int)'\n')
        flush();
}
```

The `get` method pulls the thread local buffer out of
`threadBuf`. Note that because this buffer is thread local, it is not
shared, so we don’t need to synchronize on it.

Second, we need an implementation of flush that atomically writes the
current thread line buffer to the underlying output:

```java
protected void flush()
    throws IOException
{
    synchronized(underlying) {
        underlying.write(threadBuf.get().toString().getBytes());
 
        threadBuf.get().setLength(0);
    }
}
```

Here we do need synchronization, because `underlying` is shared between
threads, even if the `threadBuf` is not.

While there are still a few implementation details to worry about,
this provides the bulk of the functionality we were originally looking
for. Multiple threads can write into an output stream wrapped in this
way, and the output will be synchronized on a per-line basis. If you’d
like to experiment with the idea a bit, or read through an
implementation, I’ve put code on [github](https://github.com/mschaef/tls-writer).
This sample project contains a synchronized output stream, as well as
a test program that launches multiple threads, all writing to
`System.out`, via our synchronized. A command line flag lets you turn
the wrapper off to see the ill effects of finer grained
synchronization.
