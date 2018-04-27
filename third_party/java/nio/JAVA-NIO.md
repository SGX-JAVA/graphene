## Buffers
### Buffer Basics
#### Filling
```
  0   1   2   3   4   5   6   7   8   9  10
---------------------------------------------
| M | e | 1 | 1 |  o|  w|   |   |   |   |   |
| 4D| 65| 6C| 6C| 6F| 77|   |   |   |   |   |
---------------------------------------------
                      |                  | |
           ------------        ----------- --
           |                   |            |
-----    -----               -----        -----
| X |    | 5 |               | 10|        | 10|
-----    -----               -----        ----- 
mark     position            limit        capacity
```

#### Flipping
If we set the position back to 0, the channel will start fetching at the right place, but how will it know when it has reached the end of the data we inserted? This is where the limit attribute comes in. The limit indicates the end of the active buffer content. We need to set the limit to the current position, then reset the position to 0.

But this flipping of buffers from fill to drain state was anticipated by the designers of the API; they provided a handy convenience method to do it for us:
```c
buffer.flip();
```
Following a flip, the buffer of Figure 2-4 would look like Figure 2-5.

```
  0   1   2   3   4   5   6   7   8   9  10
---------------------------------------------
| M | e | 1 | 1 |  o|  w|   |   |   |   |   |
| 4D| 65| 6C| 6C| 6F| 77|   |   |   |   |   |
---------------------------------------------
  |                       |               |
  ----------              ------          ---
           |                   |            |
-----    -----               -----        -----
| X |    | 0 |               | 6 |        | 10|
-----    -----               -----        ----- 
mark     position            limit        capacity
```

#### Draining
The *clear()* method resets a buffer to an empty state. It doesn't change any of the data elements of the buffer but simply sets the limit to the capacity and the position back to 0.

### Summary

- *Byte buffers*

   While buffers can be created for any primitive data type other than boolean, byte buffers have special features not shared by the other buffer types. Only byte buffers can be used with channels (discussed in Chapter 3), and byte buffers offer views of their content in terms of the other data types.

## Selectors
### Selector Basics
#### The Selector, SelectorChannel, and SelectionKey Classes
We've already discussed how to configure and check a channel's blocking mode with the last three methods os *SelectableChannel*, which are listed above. (Refer to Section 3.5.1 for a detailed discussion.) A channel must be placed in nonblocking mode (by calling *configureBlocking(false)*) before it can be registered with a selector.

The choice to put the *register()* method in *SelectableChannel* rather than in *Selector* was somewhat arbitrary. It returns a *SelectionKey* object that encapsulates a relationship between the two objects. The important thing is to remember that the *Selector* object controls the selection process for the channels registered with it.

#### Setting Up Selectors
To set up a *Selector* monitor three *Socket* channels, you'd something like this (refer to Figure 4-2):
```java
Selector selector = Selector.open();

channel1.register (selector, SelectionKey.OP_READ);
channel2.register (selector, SelectionKey.OP_WRITE);
channel3.register (selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);

// Wait up to 10 seconds for a channel to become ready
readyCount = selector.select(10000);
```

Selector objects are instantiated by calling the static factory method *open()*.

### Using Selection Keys
As mentioned earlier, a key represents the registration of a particular channel object with a particular selector object.

A *SelectionKey* object contains two sets encoded as integer bit masks: one for those operations of interest to the channel/selector combination (the *interest set*) and one representing operations the channel is currently ready to perform (the *ready set*). The current interest set can be retrieved from the key object by invoking its *interestOps()* method. Initially, this will be the value passed in when the channel was registered. This interest set will never be changed by the selector, but you can change it by calling *interestOps()* with a new bit mask argument. The interest set can also be modified by reregistering the channel with the selector (which is effectively a roundabout way of invoking *interestOps()*), as described in Section 4.1.2. Changes made to the interest set of a key while a *select()* is in progress on the associated *Selector* will not affect that selection operation. Any changes will be seen on the next invocation of *select()*.

As noted earlier, there are currently four channel operations that can be tested for readiness. You can check these by testing the bit mask as shown in the code above, but the *SelectionKey* class defines four boolean convenience methods to test the bits for you: *isReadable()*, *isWritable()*, *isConnectable()*, and *isAcceptable()*. 

### Using Selectors
#### The Selection Process
Each *Selector* object maintains three sets of keys:

- Registered key set

   The set of currently registered keys associated with the selector.

- Selected key set

   A subset of the registered key sets. Each member of this set is a key whose associated channel was determined by the selector (during a prior selection operation) to be ready for at least one of the operations in the key's interest set. This set is returned by the *selectedKeys()* method (and may be empty).

The core of the *Selector* class is the *selection* process. You've seen several references to it already - now it's time to explain it. Essentially, selectors are a wrapper for a native call to *select()*, *poll()*, or a similar operating system-specific system call. But the Selector does more than a simple pass-through to native code. It applies a specific process on each selection operation. An understanding of this process is essential to properly managing keys and the state information they represent.

A selection operation is performed by a selector when one of the three forms of *select()* is invoked. Whichever is called, the following three steps are performed:

1. The cancelled key set is checked.
2. The operation interest sets of each key in the registered key set are examined. Changes made to the interest sets after they've been examined in this step will not be seen during the remainder of the selection operation.
3. Step 2 can potentially take a long time, especially if the invoking thread sleeps.
4. The value returned by the select operation is the number of keys whose operation ready sets were modified in Step 2, not the total number of channels in the selection key set.

