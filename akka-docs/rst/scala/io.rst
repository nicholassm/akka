.. _io-scala:

I/O (Scala)
===========

Introduction
------------

The ``akka.io`` package has been developed in collaboration between the Akka
and `spray.io`_ teams. Its design incorporates the experiences with the
``spray-io`` module along with improvements that were jointly developed for
more general consumption as an actor-based service.

This documentation is in progress and some sections may be incomplete. More will be coming.

.. toctree::

  io-old

.. note::
  The old I/O implementation has been deprecated and its documentation has been moved: :ref:`io-scala-old`

Terminology, Concepts
---------------------
The I/O API is completely actor based, meaning that all operations are implemented as message passing instead of
direct method calls. Every I/O driver (TCP, UDP) has a special actor, called *manager* that serves
as the entry point for the API. The manager is accessible through an extension, for example the following code
looks up the TCP manager and returns its ``ActorRef``:

.. code-block:: scala

  val tcpManager = IO(Tcp)

For various I/O commands the manager instantiates worker actors that will expose themselves to the user of the
API by replying to the command. For example after a ``Connect`` command sent to the TCP manager the manager creates
an actor representing the TCP connection. All operations related to the given TCP connections can be invoked by sending
messages to the connection actor which announces itself by sending a ``Connected`` message.

DeathWatch and Resource Management
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Worker actors usually need a user-side counterpart actor listening for events (such events could be inbound connections,
incoming bytes or acknowledgements for writes). These worker actors *watch* their listener counterparts, therefore the
resources assigned to them are automatically released when the listener stops. This design makes the API more robust
against resource leaks.

Thanks to the completely actor based approach of the I/O API the opposite direction works as well: a user actor
responsible for handling a connection might watch the connection actor to be notified if it unexpectedly terminates.

Write models (Ack, Nack)
^^^^^^^^^^^^^^^^^^^^^^^^

Basically all of the I/O devices have a maximum throughput which limits the frequency and size of writes. When an
application tries to push more data then a device can handle, the driver has to buffer all bytes that the device has
not yet been able to write. With this approach it is possible to handle short bursts of intensive writes --- but no buffer is infinite.
Therefore, the driver has to notify the writer (a user-side actor) either that no further writes are possible, or by
explicitly notifying it when the next chunk is possible to be written or buffered.

Both of these models are available in the TCP and UDP implementations of Akka I/O. Ack based flow control can be enabled
by providing an ack object in the write message (``Write`` in the case of TCP and ``Send`` for UDP) that will be used by
the worker to notify the writer about the success.

If a write (or any other command) fails, the driver notifies the commander with a special message (``CommandFailed`` in
the case of UDP and TCP). This message also serves as a means to notify the writer of a failed write. Please note, that
in a Nack based flow-control setting the writer has to buffer some of the writes as the failure notification for a
write ``W1`` might arrive after additional write commands ``W2`` ``W3`` has been sent.

.. warning::
  An acknowledged write does not mean acknowledged delivery or storage. The Ack/Nack
  protocol described here is a means of flow control not error handling: receiving an Ack for a write signals that the
  I/O driver is ready to accept a new one.

ByteString
^^^^^^^^^^

A primary goal of Akka's IO support is to only communicate between actors with immutable objects. When dealing with network I/O on the jvm ``Array[Byte]`` and ``ByteBuffer`` are commonly used to represent collections of ``Byte``\s, but they are mutable. Scala's collection library also lacks a suitably efficient immutable collection for ``Byte``\s. Being able to safely and efficiently move ``Byte``\s around is very important for this I/O support, so ``ByteString`` was developed.

``ByteString`` is a `Rope-like <http://en.wikipedia.org/wiki/Rope_(computer_science)>`_ data structure that is immutable and efficient. When 2 ``ByteString``\s are concatenated together they are both stored within the resulting ``ByteString`` instead of copying both to a new ``Array``. Operations such as ``drop`` and ``take`` return ``ByteString``\s that still reference the original ``Array``, but just change the offset and length that is visible. Great care has also been taken to make sure that the internal ``Array`` cannot be modified. Whenever a potentially unsafe ``Array`` is used to create a new ``ByteString`` a defensive copy is created. If you require a ``ByteString`` that only blocks a much memory as necessary for it's content, use the ``compact`` method to get a ``CompactByteString`` instance. If the ``ByteString`` represented only a slice of the original array, this will result in copying all bytes in that slice.

``ByteString`` inherits all methods from ``IndexedSeq``, and it also has some new ones. For more information, look up the ``akka.util.ByteString`` class and it's companion object in the ScalaDoc.

``ByteString`` also comes with it's own optimized builder and iterator classes ``ByteStringBuilder`` and ``ByteIterator`` which provides special features in addition to the standard builder / iterator methods:

Compatibility with java.io
..........................

A ``ByteStringBuilder`` can be wrapped in a `java.io.OutputStream` via the ``asOutputStream`` method. Likewise, ``ByteIterator`` can we wrapped in a ``java.io.InputStream`` via ``asInputStream``. Using these, ``akka.io`` applications can integrate legacy code based on ``java.io`` streams.

