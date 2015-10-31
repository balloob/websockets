Getting started
===============

Basic example
-------------

.. _server-example:

Here's a WebSocket server example. It reads a name from the client and sends a
message.

.. literalinclude:: ../example/server.py

.. _client-example:

Here's a corresponding client example.

.. literalinclude:: ../example/client.py

On the server side, the handler coroutine ``hello`` is executed once for
each WebSocket connection. The connection is automatically closed when the
handler returns.

Browser-based example
---------------------

Here's an example of how to run a WebSocket server and connect from a browser.

Run this script in a console:

.. literalinclude:: ../example/time.py

Then open this HTML file in a browser.

.. literalinclude:: ../example/time.html
   :language: html

Common patterns
---------------

You will almost always want to process several messages during the
lifetime of a connection. Therefore you must write a loop. Here are the
recommended patterns to exit cleanly when the connection drops, either
because the other side closed it or for any other reason.

Consumer
........

For receiving messages and passing them to a ``consumer`` coroutine::

    @asyncio.coroutine
    def handler(websocket, path):
        while True:
            message = yield from websocket.recv()
            if message is None:
                break
            yield from consumer(message)

:meth:`~websockets.protocol.WebSocketCommonProtocol.recv` returns ``None``
when the connection is closed. In other words, ``None`` marks the end of
the message stream. The handler coroutine should check for that case and
return when it happens.

Producer
........

For getting messages from a ``producer`` coroutine and sending them::

    @asyncio.coroutine
    def handler(websocket, path):
        while True:
            message = yield from producer()
            if not websocket.open:
                break
            yield from websocket.send(message)

:meth:`~websockets.protocol.WebSocketCommonProtocol.send` fails with an
exception when it's called on a closed connection. Therefore the handler
coroutine should check that the connection is still open before attempting
to write and return otherwise.

Both
....

Of course, you can combine the two patterns shown above to read and write
messages on the same connection::

    @asyncio.coroutine
    def handler(websocket, path):
        while True:
            listener_task = asyncio.ensure_future(websocket.recv())
            producer_task = asyncio.ensure_future(producer())
            done, pending = yield from asyncio.wait(
                [listener_task, producer_task],
                return_when=asyncio.FIRST_COMPLETED)

            if listener_task in done:
                message = listener_task.result()
                if message is None:
                    break
                yield from consumer(message)
            else:
                listener_task.cancel()

            if producer_task in done:
                message = producer_task.result()
                if not websocket.open:
                    break
                yield from websocket.send(message)
            else:
                producer_task.cancel()

(This code looks convoluted. If you know a more straightforward solution,
please let me know about it!)

That's really all you have to know! ``websockets`` manages the connection
under the hood so you don't have to.