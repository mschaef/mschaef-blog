title: Why is Traditional Java I/O Uninterruptable?
date: 2013-07-22
sponsor: ksm

One of the best things about writing code in the Java ecosystem is
that so much of the underlying platform is open source. This makes it
easy to get good answers to questions about how the platform actually
works. To illustrate, I’ll walk through the JVM code to show why Java
I/O isn’t interruptable. This will explain why threads performing Java
I/O can’t be interrupted.

The core issue with interrupting a Java thread performing I/O is that
the underlying system call is uninterruptable. Let’s see why that is,
for a FileInputStream.

The start of the call stack is in the Java standard class library. In
a traditional source tree, this is located under:
`${JDK_SRC_ROOT}/jdk/src/share/classes/`. In keeping with Java
tradition, the class library code is structured with a directory per
package. Looking at the code for `FileInputStream`, both bulk read
operations delegate to a method `readBytes`:

```java
public int read(byte b[]) throws IOException {
    return readBytes(b, 0, b.length);
}
// ...
public int read(byte b[], int off, int len) throws IOException {
    return readBytes(b, off, len);
}
```

`readBytes`, however, is declared to be native:

```java
private native int readBytes(byte b[], int off, int len) throws IOException;
```

The native code for the class library is located in a parallel
directory structure. `${JDK_SRC_ROOT}/jdk/src/share/native/`.... Like
the Java code, this native code is structured with a file-per-class
and directory-per package. Looking at this code, you can see that the
function name is structured to include the java class name, and the
prototype contains extra parameters that the JVM uses to manage
internal state. env stores a pointer to the current thread’s JVM
environment, and this is an explicit declaration of the usual implicit
Java this argument. The remaining three arguments are the arguments
declared in the original source file.

```c
JNIEXPORT jint JNICALL
Java_java_io_FileInputStream_readBytes(JNIEnv *env, jobject this,
    jbyteArray bytes, jint off, jint len) {
    return readBytes(env, this, bytes, off, len, fis_fd);
}
```

At this point, the class library is calling into another layer of
code, outside the class library. Most of the Java policy surrounding
File I/O is implemented in `readBytes`. (Note particularly the call to
`malloc` in line 23…. traditional File I/O in Java is implemented by
reading into a local buffer, and then copying from the local buffer
into the Java `byte[]` initially passed into int `read(byte buf[])`. This
double-copy is slow, but it also requires heap allocation of a second
read buffer, if the read buffer is large than 8K. The last time I was
reading this code, it was to diagnose an `OutOfMemory` caused by an
over-large buffer size.)

```c
jint
readBytes(JNIEnv *env, jobject this, jbyteArray bytes,
          jint off, jint len, jfieldID fid)
{
    jint nread;
    char stackBuf[BUF_SIZE];
    char *buf = NULL;
    FD fd;
 
    if (IS_NULL(bytes)) {
        JNU_ThrowNullPointerException(env, NULL);
        return -1;
    }
 
    if (outOfBounds(env, off, len, bytes)) {
        JNU_ThrowByName(env, "java/lang/IndexOutOfBoundsException", NULL);
        return -1;
    }
 
    if (len == 0) {
        return 0;
    } else if (len > BUF_SIZE) {
        buf = malloc(len);
        if (buf == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            return 0;
        }
    } else {
        buf = stackBuf;
    }
 
    fd = GET_FD(this, fid);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        nread = -1;
    } else {
        nread = IO_Read(fd, buf, len);
        if (nread > 0) {
            (*env)->SetByteArrayRegion(env, bytes, off, nread, (jbyte *)buf);
        } else if (nread == JVM_IO_ERR) {
            JNU_ThrowIOExceptionWithLastError(env, "Read error");
        } else if (nread == JVM_IO_INTR) {
            JNU_ThrowByName(env, "java/io/InterruptedIOException", NULL);
        } else { /* EOF */
            nread = -1;
        }
    }
 
    if (buf != stackBuf) {
        free(buf);
    }
    return nread;
}
```

The meat of the read is done by the `IO_Read` in line 37. This is
aliased with a preprocessor definition to `JVM_Read`, which is a JVM
primitive operation. JVM primitives are outside the Java class
library, and are in the HotSpot JVM itself. This particular primitive
is defined in `${JDK_SRC_ROOT}/hotspot/src/share/vm/prims/jvm.cpp`. (In
case you’re wondering how I’ve been finding these functions outside of
the class library, I usually use a code text search facility.)

```c
JVM_LEAF(jint, JVM_Read(jint fd, char *buf, jint nbytes))
  JVMWrapper2("JVM_Read (0x%x)", fd);
 
  //%note jvm_r6
  return (jint)os::restartable_read(fd, buf, nbytes);
JVM_END
```

The Java read operation is the point where the code path goes from
code common to all JVM platforms and into OS specific code. For the
Solaris (Unix) code path, the definition looks like this.

```c
size_t os::restartable_read(int fd, void *buf, unsigned int nBytes) {
  INTERRUPTIBLE_RETURN_INT(::read(fd, buf, nBytes), os::Solaris::clear_interrupted);
}
```

Line 2 of this code, finally is the OS system call itself:
::read. However, it’s wrapped in a macro call to
`INTERRUPTIBLE_RETURN_INT`. This macro turns out to be a standard
retry loop for a Unix system call.

```c
#define INTERRUPTIBLE_RETURN_INT(_cmd, _clear) do { \
  int _result; \
  do { \
    INTERRUPTIBLE(_cmd, _result, _clear); \
  } while((_result == OS_ERR) && (errno == EINTR)); \
  return _result; \
} while(false)
```

This macro expansion turns into a loop that will repeatedly issue the
system call as long as it returns `EINTR` - the return code for an
interrupted system call. As long as the system call doesn’t outright
fail, the JVM will keep retrying the read if it’s interrupted. To get
interruptable I/O semantics, you have to call the OS differently.

Slava Pestov has written a nice piece on how [EINTR is used by Unix](http://factor-language.blogspot.com/2010/09/two-things-every-unix-developer-should.html).

Unix’s use of `EINTR` is one of the original aspects of Unix that led
to ‘worse is better’. Other contemporary operating systems of the time
went to greater lengths to handle long running system calls. ITS would
interrupt the system call, and then arrange for it to be restarted
after the interruption, but with parameters that allow it to pick up
where it left off.

See Also: [The original paper on ITS system call restarts](http://fare.tunes.org/tmp/emergent/pclsr.htm).