##### Stopping the Selection Process
The last of the `Selector` API methods, *wakeup()*, provides the capability to gracefully break out a thread from a blocked *select()* invocation:

There are three ways to wake up a thread sleeping in *select()*:

- Call wakeup()

   Calling *wakeup()* on a *Selector* object causes the first selection operation on that selector that has not yet returned to return immediately.


```java

package com.ronsoft.books.nio.channels;

import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.channels.Selector;
import java.nio.channels.SelectionKey;
import java.nio.channels.SelectableChannel;

import java.net.Socket;
import java.net.ServerSocket;
import java.net.InetSocketAddress;
import java.util.Iterator;

/**
 * Simple echo-back server which listens for incoming stream connections
 * and echoes back whatever it reads.  A single Selector object is used to
 * listen to the server socket (to accept new connections) and all the
 * active socket channels.
 *
 * @author Ron Hitchens (ron@ronsoft.com)
 * @version $Id: SelectSockets.java,v 1.5 2002/05/20 07:24:29 ron Exp $
 */
public class SelectSockets
{
	public static int PORT_NUMBER = 1234;

	public static void main (String [] argv)
		throws Exception
	{
		new SelectSockets().go (argv);
	}

	public void go (String [] argv)
		throws Exception
	{
		int port = PORT_NUMBER;

		if (argv.length > 0) {	// override default listen port
			port = Integer.parseInt (argv [0]);
		}

		System.out.println ("Listening on port " + port);

		// allocate an unbound server socket channel
		ServerSocketChannel serverChannel = ServerSocketChannel.open();
		// Get the associated ServerSocket to bind it with
		ServerSocket serverSocket = serverChannel.socket();
		// create a new Selector for use below
		Selector selector = Selector.open();

		// set the port the server channel will listen to
		serverSocket.bind (new InetSocketAddress (port));

		// set non-blocking mode for the listening socket
		serverChannel.configureBlocking (false);

		// register the ServerSocketChannel with the Selector
		serverChannel.register (selector, SelectionKey.OP_ACCEPT);

		while (true) {
			// this may block for a long time, upon return the
			// selected set contains keys of the ready channels
			int n = selector.select();

			if (n == 0) {
				continue;	// nothing to do
			}

			// get an iterator over the set of selected keys
			Iterator it = selector.selectedKeys().iterator();

			// look at each key in the selected set
			while (it.hasNext()) {
				SelectionKey key = (SelectionKey) it.next();

				// Is a new connection coming in?
				if (key.isAcceptable()) {
					ServerSocketChannel server =
						(ServerSocketChannel) key.channel();
					SocketChannel channel = server.accept();

					registerChannel (selector, channel,
						SelectionKey.OP_READ);

					sayHello (channel);
				}

				// is there data to read on this channel?
				if (key.isReadable()) {
					readDataFromSocket (key);
				}

				// remove key from selected set, it's been handled
				it.remove();
			}
		}
	}

	// ----------------------------------------------------------

	/**
	 * Register the given channel with the given selector for
	 * the given operations of interest
	 */
	protected void registerChannel (Selector selector,
		SelectableChannel channel, int ops)
		throws Exception
	{
		if (channel == null) {
			return;		// could happen
		}

		// set the new channel non-blocking
		channel.configureBlocking (false);

		// register it with the selector
		channel.register (selector, ops);
	}

	// ----------------------------------------------------------

	// Use the same byte buffer for all channels.  A single thread is
	// servicing all the channels, so no danger of concurrent acccess.
	private ByteBuffer buffer = ByteBuffer.allocateDirect (1024);

	/**
	 * Sample data handler method for a channel with data ready to read.
	 * @param key A SelectionKey object associated with a channel
	 *  determined by the selector to be ready for reading.  If the
	 *  channel returns an EOF condition, it is closed here, which
	 *  automatically invalidates the associated key.  The selector
	 *  will then de-register the channel on the next select call.
	 */
	protected void readDataFromSocket (SelectionKey key)
		throws Exception
	{
		SocketChannel socketChannel = (SocketChannel) key.channel();
		int count;

		buffer.clear();			// make buffer empty

		// loop while data available, channel is non-blocking
		while ((count = socketChannel.read (buffer)) > 0) {
			buffer.flip();		// make buffer readable

			// send the data, don't assume it goes all at once
			while (buffer.hasRemaining()) {
				socketChannel.write (buffer);
			}
			// WARNING: the above loop is evil.  Because
			// it's writing back to the same non-blocking
			// channel it read the data from, this code can
			// potentially spin in a busy loop.  In real life
			// you'd do something more useful than this.

			buffer.clear();		// make buffer empty
		}

		if (count < 0) {
			// close channel on EOF, invalidates the key
			socketChannel.close();
		}
	}

	// ----------------------------------------------------------

	/**
	 * Spew a greeting to the incoming client connection.
	 * @param channel The newly connected SocketChannel to say hello to.
	 */
	private void sayHello (SocketChannel channel)
		throws Exception
	{
		buffer.clear();
		buffer.put ("Hi there!\r\n".getBytes());
		buffer.flip();

		channel.write (buffer);
	}

}
```