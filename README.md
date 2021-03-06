# Naga
## Simplified Java NIO asynchronous sockets
This library was not created by me :
Automatically exported from code.google.com/p/naga

Naga is not a framework.

Naga aims to be a very small NIO library that provides a handful of java classes to wrap the usual Socket and ServerSocket with asynchronous NIO counterparts (similar to NIO2 planned for Java 1.7).

All of this is driven from a single thread, making it useful for both client (e.g. allowing I/O to be done in the AWT-thread without any need for threads) and server programming (1 thread for all connections instead of 2 threads/connection).

Internally Naga is a straightforward NIO implementation without any threads or event-queues thrown in, it is "just the NIO-stuff", to let you build things on top of it.

Naga contains the code needed to get NIO up and running without having to code partially read buffers and setting various selection key flags.


Creating a simple echo server.
Introduction
This examples shows how to create a simple echo server using Naga.

~~Note that this example is slightly out of date for 1.1~~. Fixed example for version 3.0 , changed the example to match android 4.0+

##Details
### Creating a NIOService
The first thing we want to do is to create a new NIOService object. This object will hold our Selector and is our factory for creating Sockets.
```
NIOService service = new NIOService();
```
At this point nothing is running yet.

To run a pass and execute all NIO reads and writes queued up, we would write:
```
service.selectNonBlocking();
```
You can also make non-blocking calls, which is useful for putting the selector in its own thread.

This code would start a thread, which would then run all IO for the NIOService until the service shuts down. Note that you should never call the select calls from more than one thread at a time.
```
new Thread()
{
  public void run()
  {
    try
    {
      while (true) { service.selectNonBlocking(); }
    }
    catch (Exception e) { }
  }
}.start();
```
####Opening a server socket
Now we want to open a server socket on a port to receive connection. We do this with the openServerSocket(port) call.

This code:
```
NIOServerSocket socket = service.openServerSocket(3366);
```

Opens a server socket at port 3366. What you get back is a NIOServerSocket, which is a proxy for the real ServerSocket we just created.

We want to add some behaviour to the socket, because right now it accepts no connections.

First thing is that we want to set the observer for the socket. This will give us notifications when when connections fail, succeed and when the server socket itself goes down.

We are only interested in the connects, so let us use the ServerSocketObserverAdapter to only handle connects.

```
socket.listen(new ServerSocketObserverAdapter()
{
  public void newConnection(NIOSocket nioSocket)
  {
    System.out.println("Connect!");
  }
});
```
Here our server would print "Connect!" every time a connection was accepted on 3366. We will want to change this code later to actually do some echoing.

The second thing we need to do is to set an acceptor for the socket. This acceptor is given an IP and can decide if the connection should be accepted or denied. In our echo server case, we want all calls to be accepted, so we use a predefined acceptor for that.

socket.setConnectionAcceptor(ConnectionAcceptor.ALLOW);
At this point we could start reading on the NIOService like this:

```
while (true) { service.selectNonBlocking(); }
```

This server would simply accept connections and write "Connect!" when a connection was made.

How do we get the echo-part of the server then?

Customizing our incoming socket
Well, remember how the newConnection method gave a NIOSocket? We use that socket to register an observer and a reader:
```
public void newConnection(NIOSocket nioSocket)
{
  nioSocket.listen(new SocketObserverAdapter()
  {
    public void packetReceived(NIOSocket socket, byte[] packet)
    {
      socket.write(packet);
    }
  });
}
```
####Let us review this code.

Start listening to the socket
The next line:
```
nioSocket.listen(new SocketObserverAdapter());
```

Registers the observer for the socket and starts listening for incoming packets. Just like in the server socket case, this will run callbacks for various events that happen to the socket. Connects, disconnects and reads.

Obviously the reads are exactly what we want to hook into.

Writing to a socket
Here is the code that reacts to a new packet:
```
public void notifyReadPacket(NIOSocket socket, byte[] packet)
{
  socket.write(packet);
}
```

We get a packet, in this case we are relying on the underlying RawPacketReader to give us the last bunch of bytes we received from the client (read more about PacketReaders) which we send back using NIOSocket's write method, which is fully asynchronous.

It is possible to register a PacketWriter that converts our packet before we send it, but for now we are happy to to get the default behaviour which is a writer that dispatches the raw bytes to the socket.

Code Listing
Now we have all the code in place. Here is the full listing - run using "java EchoServer <port>"

```
import naga.*;

import java.io.IOException;

public class EchoServer
{
  public static void main(String... args)
  {
    int port = Integer.parseInt(args[0]);
    try
    {
      // Create the NIO service.
      NIOService service = new NIOService();

      // Open a server socket.
      NIOServerSocket socket = service.openServerSocket(port);
      
      // Set our server socket observer to listen to the server socket.
      socket.listen(new ServerSocketObserverAdapter()
      { 
        public void newConnection(NIOSocket nioSocket)
        {

          // Set our socket observer to listen to the new socket.
          nioSocket.listen(new SocketObserverAdapter()
          {
            public void notifyReadPacket(NIOSocket socket, byte[] packet)
            {
              // Write the bytes back to the client.
              socket.write(packet);
            }
          });
        }
      });

      // Set server socket accept policy to always accept new clients.
      socket.setConnectionAcceptor(ConnectionAcceptor.ALLOW);

      // Handle IO until we quit the program.
      while (true)
      {
        service.selectNonBlocking();
      }
    }
    catch (IOException e)
    {
      // Ignore any IOExceptions in this simple implementation.
    }
  }
}
```