Encoding and decoding of binary data
....................................

``ByteStringBuilder`` and ``ByteIterator`` support encoding and decoding of binary data. As an example, consider a stream of binary data frames with the following format:

.. code-block:: text

  frameLen: Int
  n: Int
  m: Int
  n times {
    a: Short
    b: Long
  }
  data: m times Double

In this example, the data is to be stored in arrays of ``a``, ``b`` and ``data``.

Decoding of such frames can be efficiently implemented in the following fashion:

.. includecode:: code/docs/io/BinaryCoding.scala
   :include: decoding

This implementation naturally follows the example data format. In a true Scala application, one might, of course, want use specialized immutable Short/Long/Double containers instead of mutable Arrays.

After extracting data from a ``ByteIterator``, the remaining content can also be turned back into a ``ByteString`` using the ``toSeq`` method

.. includecode:: code/docs/io/BinaryCoding.scala
   :include: rest-to-seq

with no copying from bytes to rest involved. In general, conversions from ByteString to ByteIterator and vice versa are O(1) for non-chunked ByteStrings and (at worst) O(nChunks) for chunked ByteStrings.

Encoding of data also is very natural, using ``ByteStringBuilder``

.. includecode:: code/docs/io/BinaryCoding.scala
   :include: encoding

Using TCP
---------

As with all of the Akka I/O APIs, everything starts with acquiring a reference to the appropriate manager:

.. code-block:: scala

  import akka.io.IO
  import akka.io.Tcp
  val tcpManager = IO(Tcp)

This is an actor that handles the underlying low level I/O resources (Selectors, channels) and instantiates workers for
specific tasks, like listening to incoming connections.

Connecting
^^^^^^^^^^

The first step of connecting to a remote address is sending a ``Connect`` message to the TCP manager:

.. code-block:: scala

  import akka.io.Tcp._
  IO(Tcp) ! Connect(remoteSocketAddress)
  // It is also possible to set various socket options or specify a local address:
  IO(Tcp) ! Connect(remoteSocketAddress, Some(localSocketAddress), List(SO.KeepAlive(true)))

After issuing the Connect command the TCP manager spawns a worker actor that will handle commands related to the
connection. This worker actor will reveal itself by replying with a ``Connected`` message to the actor who sent the
``Connect`` command.

.. code-block:: scala

  case Connected(remoteAddress, localAddress) =>
    connectionActor = sender

At this point, there is still no listener associated with the connection. To finish the connection setup a ``Register``
has to be sent to the connection actor with the listener ``ActorRef`` as a parameter.

.. code-block:: scala

  connectionActor ! Register(listener)

After registration, the listener actor provided in the ``listener`` parameter will be watched by the connection actor.
If the listener stops, the connection is closed, and all resources allocated for the connection released. During the
lifetime the listener may receive various event notifications:

.. code-block:: scala

  case Received(dataByteString) => // handle incoming chunk of data
  case CommandFailed(cmd)       => // handle failure of command: cmd
  case _: ConnectionClosed      => // handle closed connections

The last line handles all connection close events in the same way. It is possible to listen for more fine-grained
connection events, see the appropriate section below.


Accepting connections
^^^^^^^^^^^^^^^^^^^^^

To create a TCP server and listen for inbound connection, a ``Bind`` command has to be sent to the TCP manager:

.. code-block:: scala

  import akka.io.IO
  import akka.io.Tcp
  IO(Tcp) ! Bind(handler, localAddress)

The actor sending the ``Bind`` message will receive a ``Bound`` message signalling that the server is ready to accept
incoming connections. Accepting connections is very similar to the last two steps of opening outbound connections: when
an incoming connection is established, the actor provided in ``handler`` will receive a ``Connected`` message whose
sender is the connection actor:

.. code-block:: scala

  case Connected(remoteAddress, localAddress) =>
    connectionActor = sender

At this point, there is still no listener associated with the connection. To finish the connection setup a ``Register``
has to be sent to the connection actor with the listener ``ActorRef`` as a parameter.

.. code-block:: scala

  connectionActor ! Register(listener)

After registration, the listener actor provided in the ``listener`` parameter will be watched by the connection actor.
If the listener stops, the connection is closed, and all resources allocated for the connection released. During the
lifetime the listener will receive various event notifications in the same way as we has seen in the outbound
connection case.

Closing connections
^^^^^^^^^^^^^^^^^^^

A connection can be closed by sending one of the commands ``Close``, ``ConfirmedClose`` or ``Abort`` to the connection
actor.

``Close`` will close the connection by sending a ``FIN`` message, but without waiting for confirmation from
the remote endpoint. Pending writes will be flushed. If the close is successful, the listener will be notified with
``Closed``

``ConfirmedClose`` will close the sending direction of the connection by sending a ``FIN`` message, but receives
will continue until the remote endpoint closes the connection, too. Pending writes will be flushed. If the close is
successful, the listener will be notified with ``ConfirmedClosed``

