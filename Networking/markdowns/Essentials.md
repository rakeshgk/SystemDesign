# Networking Essentials

## Networking Layers

The Open Systems Interconnection (OSI) model is a conceptual framework created by the ISO that standardizes network communication into seven distinct layers. While the full networking stack is fascinating, there are three key layers that come up most often in system design interviews.

![OSI Layers](../assets/OSI-Layers.png)

1. **Application Layer** - Deals with application protocols like DNS, HTTP, Websockets, WebRTC. These are common protocols that build on top of TCP  (or UDP, in the case of WebRTC) to provide a layer of abstraction for different types of data typically associated with web applications.
2. **Transport Layer** - Deals with end-to-end communication protocols like TCP, QUIC or UDP. This layer provides reliability, ordering and flow control.
3. **Network Layer** - At this layer is IP, the protocol that handles routing and addressing. It's responsible for breaking the data into packets, handling packet forwarding between networks, and providing best-effort delivery to any destination IP address on the network.

## Network Layer Protocols 

1. This layer is dominated by the IP protocol, which is responsible for routing and addressing. 
2. In a system, nodes are assigned IPs usually by a DHCP server when they boot up. These IP addresses are arbitrary.
3. If I want to, I can create a private network with my servers and give them any IP address I want, but if we want internet traffic to be able to find them we will need to use IP addresses that are routable and allocated by a Regional Internet Registry (RIR). These assigned IP addresses are called public IPs and are used to identify devices on the Internet.

## Transport Layer Protocols

The transport layer is where we establish end-to-end communication between applications. They give us some guarantees instead of handing us a jumbled mess of packets. The three primary protocols at this layer are TCP, UDP, and QUIC, each with distinct characteristics that make them suitable for different use cases.

### UDP: Fast but unreliable

User Datagram Protocol (UDP) offers few features on top of IP but is very fast. Spray and pray is the right way to think about this. It provides a simpler, connectionless service with no guarantees of delivery, ordering, or duplicate protection.

Key characteristics of UDP include:

1. **Connectionless**: No handshake or connection setup
2. **No guarantee of delivery**: Packets may be lost without notification
3. **No ordering**: Packets may arrive in a different order than sent
4. **Lower latency**: Less overhead means faster transmission
5. **Lack of Browser support**: Browsers don't have widespread support for UDP yet outside of WebRTC 

UDP is perfect for applications where speed is more important than reliability, such as live video streaming, online gaming, VoIP, and DNS lookups. In these cases the application or client is equipped to handle the occasional packet loss or out of order packet. For VOIP as an example, the client might just drop the occasional packet leading to a hiccup in the audio but overall the conversation is still intelligible. This is vastly preferable to retransmitting those lost packets and clogging up the network with ACKs. 

### TCP: Reliable but with Overhead

Transmission Control Protocol (TCP) is the workhorse of the internet. It provides reliable, ordered, and error-checked delivery of data. It establishes a connection through a three-way handshake and maintains that connection throughout the communication session.

This connection is called a "stream" and is a stateful connection between the client and server. TCP will ensure that recipients of messages acknowledge their receipt and, if they don't, will retransmit the message until it is acknowledged.

3-way handshake between the client and server to initiate a TCP connection. 
1. **SYN**: The client sends a SYN (synchronize) packet to the server to request a connection.
2. **SYN-ACK**: The server responds with a SYN-ACK (synchronize-acknowledge) packet to acknowledge the request.
3. **ACK**: The client sends an ACK (acknowledge) packet to establish the connection.

After the data transfer is complete, the client and server close the TCP connection using a four-way handshake.
1. **FIN**: The client sends a FIN (finish) packet to the server to terminate the connection.
2. **ACK**: The server acknowledges the FIN packet with an ACK.
3. **FIN**: The server sends a FIN packet to the client to terminate its side of the connection.
4. **ACK**: The client acknowledges the server's FIN packet with an ACK.

Key Characteristics of TCP: 

1. **Connection-oriented**: Establishes a dedicated connection before data transfer
2. **Reliable delivery**: Guarantees that data arrives in order and without errors
3. **Flow control**: Prevents overwhelming receivers with too much data
4. **Congestion control**: Adapts to network congestion to prevent collapse
5. TCP is ideal for applications where data integrity is critical — that is, basically everything where UDP is not a good fit.

## Application Layer Protocol

These protocols define how applications communicate and are built on top of Transport Layer protocols. 

### HTTP/HTTPS: The Web's Foundation

Hypertext Transfer Protocol (HTTP) is the de-facto standard for data communication on the web. It's a request-response protocol where clients send requests to servers, and servers respond with the requested data. HTTP is a stateless protocol. Each request is independent and the server doesn't need to maintain any information about previous requests. 

HTTPS adds a security layer (TLS/SSL) to encrypt communications, protecting against eavesdropping and man-in-the-middle attacks. If you're building a public website you're going to be using HTTPS without exception. Generally speaking this means that the contents of your HTTP requests and responses are encrypted and safe in transit.

HTTP includes a few key concepts like

