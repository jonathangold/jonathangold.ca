---
title: "Go Netlink and Select"
date: 2018-05-14T11:59:07-04:00
draft: false
---

Go is a wonderful programming language, nestled comfortably in between
low-level languages like C or Rust, and higher level languages like Python or
Ruby. The convenience one gains by doing systems programming in Go, thanks to
perks like syntactic simplicity and first-class concurrency, is offset by
repetitive error handling, lack of direct memory access, and a few glaring
omissions from the standard library (generics and binary literals come to mind.)

This article will describe the process of writing the network resource for
[mgmt](http://github.com/purpleidea/mgmt), a tool for next generation
distributed, event-driven, parallel config management. The network resource
is tasked with applying the prescribed network configuration, monitoring the
defined device, and correcting its state if it diverges from the configuration.
In simpler terms, it sets up a network device and makes sure it stays set up
exactly how you want.

Unfortunately Go's net library is only useful when it comes to checking the
network state, and lacks the ability to apply changes to an interface, so it
was necessary to venture outside the comfort of the standard library and employ
a more powerful solution. Enter Netlink.

There are several libraries available to interface with kernel netlink sockets
at various levels of abstraction, but none of them are without problems. For
example, `vishvananda/netlink` has a "subscribe" function, which is supposed
to subscribe to events on a netlink socket. Unfortunately the package fails to
clean up spawned goroutines and their api design leads to a race, where data
and errors may not be serialized in the correct order. Another package,
`mdlayher/netlink`, gets one step closer to a correct implementation but the
`conn.Receive()` method will block indefinitely until a message arrives on the
socket. Given these shortcomings, it was necessary to roll my own netlink
socket listener. I was however, able to use some helper functions from
`vishvananda/netlink` when it came time to apply changes to the interface.

Enter `unix.Select()`. Systems programmers should already be familiar with
`select(2)`, but for those that aren't, here's a quote from the man page:

```
select() and pselect() allow a program to monitor multiple file
descriptors, waiting until one or more of the file descriptors become
"ready" for some class of I/O operation (e.g., input possible).  A
file descriptor is considered ready if it is possible to perform a
corresponding I/O operation (e.g., read(2) without blocking, or a
sufficiently small write(2)).
```

On linux, everything is a file, and that includes sockets (netlink or
otherwise.) So, we can use select to watch the socket, and return when an event
occurs. Simple enough. In fact, this is exactly what `mdlayher/netlink` does.
The problem is that when you want to exit, the select will block indefinitely.
On some systems, select will return when the file descriptor is destroyed, but
that is undefined behavior, and doesn't happen on many linux systems.

The solution is the Unix `Pipe` pattern, where we employ a second file
descriptor to signal select when it's time to close. The pipe in this case, is
an ordinary Unix socket, stored in the temp filesystem. With both file
descriptors in place, we can call select, using `golang.org/x/sys/unix`'s
`Select()` function, and pass it a set containing file descriptors for both
netlink and pipe sockets. Then, to unblock `select()` when we want to shut
down, all we have to do is close the netlink socket, and then send an empty
message to the pipe socket. This will cause select to return and our program
can exit cleanly. Here's what that looks like:

```
// shutdown closes the event file descriptor and unblocks receive by sending
// a message to the pipe file descriptor. It must be called before close, and
// should only be called once.
func (obj *socketSet) shutdown() error {
	// close the event socket so no more events are produced
	if err := unix.Close(obj.fdEvents); err != nil {
		return err
	}
	// send a message to the pipe to unblock select
	return unix.Sendto(obj.fdPipe, nil, 0, &unix.SockaddrUnix{
		Name: path.Join(obj.pipeFile),
	})
}

// close closes the pipe file descriptor. It must only be called after
// shutdown has closed fdEvents, and unblocked receive. It should only be
// called once.
func (obj *socketSet) close() error {
	return unix.Close(obj.fdPipe)
}

// receive waits for bytes from fdEvents and parses them into a slice of
// netlink messages. It will block until an event is produced, or shutdown
// is called.
func (obj *socketSet) receive() ([]syscall.NetlinkMessage, error) {
	// Select will return when any fd in fdSet (fdEvents and fdPipe) is ready
	// to read.
	_, err := unix.Select(obj.nfd(), obj.fdSet(), nil, nil, nil)
	if err != nil {
		// if a system interrupt is caught
		if err == unix.EINTR { // signal interrupt
			return nil, nil
		}
		return nil, errwrap.Wrapf(err, "error selecting on fd")
	}
	// receive the message from the netlink socket into b
	b := make([]byte, os.Getpagesize())
	n, _, err := unix.Recvfrom(obj.fdEvents, b, unix.MSG_DONTWAIT) // non-blocking receive
	if err != nil {
		// if fdEvents is closed
		if err == unix.EBADF { // bad file descriptor
			return nil, nil
		}
		return nil, errwrap.Wrapf(err, "error receiving messages")
	}
	// if we didn't get enough bytes for a header, something went wrong
	if n < unix.NLMSG_HDRLEN {
		return nil, fmt.Errorf("received short header")
	}
	b = b[:n] // truncate b to message length
	// use syscall to parse, as func does not exist in x/sys/unix
	return syscall.ParseNetlinkMessage(b)
}
```

If you are interested in the details of my implementation, check out [the source](https://github.com/jonathangold/mgmt/blob/master/resources/net.go) for context
and if you have any questions, send me an [email](mailto:info@jonathangold.ca), or ping me on twitter [@JonWritesCode](https://twitter.com/JonWritesCode).
