# Microservices Software Architecture
###### patterns & technique
___
##### Characteristics
 * Services are 
      * fine grained
      * lightweight  
    
##### Advantages 
 * Low coupling 
 * Improves modularity
 * Promotes parallel development
 * Promotes scalability
##### Drawacks
  * Infrastructure costs are usually higher
  * Integration testing complexity
  * Service management and deployment
  * Nanoservice anti pattern i.e where a service is to fine grained 
##### Why do many microservice project fail ?
  * Lack of 
      * Planning 
      * Knowledge
      * Skills
      * Time
##### How to prevent your projects from failing ?
  * Determine applicability
  * Prioritise automation
  * Have a clear plan
  * Avoid common pitfalls
### Microservice Template
 This is a code project that can be used to start off from when developing a new microservice to save time.
 ##### Why is important ?
   * Significant amount of time setting up
   * Similiar code for each microservice setup
##### What should the template contain ?
  * Cross cutting concerns
     * Logging
     * Metrics
     * Connection setup and configuration to databases and message brokers
  * Project structure
### Code Repository Setup
  * Mono : where all code will be in a single repository
  * Discrete : where codebase is split into seperate repository
__Mono Repo__ . 
__*Pros*__ . 
1. Easier to keep input/output contracts in sync
2. Can version the entire repo with a build number
__*Cons*__ . 
1. Different teams working in the same repo can break the build, disrupting CI/CD for other teams
2. Easier to create tight coupling
3. Long build times, large code repo to download
__Discrete Repo__ . 
__*Pros*__ . 
1. Different teams can 'own' different repositories
2. Scope of a single repo is more clear
__*Cons*__ . 
1. Contract versioning becomes more complex
2. Unless managed properly , discrete repositories can easily become monoliths
3. More up front cost in setting up repos and CI/CD pipeline
##### Microservice Decompositon
They should be loosely coupled from each other but should have high cohesion i.e each microservice will contain only things that are strongly related to each other. Remember the common closure principle which states that things that often change together should often close together. The most widely used tactics is through business cases, technical capabilities or function objective to make sure a suitable microservice has a level of granularity eg . Order Management , Shopping Cart Management
__*Order Management*__ can be decomposed into the following microservices : 
 * Order history
 * Order tracking
 * Order placement
 * Order dispute  
  
__*Shopping Cart Management*__ can be decomposed into the following microservices : 
  * Cart Upselling
  * Cart Promotions
  * Cart Cost Calculator
  * Cart Recovery
### Microservice Communication
 Inter-service communication
 1. Remote Procedure Invocation : is simple and easy to understand and can be used using REST and Apache thrift. It follows the Request/Response pattern ie. synchronous
2. Asynchronous Message Based Communication : in a scenario where response is not immediately required eg using Messaging , a microservice can publish a message on the message bus for other services to consume. It follows publish/subscribe pattern. This can however lead to over complexity. It is always advisable to stick to synchronous communication except in-cases where the communication must be asynchronous
##### Microservice Registry
This is an extremely important component required in microservice architecture in order to be able to support dynamic scaling.
If one microservice needs to send request to another microservice it needs to be aware of the available instances and their network. The number of instances of each microsrervice maybe scaled dynamically to adjust changed in node therefore other services communicating with this services must be aware of this changes.
To solve this problem , we introduce a service registry component that holds the current available instances of the microservice and their network location.

*How it works* 
When a microsevice starts it registers itself with the service registry which will add this microservice to it database same in a shutdown instance which triggers a remove from the service registry.
Then at regular intervals we can introduce a health check api which performs a health check at regular intervals. 
When a microservice is required the service registry is queried for the available instances and the network location.
##### Microservice Discovery
This talks about how services are able to query the service registry either directly or in-directly.

*Client Side Discovery*
This is when a service directly queries the service registry to obtain network location for an instance of the required service and the service registry replies back the network location information and the caller uses this information to call the instance
*Server Side Discovery*
Here the microservice performing this request have no knowledge of the service registry , it simple send the request to a load balanced endpoint and the load balancer will query the service registry for an instance and network location of the required microservice endpoint.

The Server Side has an advantage over the Client Side discovery because clients do not need to query the service registry while the disadvantage is that there are more network hops involved before the request arrives at the microservice destination this can be mitigated against by building the service registry directly in the load balancer

### Data 
*DataBase Patterns*
Shared Database

Order Placement Microservice   
Customer Details Microservice
Product Details Microservice 
all using the same database in this case DB transaction is used to guarantee data consistency and integrity. There is possibility of performance issue due to deadlocks also schema change too by another team

Database Per Service
Order Placement Microservice   
Customer Details Microservice
Product Details Microservice 
all have their own seperate databases. Different service can have different database technologies that base suit their requirement eg. using SQL database , NOSQL  
It is difficult getting aggregate data across services however this can be achieved by API composition or event store
##### API Composition
A Service referred to as the API composer queries data from the multiple services and then performs an in-memory join
##### Event Sourcing
It helps services keep track of state changes of object in a reliable way. The difference between event sourcing and using a database directly is that we use an EventStore and persist objects to it. Services are able to subscribe to the different events handled by the event store in this way the event store acts as a message broker.
##### Two Phase Commit
A distributed transaction implies altering data on multiple databases which arises mostly on database per service pattern as a result this becomes complex as the commit/rollback of data must be coordinated in a transaction as a self-contained unit as a result the entire transaction commits or rollback.

    Phase 1 : Commit Request
         - Coordinator service sends a query to commit message 
         - Services execute the transaction but do not commit
         - Reply Yes/No depending on if they were successful
    Phase 2 : Commit 
    -if all services replied yes
         - coordinator sends a commit message
         - Services commit the transaction
         - Reply with an acknowledgement
    - If at least one service replied with no 
         - Coordinator sends a rollback message
         - Services rollback the transaction   

Disadvantage is that this is a synchronous operation which might result in the blocking because the services will have to wait for the coordinator on instructions to proceed also if the coordinator goes down the services hang indefinitely
##### Saga Pattern
is an alternative to 2 phase commit to manage distributed transactions , it is a sequence of local transactions in different microservices each local transaction updates the database of that microservice and then publishes an event or a message to trigger the next local transaction in the saga.
If one local transaction fails the saga execute a series of compensating transactions that rollback changes of local transaction forming part of the distributed transaction that have already been executed. 
There are 2 main different types of Saga implementation
1. Choreography-based sagas
     Were each local transaction publishes domain event that will trigger local transaction in other service until the saga is completed
2. Orchestrator-based sagas
     An orchestrator which is usually created for each saga. It coordinates the whole saga. This 
     orchestrator is als a microservice on it own. It lets each microservice in the saga known when to be called or to be rollback if any of the microservice fails.
### Fault Tolerance and Monitoring
It is important to have reduncancy and high availability. We must also have a reduntant service registry and the reduntant microservice hosted on another server entirely. If a service tries to connect to a service and it is unable to connect it should immediately connect to the reduntant service and also create a log issue when the fail over resource has been used so that the main service can be fixed..
##### Circuit breaker pattern
 this helps prevent failures in some part of the network or in a microservice from bring down the system. 
 For more information on [Circuit Breaker Pattern](https://microservices.io/patterns/reliability/circuit-breaker.html)
 It should be a cross cutting module in the microservice architecture
 ##### Health Check API
 Used to get status of a service. This is usually used by the service registry to check on register services to check if they are up and running frequently. 
 ##### Logging Technique
 Always tag a request with a GUID for logging on the diferrent microservices or better to use a log aggregation technology eg elastic , logstash , kibana stack ,  aws cloud watch , splunk








   
