title: Thread Local State and its Interaction with Thread Pools
date: 2013-07-02
sponsor: ksm

I recently blogged about solving problems with thread local
storage. In that instance, thread local storage was a way to add a
form of ‘after the fact’ synchronization to an interface that was
initially designed to be called from a single thread.  Thread local
storage is useful for this purpose, because it allows each thread to
easily isolate the data it manipulates from other
threads. Unfortunately, while this is the strength of thread local
storage, it is also the weakness. In modern multi-threaded designs,
threading issues are often abstracted behind thread pools and work
queues. With these abstractions at work, the threads themselves become
an implementation detail, and thread local variables are too low level
to serve some useful scenarios.

One way to address this issue is to use some of the techniques from an
earlier post on dynamic extent. The general gist of the idea is to
provide `Runnable`‘s with way of reconstructing relevant thread local
state at they time they are invoked on a pool thread. This maps well
to the idea of ‘dynamic extent’, as presented in the earlier
post. ‘Establishing the precondition’ is initializing the thread
locals for the run, and ‘Establishing the post condition’ is restoring
their original values. Here’s how it might look in code:

```java
// This is the thread local storage that we want to migrate to worker threads.
ThreadLocal<Object> tls = new ThreadLocal<Object>();
 
Runnable bindToTls(final Runnable runnable)
{
   return new Runnable() {
      // Captured at the time/thread making the initial call to bindToTls
      final Object boundTls = tls.get();
 
      public void run() {
         // Captured at the time the Runnable is invoked
         Object initialTls = tls.get();
 
         try {
            tls.set(boundTls);
 
            runnable.run();
         } finally {
            tls.set(initialTls);
         }
   };
}
```

Calling `bindToTls` on a `Runnable` then gives a new instance of
`Runnable` that remembers the preserved thread local state at the time
of the call to `bindToTls`. The returned `Runnable` can then be provided
to a thread pool, and when it is run, it will remember the thread
local state. At the end of the run, it then ensures that the thread
local is restored to its original value. (Which eliminates the
possibility of certain classes of errors, including potential memory
leaks.)

One caveat to this version of `bindToTls` is that is only migrates the
one thread local variable that it’s been written to migrate. While
this could be made considerably more general, my suggestion is that
it’s usually unnecessary to do so. My current project uses one
instance of `bindToTls` to provide a custom Spring scope. From there,
Spring provides all the generality we need, and the lower level thread
management code is kept contained, which is as it should be.



