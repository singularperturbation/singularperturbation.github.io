---
layout: post
title: What the Fork
tags:
- Nim
- sockets
- POSIX
---

# Exploring fork (2) and sockets in Nim

## Background
Despite having used Linux for a long time, I hadn't really done anything low level
enough to require understanding sockets (especially at the POSIX API level).  Abstruse
as it may seem, there's a lot to be gained by learning about this, as it's an abstraction
that's used [over](http://ruby-doc.org/stdlib-2.3.0/libdoc/socket/rdoc/Socket.html)
and [over](https://github.com/apache/thrift/blob/master/lib/go/thrift/socket.go#L37)
and [over again](https://docs.python.org/3.4/library/socket.html).  While there are
plenty of higher-level interfaces or other abstractions that are important to
network / backend programming{% sidenote side1 Request / response objects from Express, streams,
the actor model, etc.%} the underlying socket primitive is everywhere.

I've just started exploring using sockets in earnest (and some lower-level POSIX
APIs), and decided to do so using [Nim](https://nim-lang.org).  What prompted
this?  I read through Ruslan Spivak's excellent [blog post](https://ruslanspivak.com/lsbaws-part3/)
with examples in Python, started reading through
[Beej's guide to network programming](http://beej.us/guide/bgnet/) and wanted to
see if I could implement the examples in my favorite programming language, Nim.
You can find it [here](https://github.com/singularperturbation/prefork_experiment)

So far, I've gotten about as far as redoing the original Python blog post in Nim,
with some minor modifications.{%sidenote side2 Changing the server to echo the response
as well as be a plaintext server instead of an HTTP server.%}  This post (and the
code) is heavily based on Ruslan's post.

## Why Nim?

Nim is a programming language that compiles to C as an intermediate step before it
becomes an executable.  As such, it has [excellent](http://nim-lang.org/docs/manual.html#types-cstring-type)
interoperability with C and provides [access](http://nim-lang.org/docs/posix.html)
to many POSIX functions.  It's also very pleasant to program in (in my humble
opinion), with a lot of advanced metaprogramming features, expressive syntax, and
a growing and friendly community.  I wanted to use it for exploring POSIX programming
because:

- I knew I would have fun using it
- I thought that a port of Spivak's example would be fairly close to the original
- It can be low level enough that you still get a very real sense of what's going on,
  rather than just how a language's wrapper works.


## POSIX APIs

POSIX{%sidenote side3
[Wikipedia](https://en.wikipedia.org/wiki/POSIX) defines as: The Portable Operating System Interface (POSIX) is a family of standards specified by the IEEE Computer Society for maintaining compatibility between operating systems. POSIX defines the application programming interface (API), along with command line shells and utility interfaces, for software compatibility with variants of Unix and other operating systems.%}
is implemented by BSD, Linux, and Mac OS X.  Back when there were many rival
implementations of Unix in widespread use, there was a need for vendors to provide
a unified API for programmers to use so that programs could run on a variety of
operating systems with little adaptation.

That's... the theory, anyway.  In practice, POSIX API calls usually seem like they
provide only the lowest common denominator of what you want, because that's exactly
how they came to be: an implementation based on what was possible across all of the
competing UNIX operating systems, constrained by the limitations of all.

But they're better than nothing, and at least they're fairly stable.  For today,
we'll be looking at `fork` and functions associated with socket programming:
`bind`, `listen`, `accept`, `send`, `recv`, `signal`, and `socket`.

All of these can be found in section 2 of your trusty man pages if you're on Linux,
i.e.:

```
$ man 2 signal
```

## What are sockets?

Sockets are a way of sending information across the network or to another UNIX process,
while still pretending that you're just writing to a file.{% sidenote side4
In UNIX, everything (more or less) is a file, including the standard input, output,
and error 'pipes' that are given to each process.  That's why you can redirect
output so easily with the `<`, `>`, and `|` shell operators - all they're doing
is substituting one file for another, or writing to one process's standard
input while reading another's standard output.%}

When you create a socket, you can get a file descriptor that can be used for IO
with network devices.  The usual way of doing this is:

1) Use `socket` to create the socket object.  
2) Use `bind` to associate the socket with a given port (for network sockets, which
   is what we'll restrict ourselves to here).  
3) Call `listen` to start listening for incoming connections.  
4) Call `accept`- block until get an incoming connection to the original socket,
 and return a new socket and file descriptor to be used for the incoming connection.

### Using sockets with Nim
All of these are functions defined by the POSIX API, and are callable by C.  In Nim,
we have a choice: I can call the [net](http://nim-lang.org/docs/net.html) or
[nativesockets](http://nim-lang.org/docs/nativesockets.html) modules in the standard
library that provide cross-platform access to sockets, or just call the native
POSIX functions by using the [posix](http://nim-lang.org/docs/posix.html) module.

{%marginnote margin1%}
If I were writing a *real* network program in Nim, I would probably use something
higher-level like [jester](https://github.com/dom96/jester) for HTTP, or
[asyncnet](http://nim-lang.org/docs/asyncnet.html) if doing something that needed
to be at the socket level.  I used Nim for this mostly to learn how the underlying
API worked, explore using `fork`, and do so in a language that's more fun and forgiving
than C.
{% endmarginnote %}

I opted to use the [net module](http://nim-lang.org/docs/net.html) to handle creating
the socket objects, since it provided a low level enough API (access to sockets,
binding to address, etc.) while still giving access to a convenience function
like `readLine`.  Also, mixing a higher-level module like 'net' in the standard
library with native POSIX calls would help show how well Nim can work with C
libraries without any special modifications.

{% highlight nim %}

import net, posix, strutils

let mySocket = newSocket()
# ...
# Preamble removed
# ...

proc main() =

  # Allow socket port to be reused on exit so that we don't have
  #to wait for OS to clean up after daemon quits.
  mySocket.setSockOpt(opt = OptReuseAddr, value = true)

  bindAddr(socket = mySocket, port = Port(9093))
  mySocket.listen()

  let (host, port) = mySocket.getLocalAddr()
  echo "Listening on: $1:$2".format(host,port)

  # Signal handlers
  signal(SIGINT,exitSuccessfully)
  signal(SIGTERM,exitSuccessfully)
  signal(SIGCHLD,handleChildren)

  # ...
  var clientSock = newSocket()
  while true:
    try:
      mySocket.accept(clientSock)
    except OSError:
      # Explicit type conversion needed to OSError since
      # getCurrentException returns the base class.
      let e = (ref OSError) getCurrentException()
      if e.errorCode == EINTR: continue
      else:
        mySocket.close
        raise e

{% endhighlight %}
(This part establishes a listening socket, waits for a connection, and (on
success), puts the result into the socket `clientSock`).  We'll go into the signal
stuff a bit later on.

## What about fork?

`fork` is a system call that allows you to create a new process that inherits all of
the parent process's state (including file descriptors and sockets).  

### Why use it?

When you call `accept`, it blocks the process - all the calling process is doing
is waiting for incoming connections.  As such, one process can't simultaneously
wait for new connections and do any sort of work on a connection that it just
accepted.  This makes for bad latency, especially if each incoming request requires
a lot of work (or needs to wait on a DB connection) in order to respond.

When `fork` is called, the newly created 'child' process can be used to handle
the current request, and the calling process closes the child socket (the child
process will still be listening on it) and loops back around to wait with `accept`
for another incoming connection.  The only indication whether we are in the
parent or child process is `pid`, the return value of `fork`.

{% highlight nim %}
# Still in the loop from above
let pid = fork()
if pid == 0:
  # Child process, so close the listening socket here.
  mySocket.close
  clientSock.handleConnection
  quit(0)
else:
  # Main process, close the client socket.
  clientSock.close
{% endhighlight %}

It's slightly amazing to me that I can call `fork` from the posix module (which
is really just calling the native C `fork` function) and install signal handlers
and my Nim program (that has a runtime, however minimal, a garbage collector,
etc.) will 'just work' regardless.

The `clientSock.handleConnection` part is a call to the `handleConnection`
procedure that reads data from the client socket, writes it back out (with a
prefix) and closes the connection.  

{% highlight nim %}
proc handleConnection(clientSocket: var Socket) =
  # Starting capacity for the string- can grow to be larger if needed
  var inputFromClient: string = newStringOfCap(80)
  clientSocket.readLine(inputFromClient)
  clientSocket.send("SERVED BY Nim: " & inputFromClient & "\n")
  # Simulate load but disconnect first
  clientSocket.close
  discard sleep(3)
{% endhighlight %}

I have it sleep before exiting to simulate the
worker taking some time to process the request.

By using `fork`, we don't have to alter our program design, and (because all of the
state is copied to the child process, not shared) we don't have to worry about
thread safety.  Ha!  Also (in Linux at least), there is an optimization called
copy-on-write which allows the operating system to avoid taking up as much memory
as would be needed for a full copy unless something is changed.

### Why not use it?  For everything?!

- Forking takes time, meaning that you'll be slow(er) to respond to requests.  
- Forking takes memory, which limits the number of 'workers' you can create.

Forking spawns a whole new process (it's not a thread, not a goroutine, and
certainly not a callback in an event loop).  This is beefy, even with the
copy-on-write optimization.

## Signals

In order for things to go smoothly, the parent process has to listen for children
to exit.  Otherwise, they stick around, waiting for a parent's acknowledgement
that will never come.{%sidenote side5 Sad! %} In the land of UNIX, telling a
parent process that its child has completed is accomplished by sending the
`SIGCHLD` signal to the parent process.

By using the `signal` function, we can install a 'signal handler' for a specific
signal for that process - in this case, we use `handleChildren`:

{% highlight nim %}
proc handleChildren(code: cint) {.noconv.} =
  var status: cint = 0
  while true:
    let pid = posix.waitpid(-1.Pid, status, WNOHANG)

    # Return on error or if no child jobs changed state
    if pid == 0 or pid < 0: return
    else: echo "Worker $# is shutting down" % [$pid]
{% endhighlight %}

When the main process gets `SIGCHLD`, it's interrupted and the `handleChildren`
procedure is called - this waits in a loop, using the POSIX `waitpid` call to
find the process ID of any child process has changed state.  It does this in a
loop so that if many children processes have changed state, the parent process
can continue to acknowledge them exiting - a more naive approach can get
overwhelmed if many are exiting at the same time.

The `WNOHANG` option is also from the `posix` module, and tells `waitpid` to return
immediately if there is no change in the status of any child processes.  This way, we
aren't stuck waiting for child processes forever.

(In addition to the signal handler for `SIGCHLD`, there are also signal handlers for
`SIGTERM` and `SIGINT` to allow graceful shutdown if `Ctrl-C` is pressed).

## Interrupts

In the native C version of `accept` and `recv` (getting data from a socket),
there is no exception thrown if there is an interrupt (C can't!) - instead, control
is taken away to the signal handler function, and the call returns an error
code of the special `EINTR` value{%sidenote side7 If you want more information
about EINTR, I found [this](http://250bpm.com/blog:12) to be very informative.%},
which indicates that the function was interrupted.

In Nim, when `accept` or similar is interrupted, this is instead an exception of
type `OSError`, but we can listen for it by catching the exception and checking
whether or not the error code on the exception is equal to `EINTR` from the
posix module; if it is, we just restart the loop and keep listening for new
connections.

---

So, that's a little tour of using Nim with POSIX calls in order to create an
authentic 80s-style forking server.  Hope it was fun!  I tested only on Debian
Linux, but I'd be interested to see if it works on Mac OS X or FreeBSD if
someone tries it out.
