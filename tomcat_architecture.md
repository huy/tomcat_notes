# Core objects

**Connector** 

When starting Connector, connector creates server socket listen on port. 
On new incomming request, connector accept a socket , creates or get processor from pool, assign socket to it, 
and lets processor's thread to finish the work. Connector has 1 to many relationship with processor.

Each connector is associated with one Protocol Handler e.g. Http or AJP, which is 

The connector start() method after performing initialization will call protocolHandler.start()

code: org/apache/catalina/connector/Connector.java

**Protocol aka ProtocolHandler** 

Define logic (FSM) to process message received

The ProtocolHandler start() method call JIoEndpoint.start()
The ProtocolHandler process() is called by Worker to process message from socket

        public boolean process(Socket socket) {
            Http11Processor processor = recycledProcessors.poll();
            try {

                if (processor == null) {
                    processor = createProcessor();
                }

                if (processor instanceof ActionHook) {
                    ((ActionHook) processor).action(ActionCode.ACTION_START, null);
                }

                if (proto.isSSLEnabled() && (proto.sslImplementation != null)) {
                    processor.setSSLSupport
                        (proto.sslImplementation.getSSLSupport(socket));
                } else {
                    processor.setSSLSupport(null);
                }

                processor.process(socket);
                return false;

            } catch(java.net.SocketException e) {
                // SocketExceptions are normal
                Http11Protocol.log.debug
                    (sm.getString
                     ("http11protocol.proto.socketexception.debug"), e);
            } catch (java.io.IOException e) {
                // IOExceptions are normal
                Http11Protocol.log.debug
                    (sm.getString
                     ("http11protocol.proto.ioexception.debug"), e);
            }
            // Future developers: if you discover any other
            // rare-but-nonfatal exceptions, catch them here, and log as
            // above.
            catch (Throwable e) {
                // any other exception or error is odd. Here we log it
                // with "ERROR" level, so it will show up even on
                // less-than-verbose logs.
                Http11Protocol.log.error
                    (sm.getString("http11protocol.proto.error"), e);
            } finally {
                //       if(proto.adapter != null) proto.adapter.recycle();
                //                processor.recycle();

                if (processor instanceof ActionHook) {
                    ((ActionHook) processor).action(ActionCode.ACTION_STOP, null);
                }
                recycledProcessors.offer(processor);
            }
            return false;
        }

The process() method is in turn obtains the processor from pool then call the Processor method process() 


code: org/apache/coyote/http11/Http11Protocol.java, org/apache/coyote/ajp/AjpProtocol.java

**End point**

This class implements one listener thread accepts on a socket and creates a new worker thread for each incoming connection.


    protected class Acceptor implements Runnable {
        /**
         * The background thread that listens for incoming TCP/IP connections and
         * hands them off to an appropriate processor.
         */
        public void run() {
            // Loop until we receive a shutdown command
            while (running) {
     ...
                // Accept the next incoming connection from the server socket
                try {
                    Socket socket = serverSocketFactory.acceptSocket(serverSocket);
                    serverSocketFactory.initSocket(socket);
                    // Hand this socket off to an appropriate processor
                    if (!processSocket(socket)) {
                        // Close socket right away
                        try {
                            socket.close();
                        } catch (IOException e) {
                            // Ignore
                        }
                    }
                }catch ( IOException x ) {
                    if ( running ) log.error(sm.getString("endpoint.accept.fail"), x);
                } catch (Throwable t) {
                    log.error(sm.getString("endpoint.accept.fail"), t);
                }

                // The processor will recycle itself when it finishes

            }

        }

The method processSocket pick a thred from pool and assign a socket to it


    protected boolean processSocket(Socket socket) {
        try {
            if (executor == null) {
                getWorkerThread().assign(socket);
            } else {
                executor.execute(new SocketProcessor(socket));
            }
        } catch (Throwable t) {
            // This means we got an OOM or similar creating a thread, or that
            // the pool and its queue are full
            log.error(sm.getString("endpoint.process.fail"), t);
            return false;
        }
        return true;
    }

The worker thread is waiting until got socket assigned by the JIoEndPoint. After getting the socket, it
makes a call back to the protocol handler process() method

    protected class Worker implements Runnable {

        protected Thread thread = null;
        protected boolean available = false;
        protected Socket socket = null;

     ...
        public void run() {

            // Process requests until we receive a shutdown signal
            while (running) {

                // Wait for the next socket to be assigned
                Socket socket = await();
                if (socket == null)
                    continue;

                // Process the request from this socket
                if (!setSocketOptions(socket) || !handler.process(socket)) {
                    // Close socket
                    try {
                        socket.close();
                    } catch (IOException e) {
                    }
                }

                // Finish up this request
                socket = null;
                recycleWorkerThread(this);

            }

        }


source : JIoEndpoint.java 

**Processor** 

There is one thread per procesor's instance i.e processor code run on its own thread. 
Processor instantiates HttpRequest, HttpResponse passing InputStream and OutputStream of the socket to them.
Processor parses the request line and header then procesor ask the connector to obtain container and calls 
container's method invoke to find/create appropriate Servlet instance and invokes its method service(Request,Response)

e.g AjpProcessor,Http11Processor is maintained in a pool associated with a ProtocolHandler
e.g. Http11ConnectionHandler which is created one per Http11Protocol. 

Number of processor is closed to Max Threads




**Container**

There are different types of container Engine, Host, Context and Wrapper. There is 1 to many relationship between
Engine and Host, Host and Context, Context and Wrapper.

Container invoke delegate to pipeline invoke, which contains series of  tasks called valves. Each pipeline has one 
basic valve and several additional valves defined in server.xml (e.g RemoteAddressFilter,AccessLog). Through the pipeline, 
after finishing one valve, the next valve is called. The basic valve is always called last.

The pipeline implements ValveContext to faciliate the calling of valves in the chain, the ValveContext is passed on invoke 
method of each valve so they can access the "Context" of invokation.

**Wrapper**

Wrapper represents individual Servlet

**Context**

Context represents individual Web application, which usually has many Servlets

**HttpRequest, HttpRequestFacade**

Representing information of HTTP protocol request, contains logic to parse (lazily) requests and convenient interface
for Servlet 

**Servlet**

Is unit of code implementing custom business logic writen and deployed by developers 

**Request** 

Represents Http Request, there is org.apache.coyote.Request and its wrapper org.apache.catalina.connector.Request

Request is created one per Processor