1. **Request Methods** - GET, PUT, POST, PATCH, DELETE
    * PUT - Update data on the server
    * PATCH - Update a resource partially
    * PUT replaces the entire resource while PATCH updates part of a resource. Clients typically send the full object with PUT while clients only send the changed fields as part of PATCH. 
2. **Status Codes** 
    * SUCCESS (2xx)
    * Moved (3xx)
    * Client errors (4xx)
    * Server errors (5xx)
3. **Headers** - Metadata about the request or response
4. **Body** - The actual content being transferred

### REST: Simple and Flexible

Representational State Transfer (REST) is an architectural style for designing APIs. It defines principles (opinions) such as:

1. Resources should have URLs
2. Use HTTP methods meaningfully
3. Stateless communication
4. Standard representations (usually JSON)

REST is the most common API paradigm you'll use in system design interviews. It's a simple and flexible way to create APIs that are easy to understand and use. The core principle behind REST is that clients are often performing simple operations against resources (think of them like database tables or files on a server).

REST is what we should think by default when answering system-design interview questions. However, REST is not going to be the most performant solution for very high throughput services, and generally speaking JSON is a pretty inefficient format for serializing and deserializing data.

### GraphQL : Flexible Data Fetching

GraphQL is an API paradigm that allows clients to request exactly the data they need. When the frontend team wants to display a new page, they can either

1. Cobble together a bunch of different requests to backend endpoints (imagine querying 1 API for a list of users and making 10 API calls to get their details
2. Create huge aggregation APIs which are hard to maintain and slow to change
3. Write brand new APIs for every new page they want to display. 

None of these are particularly good solutions but it's easy to run into them with a standard REST API. The problem with under-fetching is that you may need multiple requests and round trips. This adds overhead and latency to the page load. Over-fetching is the opposite: when we pack way more than we need in an API response to guard ourselves against future use-cases that we don't have today. It means that APIs take a long time to load and return too much data.

GraphQL solves these problems by allowing the frontend team to flexibly query the backend for exactly the data they need. The backend can then respond with the data in the shape that the frontend needs it. This is a great fit for mobile apps and other use-cases where you want to reduce the amount of data transferred.

In most GraphQL architectures, there is a GraphQL server (or GraphQL gateway) sitting between the client and the backend services.

```
Client (Web/Mobile)
    |
    | GraphQL Query
    v
GraphQL Server
    |
    +--> User Service
    |
    +--> Product Service
    |
    +--> Order Service
    |
    +--> Database
```

The GraphQL server receives this query and then:

1. Parses and validates it.
2. Determines which fields are requested.
3. Invokes resolver functions.
4. Fetches data from one or more backend systems.
5. Combines the results.
6. Returns a single JSON response.

GraphQL is a great fit for use-cases where the frontend team needs to iterate quickly and adjust. They can flexibly query the backend for exactly the data they need. On the other hand, execution of these GraphQL queries can be a source of latency and complexity for the backend — sometimes involving the same bespoke backend code that we're trying to avoid. In practice, GraphQL finds its sweet spot with complex clients and when multiple teams are making wide queries to overlapping data.

### gRPC: Efficient Service Communication 

gRPC is a high-performance RPC (Remote Procedure Call) that uses HTTP/2 and Protocol Buffers.

Think of Protocol Buffers like JSON but with a more rigid schema that allows for better performance and more efficient serialization.

gRPC builds on this to provide service definitions. These definitions are compiled into a client and server stub which a wide variety of languages and frameworks can consume to build clients and services. 

gRPC shines in microservices architectures where services need to communicate efficiently. Its strong typing helps catch errors at compile time rather than runtime, and its binary protocol is more efficient than JSON over HTTP. 

Consider gRPC for internal service-to-service communication, especially when performance is critical or when latencies are dominated by the network rather than the work the server is doing. Having internal APIs using gRPC and external APIs using REST is a great way to get the benefits of a binary protocol without the complexity of a public-facing API. 

In many interviews, using REST both for internal and external APIs is fine and you can build from there depending on the needs of the problem and probes from your interviewer.

### Server Sent Events (SSE) : Real-time Push Communication

SSE is a nice hack on top of HTTP that allows a server to stream many messages, over time, in a single response from the server. With most HTTP APIs you'd get a single, cohesive JSON blob as a response from the server that is processed once the whole thing has been received. Since we have to wait for the whole response to come in before we can process it, it's not much good for push notifications! On the other hand, with SSE, the server can push many messages as "chunks" in a single response from the server. Clients can then process each message as it comes in. It's still one big HTTP response (same TCP connection), but it comes in over many smaller packets and clients are expected to process each line of the body individually to allow them to react to the data as it comes in.

We can't keep an SSE connection open for too long because the server (or the load balancer, or a middle box proxy) will close down the connection. So the SSE standard defines the behavior of an `EventSource` object that, once the connection is closed, will automatically reconnect with the ID of the last message received. Servers are expected to keep track of prior messages that may have been missed while the client was disconnected and resend them.

Use this pattern in system design inteviews where you want clients to get notifications or events as soon as they happen. For example - Real-time online bidding. 

### WebSockets: Real-Time Bidirectional Communication

