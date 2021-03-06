.. _advanced-structured-logging:

Structured logging
==================
Although traditional logging is a useful tool to diagnose the behavior of an application, it has its own
problem; the resulting log messages are not always machine-friendly. This section explains the Armeria API for
retrieving the information collected during request life cycle in a machine-friendly way.

What properties can be retrieved?
---------------------------------
:api:`RequestLog` provides all the properties you can retrieve:

+----------------------------------------------------------------------------------------------------+
| Request properties                                                                                 |
+=============================+======================================================================+
| ``requestStartTimeMillis``  | when the request processing started                                  |
+-----------------------------+----------------------------------------------------------------------+
| ``requestDurationNanos``    | the duration took to process the request completely                  |
+-----------------------------+----------------------------------------------------------------------+
| ``requestLength``           | the byte length of the request content                               |
+-----------------------------+----------------------------------------------------------------------+
| ``requestCause``            | the cause of request processing failure (if any)                     |
+-----------------------------+----------------------------------------------------------------------+
| ``sessionProtocol``         | the protocol of the connection (e.g. ``H2C``)                        |
+-----------------------------+----------------------------------------------------------------------+
| ``serializationFormat``     | the serialization format of the content (e.g. ``tbinary``, ``none``) |
+-----------------------------+----------------------------------------------------------------------+
| ``host``                    | the name of the virtual host that serves the request                 |
+-----------------------------+----------------------------------------------------------------------+
| ``requestHeaders``          | the HTTP headers of the request.                                     |
|                             | the header contains the method (e.g. ``GET``, ``POST``),             |
|                             | the path (e.g. ``/thrift/foo``),                                     |
|                             | the query (e.g. ``foo=bar&bar=baz``), the content type, etc.         |
+-----------------------------+----------------------------------------------------------------------+
| ``requestContent``          | the serialization-dependent content object of the request.           |
|                             | ``ThriftCall`` for Thrift. ``null`` otherwise.                       |
+-----------------------------+----------------------------------------------------------------------+
| ``requestContentPreview``   | the preview of the request content                                   |
+-----------------------------+----------------------------------------------------------------------+

+-----------------------------+----------------------------------------------------------------------+
| Response properties                                                                                |
+=============================+======================================================================+
| ``responseStartTimeMillis`` | when the response processing started                                 |
+-----------------------------+----------------------------------------------------------------------+
| ``responseDurationNanos``   | the duration took to process the response completely                 |
+-----------------------------+----------------------------------------------------------------------+
| ``responseLength``          | the byte length of the response content                              |
+-----------------------------+----------------------------------------------------------------------+
| ``responseCause``           | the cause of response processing failure (if any)                    |
+-----------------------------+----------------------------------------------------------------------+
| ``totalDurationNanos``      | the duration between the request start and the response end          |
|                             | (i.e. response time)                                                 |
+-----------------------------+----------------------------------------------------------------------+
| ``responseHeaders``         | the HTTP headers of the response.                                    |
|                             | the header contains the statusCode (e.g. 404), the content type, etc.|
+-----------------------------+----------------------------------------------------------------------+
| ``responseContent``         | the serialization-dependent content object of the response.          |
|                             | ``ThriftReply`` for Thrift. ``null`` otherwise.                      |
+-----------------------------+----------------------------------------------------------------------+
| ``responseContentPreview``  | the preview of the response content                                  |
+-----------------------------+----------------------------------------------------------------------+

Availability of properties
--------------------------
Armeria handles requests and responses in a stream-oriented way, which means that some properties are revealed
only after the streams are processed to some point. For example, there's no way to know the ``requestLength``
until the request processing ends. Also, some properties related to the (de)serialization of request content,
such as ``serializationFormat`` and ``requestContent``, will not be available when request processing just
started.

To get notified when a certain set of properties are available, you can add a listener to a ``RequestLog``:

.. code-block:: java

    import com.linecorp.armeria.common.HttpRequest;
    import com.linecorp.armeria.common.HttpResponse;
    import com.linecorp.armeria.common.logging.RequestLog;
    import com.linecorp.armeria.common.logging.RequestLogAvailability;
    import com.linecorp.armeria.server.ServiceRequestContext;
    import com.linecorp.armeria.server.AbstractHttpService;

    public class MyService extends AbstractHttpService {
        @Override
        public HttpResponse serve(ServiceRequestContext ctx, HttpRequest req) {
            final RequestLog log = ctx.log();

            log.addListener(log -> {
                System.err.println("Handled a request: " + log);
            }, RequestLogAvailability.COMPLETE);

            return super.serve(ctx, req);
        }
    }

