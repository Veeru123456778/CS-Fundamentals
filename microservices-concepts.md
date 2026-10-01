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