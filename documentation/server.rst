Server API
==========

.. currentmodule:: p4p.server

Running a PVA Server
--------------------

The :py:class:`Server` starts/stops a PVAccess server.
In order to be useful, a Server must be associated with one or more Providers.

Two Provider containers are available: :py:class:`StaticProvider` or :py:class:`DynamicProvider`.
Both are used with one of the SharedPV classes:,
:py:class:`thread.SharedPV`, :py:class:`asyncio.SharedPV`, and/or :py:class:`cothread.SharedPV`.
These different threading models may be mixed into a single Provider.

Example
^^^^^^^

A server with a single "mailbox" PV. ::

    import time
    from p4p.nt import NTScalar
    from p4p.server import Server, StaticProvider
    from p4p.server.thread import SharedPV
    
    pv = SharedPV(nt=NTScalar('d'), # scalar double
                  initial=0.0)      # setting initial value also open()'s
    @pv.put
    def handle(pv, op):
        pv.post(op) # just store and update subscribers
        op.done()

    provider = StaticProvider('arbitrary')
    provider.add('demo:pv:name', pv) # PV name only appears here

    Server.forever(providers=[provider]) # help which runs until KeyboardInterrupt

This server can be tested using the included command line tools. eg. ::

    $ python -m p4p.client.cli get demo:pv:name
    $ python -m p4p.client.cli put demo:pv:name
    $ python -m p4p.client.cli get demo:pv:name

And in another shell. ::

    $ python -m p4p.client.cli monitor demo:pv:name


Server API
----------

.. autoclass:: Server

    .. automethod:: conf

    .. automethod:: stop

.. autoclass:: StaticProvider

    .. automethod:: close

    .. automethod:: add

    .. automethod:: remove

Global Provider registry
^^^^^^^^^^^^^^^^^^^^^^^^

If it becomes necessary for a Provider to be included in a server which is started
outside of Python code, then it must be placed in the global provider registry
with installProvider().

.. autofunction:: installProvider

.. autofunction:: removeProvider

For situations where PV names are not known ahead of time,
or when PVs are created only as requested, DynamicProvider can be used.

.. autoclass:: DynamicProvider

    .. autoattribute:: NotYet


DynamicProvider Handler Interface
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A :py:class:`DynamicProvider` Handler class will define the following:

.. class:: ProviderHandler

    .. method:: testChannel(pvname)

        Called with a PV name which some client is searching for.

        :return: True to claim this PV.  False to "permenently" disclaim this PV.  Or :py:attr:`DynamicProvider.NotYet` to temporarily disclaim.

        Each DynamicProvider maintains a cache of negative search results (when ```testChannel()==False```)
        to avoid extra work on a subnet with many clients searching for non-existant PVs.
        If it is desirable to defeat this behavour, for example as part of lazy pv creation,
        then testChannel() can return :py:attr:`DynamicProvider.NotYet` instead of False.

    .. method:: makeChannel(pvname, src):

        Called when a client attempts to create a Channel for some PV.
        The object which is returned will not be collected until
        the client closes the Channel or becomes disconnected.

        :return: A :py:class:`SharedPV` instance.


ServerOperation
^^^^^^^^^^^^^^^

This class is passed to SharedPV handler put() and rpc() methods.

.. autoclass:: ServerOperation

    .. automethod:: pvRequest

    .. automethod:: value

    .. automethod:: done

    .. automethod:: info

    .. automethod:: warn

    .. automethod:: name

    .. automethod:: peer


SharedPV concurrency options
----------------------------

There is a SharedPV class for each of other three concurrency models
supported: OS threading, asyncio coroutines, and cothreads.
All have the same methods as :py:class:`thread.SharedPV`.

The difference between :py:class:`thread.SharedPV`, :py:class:`asyncio.SharedPV`, and :py:class:`cothread.SharedPV`
is the context in which the handler methods are called (an OS thread, an asyncio coroutine, or a cothread).
This distinction determines how blocking operations may be carried out.

Note that :py:class:`thread.SharedPV` uses a fixed size thread pool.
This limits the number of concurrent callbacks.


SharedPV API
------------

.. currentmodule:: p4p.server.thread

.. autoclass:: SharedPV

    .. automethod:: open

    .. automethod:: close

    .. automethod:: post


SharedPV Handler Interface
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: Handler

    .. automethod:: put

    .. automethod:: rpc

    .. automethod:: onFirstConnect

    .. automethod:: onLastDisconnect

asyncio or cothread
^^^^^^^^^^^^^^^^^^^

.. currentmodule:: p4p.server.asyncio

.. class:: SharedPV

    Same arguments as :py:class:`thread.SharedPV` except that queue= is replaced with loop= .

    :param loop: An asyncio event loop, or None to use the default.

.. currentmodule:: p4p.server.cothread

.. class:: SharedPV
