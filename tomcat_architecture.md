# Core objects

**Connector** 

Is server endpoint i.e. listen port, each connector is associated with one protocol

code: org/apache/catalina/connector/Connector.java

**Protocol aka ProtocolHandler** 

Define logic (FSM) to process message received

code: org/apache/coyote/http11/Http11Protocol.java, org/apache/coyote/ajp/AjpProtocol.java

**Processor** 

e.g AjpProcessor,Http11Processor is maintained in a pool associated with a ProtocolHandler
e.g. Http11ConnectionHandler which is created one per Http11Protocol. 

Number of processor is closed to Max Threads

code: 

**Request** 

Represents Http Request, there is org.apache.coyote.Request and its wrapper org.apache.catalina.connector.Request

Request is created one per Processor


