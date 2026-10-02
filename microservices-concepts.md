## Sharing databases in microservices

Sharing databases in microservices is such that every microservice can write to the same database without any network call (HTTP call) to any other microservice.

Advantages of Sharing databases:

- Simplest way of integration
- no middleman involved (direct db update)
- no latency overhead
- quick development time
- simpler operation
- better performance - no middleman hence no api call needed

Disadvantages of Sharing databases:

- External parties are getting internal details (Cross teams need to know the internals of db schema, are they using soft v/s hard deletes, etc.) Strong coupling between cross teams.

- Sharing DB = sharing the business logic. (Logic showuld be replicated across all the services, hard to manage)

- Risk of data corruption and deletion. (Wrong script execution in one microservice)

- Abusing the shared DB (Other services may get chocked due to frequernt updation / read query by analytics service of one microservice)

## When we should share DB: 
  
So we can share the database initially it is fine.

- When the time is very less and team is small

- No frequent change in data schema

- Read load can be moved to a Replica


## Benefits or when we will have 1 DB per microservice:

- All microservices will be independent and decoupled.

- Different microservice may need different requirement of data base according to use cases such as chat-service requires a high write throughput database such as cassandra, while auth-service may use a relational db such as mysql, so this is possible if microservices have different databases.

- If db is down only affect the dependent microservices.

- Granular control of scaling of service and db.


## Disadvantages of having different db per service:

- Cross service transactions are omplex and expensive. (For transactions need to implement 2 phase commit across services)

- Multiple infra components to be monitored and managed


## Distributed Transactions 

# 2 Phase Commit:

This consist of two steps:

1. Reserve - We should first reserve the things that we need means suppose we need to do multiple things but atomically so instead of directly commiting we will firstly see that can we reserve the resources first across all the services. 
In this we acquire a lock on the resource along with a timer so that if anything goes wrong or even if service itself goes down then we can release the locks when the timer ends.

2. Commit - Now after reserving all the resources we need to actually commit those and needs to assign those resources across microservices so that we can finally send the succcess response to the user that we have completed all the steps within the transaction. 

This is how we implement the two phase commit protocol across distributed systems to make the transactions and to execute all steps atomically.

# If service fails during assignment:

Also if at the time of assignment of the resource if the server crashes then we can recover that with retry mechanism and it will not be having much impact on our side.



## RPC - (Remote Procedural Calls) - For interservice communication in microservice architecture:

It is designed to make a network calls just looks like a function call within the microservices.

So a problem within microservices is such that we have different microservices written in different languages such as one is written in java, other one is in golang, now for making an HTTP call over the network we need to set up the HTTP Client in all services and they all uses different libraries like in Go we have net/http and so on. So this is a problem.

We need to have something that is standardized across services and that will be language independent as well. Thats what RPC brings in microservices.

RPC uses HTTP/2 internally.

## How RPC (specifically gRPC) Solves This

Remote Procedure Call (RPC)—most commonly implemented via frameworks like gRPC—addresses this exact pain point:

• Language-Independent Contracts: Instead of writing custom HTTP clients or parsing JSON ad-hoc, you define your service methods and data structures in a single, neutral interface file called a Protocol Buffer (.proto).

• Automated Code Generation: Using the protoc compiler, you can generate strongly-typed client and server code automatically for Java, Go, Python, C++, and many other languages.

• Standardized Transport: gRPC runs over HTTP/2 out of the box, offering built-in features like multiplexing, lower binary serialization overhead (Protobuf instead of JSON), and bi-directional streaming without forcing every team to hand-code an HTTP client wrapper.


# A Quick Nuance

While RPC/gRPC makes internal service-to-service communication much more standardized and typed, REST over HTTP/JSON is still widely used in microservices because it is human-readable, browser-friendly, and easier to debug with basic tools like curl or Postman. However, for strict language-agnostic internal communication and high performance, shifting to an RPC framework like gRPC is a standard industry solution.


RPC abstract out all the complexity behind making a network call and creates an abstraction such that we think that we are making a local function call but internally RPC is managing the marshalling / unmarshalling and the network packets delivery and all.

# Stub:

In Remote Procedure Calls (RPC), the stub does act as a mediator that handles the format conversion (known as marshaling or serialization) so that different services can communicate seamlessly. However, while stubs transform data into a highly efficient binary format, payload compression (like gzip or Brotli) is usually a separate network-level configuration rather than a primary job of the stub itself.

# Stub Interface Description Language: (in gRPC)

We need to write interface definition in proto format. So that there will be an interface that we need to create for RPC system.

So there will be an interface definition file in proto format (.proto) so from this definition we can convert it into any language. (String API Contract) Proto file create stubs for multiple languages.

There are stub generators that creates all these and converts for ex: if we have java as the output then it will create the java file as output if we have Go as the output then it will give me go as the output.

# Advantages:

1. Gives tasks abstraction.
2. RPC supports most of the modern languages.
3. performance (compression, multiplexing, streaming, connection pool)
4. Security is a plug (because there is abstraction so security is also there just configure them).
5. We dont need to write client libraries, stubs are automatically generated through the proto file. Ease out the multi language communication. (Ex: Slack APIs)


# Concerns while using RPCs:

1. Stubs needs to be generated when signature changes.

2. Testing RPC is not trivial.

3. Getting started is little challanging.

4. Browser support is limited. - Javascript to server 