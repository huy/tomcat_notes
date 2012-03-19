# Core objects

**Connector** 

When starting Connector is server endpoint, connector creates server socket listen on port. 
On new incomming request, connector accept a socket , creates or get processor from pool, assign socket to it, 
and lets processor's thread to finish the work. Connector has 1 to many relationship with processor.

Each connector is associated with one protocol

code: org/apache/catalina/connector/Connector.java

**Processor** 

There is one thread per procesor's instance i.e processor code run on its own thread. Processor instantiates HttpRequest, HttpResponse passing
InputStream and OutputStream to them.

Procesor interfaces with container to find/create appropriate Servlet instance and invokes its method service(Request,Response)

e.g AjpProcessor,Http11Processor is maintained in a pool associated with a ProtocolHandler
e.g. Http11ConnectionHandler which is created one per Http11Protocol. 

**HttpRequest, HttpRequestFacade**

Representing information of HTTP protocol request, contains logic to parse (lazily) requests and convenient interface
for Servlet 

**Servlet**

Is unit of code implementing custom business logic writen and deployed by developers 

**Protocol aka ProtocolHandler** 

Define logic (FSM) to process message received

code: org/apache/coyote/http11/Http11Protocol.java, org/apache/coyote/ajp/AjpProtocol.java

Number of processor is closed to Max Threads

code: 

**Request** 

Represents Http Request, there is org.apache.coyote.Request and its wrapper org.apache.catalina.connector.Request

Request is created one per Processor


