Getting started
===============

The current verison of aiosmf only support Python-based smf clients. In order
to have a working example we need a server, which can be built using C++, Go,
or Java. We'll be using the C++-based server described in this introductory smf
post:

    https://nwat.xyz/blog/2019/03/22/getting-started-with-the-smf-rpc-framework/

This is the rpc specification for the example server. The Put interface
receives a PutRequest and returns a PutResponse. The first step in using aiosmf
is compiling this specification.

.. code-block:: text

    namespace kvstore.fbs;
    
    table PutRequest {
      key: string;
      value: string;
    }
    
    table PutResponse {
      text: string;
    }
    
    rpc_service MemoryNode {
      Put(PutRequest):PutResponse;
    }

First use the flatbuffers compiler to generate a Python library that implements
each of the PutRequest and PutResponse types.

.. code-block:: bash

    [user@localhost testtest]$ flatc -o . --python kvstore.fbs
    [user@localhost testtest]$ ls -l kvstore/fbs/
    total 8
    -rw-rw-r--. 1 user user    0 Mar 30 08:18 __init__.py
    -rw-rw-r--. 1 user user 1277 Mar 30 08:18 PutRequest.py
    -rw-rw-r--. 1 user user  939 Mar 30 08:18 PutResponse.py

Next use the smf compiler to generate a Python smf client specific to this service.

.. code-block:: bash

    [user@localhost testtest]$ smfc --filename kvstore.fbs --language=python
    [user@localhost testtest]$ ls -l kvstore/fbs/
    total 12
    -rw-rw-r--. 1 user user    0 Mar 30 08:18 __init__.py
    -rw-rw-r--. 1 user user  420 Mar 30 08:19 kvstore_smf_client.py
    -rw-rw-r--. 1 user user 1277 Mar 30 08:18 PutRequest.py
    -rw-rw-r--. 1 user user  939 Mar 30 08:18 PutResponse.py

This is the contents of the generated client. The client accepts an smf
connection (which we'll discuss next), and a method for each of the service
interfaces. In this case that is a single Put method.

.. code-block:: python

    # Generated by smfc.
    # Any local changes WILL BE LOST.
    # source: kvstore.fbs
    
    import kvstore.fbs.PutResponse
    
    class MemoryNodeClient:
        def __init__(self, conn):
            self._conn = conn
            
        async def Put(self, x):
            # request id = 504045560 ^ 3345117782
            buf, status = await self._conn.call(x, 3647565230)
            return kvstore.fbs.PutResponse.PutResponse.GetRootAsPutResponse(buf, 0), status

Finally we can begin building a sample client. The first thing that is done is
to establish a connection to the smf server, and pass this connection to an
instance of the generated client:

.. code-block:: python

    import asyncio
    import flatbuffers
    import kvstore
    import aiosmf
    from kvstore.fbs.kvstore_smf_client import MemoryNodeClient
    import kvstore.fbs.PutRequest
    
    async def main():
        conn = await aiosmf.create_connection("127.0.0.1:20776")
        client = MemoryNodeClient(conn)

Once the connection is established the rpc methods can be invoked. To invoke the Put method
we must first create a PutRequest. This is done using the standard flatbuffers api which
results in a buffer containing the serialized form of the request:

.. code-block:: python
    
        # build an rpc request buffer
        builder = flatbuffers.Builder(32)
        key = builder.CreateString("my.key")
        value = builder.CreateString("my.value")
        kvstore.fbs.PutRequest.PutRequestStart(builder)
        kvstore.fbs.PutRequest.PutRequestAddKey(builder, key)
        kvstore.fbs.PutRequest.PutRequestAddValue(builder, value)
        put = kvstore.fbs.PutRequest.PutRequestEnd(builder)
        builder.Finish(put)
        buf = builder.Output()

And finally the remote method is invoked and we print out the results.

.. code-block:: python
    
        resp, status = await client.Put(buf)
        print(resp.Text(), status)

The client and the connection should be shutdown to cleanup resources:

.. code-block:: python

    
        conn.close()
        await conn.wait_closed()

Invoke this sample client using an asyncio loop:

.. code-block:: python

    
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

If your server uses zstd compression add incoming and outgoing filters to the
connection:

.. code-block:: python

    conn = await aiosmf.create_connection("127.0.0.1:20776",
        incoming_filters=(aiosmf.ZstdDecompressionFilter(),),
        outgoing_filters=(aiosmf.ZstdCompressionFilter(128),))