``Abort`` will immediately terminate the connection by sending a ``RST`` message to the remote endpoint. Pending
writes will be not flushed. If the close is successful, the listener will be notified with ``Aborted``

``PeerClosed`` will be sent to the listener if the connection has been closed by the remote endpoint.

``ErrorClosed`` will be sent to the listener whenever an error happened that forced the connection to be closed.

All close notifications are subclasses of ``ConnectionClosed`` so listeners who do not need fine-grained close events
may handle all close events in the same way.

Throttling Reads and Writes
^^^^^^^^^^^^^^^^^^^^^^^^^^^

*This section is not yet ready. More coming soon*

Using UDP
---------

UDP support comes in two flavors: connectionless, and connection based:

.. code-block:: scala

  import akka.io.IO
  import akka.io.UdpFF
  val connectionLessUdp = IO(UdpFF)
  // ... or ...
  import akka.io.UdpConn
  val connectionBasedUdp = IO(UdpConn)

UDP servers can be only implemented by the connectionless API, but clients can use both.

Connectionless UDP
^^^^^^^^^^^^^^^^^^

Simple Send
............

To simply send a UDP datagram without listening to an answer one needs to send the ``SimpleSender`` command to the
manager:

.. code-block:: scala

  IO(UdpFF) ! SimpleSender
  // or with socket options:
  import akka.io.Udp._
  IO(UdpFF) ! SimpleSender(List(SO.Broadcast(true)))

The manager will create a worker for sending, and the worker will reply with a ``SimpleSendReady`` message:

.. code-block:: scala

  case SimpleSendReady =>
    simpleSender = sender

After saving the sender of the ``SimpleSendReady`` message it is possible to send out UDP datagrams with a simple
message send:

.. code-block:: scala

  simpleSender ! Send(data, serverAddress)


Bind (and Send)
...............

To listen for UDP datagrams arriving on a given port, the ``Bind`` command has to be sent to the connectionless UDP
manager

.. code-block:: scala

  IO(UdpFF) ! Bind(handler, localAddress)

After the bind succeeds, the sender of the ``Bind`` command will be notified with a ``Bound`` message. The sender of
this message is the worker for the UDP channel bound to the local address.

.. code-block:: scala

  case Bound =>
    udpWorker = sender // Save the worker ref for later use

The actor passed in the ``handler`` parameter will receive inbound UDP datagrams sent to the bound address:

.. code-block:: scala

  case Received(dataByteString, remoteAddress) => // Do something with the data

The ``Received`` message contains the payload of the datagram and the address of the sender.

It is also possible to send UDP datagrams using the ``ActorRef`` of the worker saved in ``udpWorker``:

.. code-block:: scala

 udpWorker ! Send(data, serverAddress)

.. note::
  The difference between using a bound UDP worker to send instead of a simple-send worker is that in the former case
  the sender field of the UDP datagram will be the bound local address, while in the latter it will be an undetermined
  ephemeral port.

Connection based UDP
^^^^^^^^^^^^^^^^^^^^

The service provided by the connection based UDP API is similar to the bind-and-send service we have seen earlier, but
the main difference is that a connection is only able to send to the remoteAddress it was connected to, and will
receive datagrams only from that address.

Connecting is similar to what we have seen in the previous section:

.. code-block:: scala

  IO(UdpConn) ! Connect(handler, remoteAddress)
  // or, with more options:
  IO(UdpConn) ! Connect(handler, Some(localAddress), remoteAddress, List(SO.Broadcast(true)))

After the connect succeeds, the sender of the ``Connect`` command will be notified with a ``Connected`` message. The sender of
this message is the worker for the UDP connection.

.. code-block:: scala

  case Connected =>
    udpConnectionActor = sender // Save the worker ref for later use

The actor passed in the ``handler`` parameter will receive inbound UDP datagrams sent to the bound address:

.. code-block:: scala

  case Received(dataByteString) => // Do something with the data

The ``Received`` message contains the payload of the datagram but unlike in the connectionless case, no sender address
will be provided, as an UDP connection only receives messages from the endpoint it has been connected to.

It is also possible to send UDP datagrams using the ``ActorRef`` of the worker saved in ``udpWorker``:

.. code-block:: scala

 udpConnectionActor ! Send(data)

Again, the send does not contain a remote address, as it is always the endpoint we have been connected to.

.. note::
  There is a small performance benefit in using connection based UDP API over the connectionless one.
  If there is a SecurityManager enabled on the system, every connectionless message send has to go through a security
  check, while in the case of connection-based UDP the security check is cached after connect, thus writes does
  not suffer an additional performance penalty.

Throttling Reads and Writes
^^^^^^^^^^^^^^^^^^^^^^^^^^^

*This section is not yet ready. More coming soon*


Architecture in-depth
---------------------

For further details on the design and internal architecture see :ref:`io-layer`.

.. _spray.io: http://spray.io