Note that :api:`RequestLogAvailability` is specified when adding a listener.
:api:`RequestLogAvailability` is an enum that is used to express which :api:`RequestLog` properties
you are interested in. ``COMPLETE`` will make your listener invoked when all properties are available.

Set ``serializationFormat`` and ``requestContent`` early if possible
--------------------------------------------------------------------
Armeria depends on the ``serializationFormat`` and ``requestContent`` property to determine whether a request
is an RPC and what the method name of the call is. If you are sure the request you handle is not an RPC, set
the ``serializationFormat`` and ``requestContent`` property explicitly to ``NONE`` and ``null`` so that Armeria
and other log listeners get the information sooner:

.. code-block:: java

    import com.linecorp.armeria.common.HttpRequest;
    import com.linecorp.armeria.common.HttpResponse;
    import com.linecorp.armeria.common.SerializationFormat;
    import com.linecorp.armeria.server.ServiceRequestContext;
    import com.linecorp.armeria.server.HttpService;

    public class MyService implements HttpService {
        @Override
        public HttpResponse serve(ServiceRequestContext ctx, HttpRequest req) {
            ctx.logBuilder().serializationFormat(SerializationFormat.NONE);
            ctx.logBuilder().requestContent(null);
            ...
        }
    }

Consider using ``AbstractHttpService`` which sets the ``serializationFormat`` and ``requestContent``
automatically for you:

.. code-block:: java

    import com.linecorp.armeria.common.HttpResponseWriter;
    import com.linecorp.armeria.common.thrift.ThriftSerializationFormats;
    import com.linecorp.armeria.server.AbstractHttpService;

    public class MyService extends AbstractHttpService {
        @Override
        public void doGet(ServiceRequestContext ctx, HttpRequest req, HttpResponseWriter res) {
            // serializationFormat and requestContent will be set to NONE and null
            // automatically when this method returns.
            ...
        }

        @Override
        public void doPost(ServiceRequestContext ctx, HttpRequest req, HttpResponseWriter res) {
            // Set serializationFormat explicitly.
            ctx.logBuilder().serializationFormat(ThriftSerializationFormats.BINARY);
            // This will prevent AbstractHttpService from setting requestContent to null
            // automatically. You should call RequestLogBuilder.requestContent(...) later
            // when the content is determined.
            ctx.logBuilder().deferRequestContent();
            // Alternatively, you can set requestContent right here:
            // ctx.logBuilder().requestContent(...);
            ...
        }
    }

Enabling content previews
-------------------------
Armeria provides the ``requestContentPreview`` and ``responseContentPreview`` properties in :api:`RequestLog`
to retrieve the textual representation of the first N bytes of the request and response content.
However, the properties are disabled by default due to performance overhead and thus they always return ``null``.
You can enable it when you configure :api:`Server`, :api:`VirtualHost` or :api:`Client`.

.. code-block:: java

    import com.linecorp.armeria.server.ServerBuilder;
    import com.linecorp.armeria.server.VirtualHostBuilder;

    ServerBuilder sb = new ServerBuilder();
    ...
    // Enable previewing the content with the maxium length of 100 for textual content.
    sb.contentPreview(100);
    ...
    VirtualHostBuilder vhb = new VirtualHostBuilder("http://example.com");
    // In this case, the property of virtual host takes precedence over that of server.
    vhb.contentPreview(150);
    ...
    sb.virtualHost(vhb.build());

.. code-block:: java

    import com.linecorp.armeria.client.HttpClientBuilder;

    HttpClientBuilder cb = new HttpClientBuilder();
    ...
    cb.contentPreview(100);

Note that the ``contentPreview()`` method enables the previews only for textual content
which meets one of the following cases:

- when its type matches ``text/*`` or ``application/x-www-form-urlencoded``.
- when its charset has been specified. e.g. application/json; charset=utf-8.
- when its subtype is ``xml`` or ``json``. e.g. application/xml, application/json.
- when its subtype ends with ``+xml`` or ``+json``. e.g. application/atom+xml, application/hal+json

You can also customize the previews by specifying your own :api:`ContentPreviewerFactory` implementation.
The following example enables the textual preview of first 100 characters for the content type of ``text/*``,
and the hex dump preview of first 100 bytes for other types:

.. code-block:: java

    import io.netty.buffer.ByteBufUtil;
    import com.linecorp.armeria.common.MediaType;
    import com.linecorp.armeria.common.logging.ContentPreviewer;

    ServerBuilder sb = new ServerBuilder();

    sb.contentPreviewerFactory((ctx, headers) -> {
        MediaType contentType = headers.contentType();
        if (contentType != null && contentType.is(MediaType.ANY_TEXT_TYPE)) {
            // Produces the textual preview of the first 100 characters.
            return ContentPreviewer.ofText(100);
        }
        // Produces the hex dump of the first 100 bytes.
        return ContentPreviewer.ofBinary(100, byteBuf -> {
            // byteBuf has no more than 100 bytes.
            return ByteBufUtil.hexDump(byteBuf);
        });
    });

You can write your own :api:`ContentPreviewer` to change the way to make the preview, e.g.

.. code-block:: java

    class HexDumpContentPreviewer implements ContentPreviewer {
        @Nullable
        private StringBuilder builder = new StringBuilder();
        @Nullable
        private String preview;

        @Override
        public void onHeaders(HttpHeaders headers) {
            // Invoked when headers of a request or response is received.
        }

        @Override
        public void onData(HttpData data) {
            // Invoked when a new content is received.
            assert builder != null;
            builder.append(ByteBufUtil.hexDump(data.array(), data.offset(), data.length()));
        }

        @Override
        public boolean isDone() {
            // If it returns true, no further event is invoked but produce().
            return preview != null;
        }

        @Override
        public String produce() {
            // Invoked when a request or response ends.
            if (preview != null) {
                return preview;
            }
            preview = builder.toString();
            builder = null;
            return preview;
        }
    }
    ...
    ServerBuilder sb = new ServerBuilder();
    ...
    sb.contentPreviewerFactory((ctx, headers) -> new HexDumpContentPreviewer());

.. _nested-log:

Nested log
----------

When you retry a failed attempt, you might want to record the result of each attempt and to group them under
a single :api:`RequestLog`. A :api:`RequestLog` can contain more than one child :api:`RequestLog`
to support this sort of use cases.

.. code-block:: java

    import com.linecorp.armeria.common.logging.RequestLogBuilder;

    RequestLogBuilder.addChild(RequestLog);

If the added :api:`RequestLog` is the first child, the request-side log of the :api:`RequestLog` will
be propagated to the parent log. You can add as many child logs as you want, but the rest of logs would not
be affected. If you want to fill the response-side log of the parent log, please invoke:

.. code-block:: java

    RequestLogBuilder.endResponseWithLastChild();

This will propagate the response-side log of the last added child to the parent log. The following diagram
illustrates how a :api:`RequestLog` with child logs looks like:

.. uml::

    @startditaa(--no-separation, scale=0.85)
    /--------------------------------------------------------------\
    |                                                              |
    |  RequestLog                                                  |
    |                                                              |
    |                             /-----------------------------\  |
    |                             :                             |  |
    |  +----------------------+   |      Child RequestLogs      |  |
    |  |                      |   |        e.g. retries         |  |
    |  |                      |   |                             |  |
    |  |   Request side log   |   |  +-----------------------+  |  |
    |  |                      |   |  | Child #1              |  |  |
    |  |                      |   |  | +-------------------+ |  |  |
    |  |     Copied from      |<-------+ Request side log  | |  |  |
    |  |     the first child  |   :  | +-------------------+ |  |  |
    |  |                      |   |  | : Response side log | |  |  |
    |  |                      |   |  | +-------------------+ |  |  |
    |  +----------------------+   |  +-----------------------+  |  |
    |                             |  | ...                   |  |  |
    |  +----------------------+   |  +-----------------------+  |  |
    |  |                      |   |              .              |  |
    |  |                      |   |              .              |  |
    |  |  Response side log   |   |  +-----------------------+  |  |
    |  |                      |   |  | Child #N              |  |  |
    |  |                      |   |  | +-------------------+ |  |  |
    |  |     Copied from      |   |  | : Request side log  | |  |  |
    |  |     the last child   |   |  | +-------------------+ |  |  |
    |  |                      |<-------+ Response side log | |  |  |
    |  |                      |   :  | +-------------------+ |  |  |
    |  +----------------------+   |  +-----------------------+  |  |
    |                             |                             |  |
    |                             \-----------------------------/  |
    |                                                              |
    \--------------------------------------------------------------/
    @endditaa

You can retrieve the child logs using ``RequestLog.children()``.

.. code-block:: java

    final RequestContext ctx = ...;
    ctx.log().addListner(log -> {
        if (!log.children().isEmpty()) {
            System.err.println("A request finished after " + log.children().size() + " attempt(s): " + log);
        } else {
            System.err.println("A request is done: " + log);
        }
    }, RequestLogAvailability.COMPLETE);

:api:`RetryingClient` is a good example that leverages this feature.
See :ref:`retry-with-logging` for more information.
