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
