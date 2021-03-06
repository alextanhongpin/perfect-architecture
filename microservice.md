## Designing a microservice

## Part 1: Enablers

Let's break it down to their specific functionality:

Infrastructure. Does your service has the following?
- graceful shutdown
- graceful background process/worker termination
- distributed tracing
- metrics collection
- logging
- [configuration](https://github.com/alextanhongpin/evolutionary-architecture/blob/master/configuration.md)

API. Do you follow the best practices when designing APIs?
- error handling
- envelope response (or json-api)
- validation
- secure headers
- ratelimiting
- retries 
- circuit breaker
- request id correlation

Architecture. What is the flavour of your service?
- worker
- api server
- web server
- distributed cron (when running multiple instances with a cron, we want the leader to handle the cron only)
- synchronous - request/response
- asynchronous - message queue, pub/sub
- service pattern
- repository pattern
- mvc pattern
- onion architecture

DevOps
- logging + errors
- service discovery
- load balancing
- schema validation
- database migration
- deployment
- build server strategy
- dockerization
- app sensor
- [health endpoint](https://github.com/alextanhongpin/evolutionary-architecture/blob/master/health.md)

Entity vs Data Transfer Object (DTO)
- only rules that concern that particular object
- usually implemented in OOP style

- data transfer object (DTO). DTO should contain only private fields for your data, getter setters and constructors. DTO should not have business logic - though some util methods should be okay. They are immutable (?).
- rich domain model 
- value object has behaviour (for read). [Value object is not a DTO](https://en.wikipedia.org/wiki/Data_transfer_object). More on value objects [here](https://martinfowler.com/bliki/ValueObject.html).
- anemic data model. They are purely data structures. Almost no logic is implemented there. Similar to DTO?
- data access object (DAO). DAO is a class that has the CRUD operation, while DTO is just an object that holds daa.

## Part 2: Principles

SOLID
- Single responsibility principle (SRP)
- open closed principle (OCP)
- liskov substitution principle (LSP)
- interface segregation principle (ISP)
- dependency injection principles (DIP)

Others
- Reuse/Release equivalence principle (REP)
- common closure principle (CCP)
- Common reuse principle (CRP)
- Acylic dependency principle (ADP)
- Stable dependency principle (SDP)
- stable abstraction principle (SAP)

## Part 3: API

Think in terms of rest and resources. 

Design a scalable, reliable Api.

Say we want to write a url shortened service, it’s different. Is it public, or private. Note, it is different for different kind of apis. If it is resource based, the we can apply rest. What if it is capability based? Which is more soap like? E,g auth microservice. Capability urls. 

Say we have a resource, book. We can have the following crud implementations:

Create book
	- create
	- createList
Get books
	- find GET /books/id
	- all GET /books
	- where GET /books?author=john
Update books
	- update
	- updateAll
Delete book
	- delete
	- deleteAll

Not all API must be based on resources. We may also have capability urls (reset password, search etc). Note that we do not hold assumptions about the versioning. See versioning for more details.

## API Versioning

When in doubt, use v1.
- https://api.server.com/v1/books


That is the basic crud. Let’s create the model first, book. This is how our book will look like.
Then let’s create the stub routes. Router.get
Then we create the controllers actions. We setup our services, and we are done. 

# Controllers

Controllers consist of actions. Controller are normally grouped by the resource they belong to. So a BookControllers may consist of a bunch of actions that are operating on the BookService.

function BookController(bookService) {
	return {
		getBooks,
		createBooks
		updateBook
		deleteBook
	}
}

This controllers will be plugged to a router. Again, we can group them by the domain that they belong to.

router.get(‘/v1/books’, controller.getBooks)
router.post(‘/v1/books’, controller.createBooks)

Additionally, we may add additional middleware such as authentication, logging, error handling, metrics collection etc.

const withAuth = router.use(logger).use(auth)…
router.post(‘/v1/books’, middleware.auth({type: ‘basic’, username: ‘john’, password: ‘123456’, hash: ‘xyz’ }), controller.createBook)
router.post(‘/v1/books’, middleware.auth({type: ‘jwt’, secret: ‘xyz’}), controller.createBook)



Next step. 
- refactoring, 
- Handling multiple domains
- Security, with helmet
- Models
- Data transfer object
- Repository
- Controller, actions
- Middlewares
- Database placed in container

Business logic should belong to the domain, and application logic should belong to the controller
What does this means?
The logic to extract params from header, body, query string etc are part of the application logic, hence they should belong to the controller. However, creation etc validation should all belong to the model. By model, it can be just a plain object ( or gaoling struct), but it has nothing to do with persistence (repository). 
https://softwareengineering.stackexchange.com/questions/176639/why-put-the-business-logic-in-the-model-what-happens-when-i-have-multiple-types


Put interfaces in domain, place implementation in packages. Don’t leak implementation details.

Domain
- Describes the application logic - what it does
- For our shortUrl application, we call our domain the ShortURL
- For each domain, we will normally have the following components/interfaces/abstraction
    - Entity: the representation of the subject, does not have to be but is usually a mapping to the database table
    - Repository: the layer that manages the storing and querying and mapping of the entity from the storage layer. This layer allows the db to be mocked.
    - Service: the operations that cannot be performed by the entity themselves are handled here. Services usually takes in a Data Transfer Object (DTO) rather than the entity themselves, since the most operation will concern on just a part of the entity, and not all the fields.
    - Statements. An addition to the repository layer, this is essential to validate the sql stmts, since we are not using an orm. 
    - DTO: The selected fields of the entity to operate on.
- There are no implementations in the domain layer, rather only interfaces that describes the input/output of the each layers


Router
- This is normally the only part that changes for most application
- Each router will have a controller
- Each controller will have actions
- Each actions calls the service
- The router implements the http.handler
- This makes it easier to perform graceful restarts for any choice of router, be it http router, gorilla, default new serve mux, gin or even chi-router.

## Model

Designing model. How do we start from there? Simple, we follow our database mapping one to one. Then we think of what methods can be required. Eg is a method that generates full name from last name. Or if we have a list of orders, then we may have a method that computes the sum. Depending on the use cases, we may choose to perform it in the db layer, since it’s expensive for the memory to withhold all the individual items. 

Next we think about the validation. Note that we may have to stick with different validation strategy. We can’t make all fields here required, because they may conflict when we use it for different endpoint, that’s where dto comes into place. 

## Model vs dto.
 As mentioned, the model differs from out dto. Our model may map one to one to out database table, but most of the time we want to return a different representation of the aykoad back to the users for the following reasons

- sensitive fields like password, user id
- Virtual fields that are computed, such as age, total expenses. Note that this depends entirely on the field and client side presentation. It is for example better to return dates as rfc3339 without formatting since the client may decide how to present it.
- Additional fields such as full name that is derived from first name and last name, full urls that are derived from base url
- Additional metadata from other entity!
- A list of entity than one entity
- Other entity
- Other data that is not an entity
- Pagination offset etc, next url
Since different response may require different representations, we can define them individually. Else, we can create another model, called simple model. E.g. simpleUser. Else we can choose to use PublicUser. I Golang, there are additional tricks to shadow the variables, but let’s not get into it.

## middleware

Middlewares adds capabilities to an endpoint. I will treat them as infra related(logging, metrics collection, request is injection etc) rather then business logic related (is auth middleware considered business logic?). Most middleware are enablers, they provide a way to extract/inject data that can be used in the next pipeline of the requests, e,g, auth middleware, request I’d middleware, the rest are mainly tracers, such as metrics gathering.

## Controller

Some of the responsibility of the controllers is to decode request, calls service, log errors, and encode response (that includes error).

- Decode requests. There are several ways to decide a requests, they can be from a JSON body, query strings, url parameters or http headers. Most of the time, we need to decide it back to a Golang struct which is typically a data transfer object, they are anemic data models, in which their sole responsibility is to transfer data, not entity or model. They do not represent an entity, but can be mapped to an entity once decoded.
- Calls service. This is where the service is executed with the requests returned from the decoder. A service accepts a requests, and returns a response or an error. If there are errrors, we log the requests along side after removing the sensitive data (password). Yes, we have to be selective of the data we log. 
- Log errors. Enuff said.
- Encode response, for JSON Api, this is normally a JSON body, if it is a html site, it can render a template or in some cases perform a redirection.

## Services

Trim requests,  validate requests, calls model/repository, return response and error.

- trim requests, it is common for the client side application to send strings with white spaces at the end. This can have serious effect on your application, say even a difference of white space in password can cause the login to fail. Your text might...But having to trim each fields is a waste of time. So Golang pkg that does this...
- Validate requests. The most crucial part of the application- validation. Don’t trust the client. Handle all validation on the server side, define what is required and what is optional. Then define the boundaries. This is essential for the business logic to operate, and to create reliable services. Why validate here and not the controller? Because this is the service later, it should hold the business logic. Also, we want to test the business layer, not the controllers. We have the service validate the requests as early as possible, so that subsequent layers no longer require the validation. Think of it as a gateway, once i pass through here, i have permission to operate on all other stuff. 
- Calls models/repository, this is where the orchestration of business logic takes place. Entities can perform an action, and they can call the repository to provide data. If there are no business logic, then your service layer will most likely be an additional service, in the scenario of pure CRUD application. 
- Return response and error, this is a tricky part, this is where we will return the responses of the data to the client. If it is a JSON Api, we will normally return a payload. In Golang, returning a nil slice will not return a JSON array, but null value. Also we envelope our response in a data object, see why. 

## app 

Is the core of your microservice. It does the following.
Initialise dependency, starts the server
Gracefully terminate the server
Terminates the dependency


## infrafactory
This is particularly interesting, since there are many ways to go about this. We can have our services initialise the dependencies directly when the app starts or lazily initialise them, when they are called,p. Why do we have this infra layer? Most of our services requires passing of dependencies (db etc) in order to initialise them, over time, it gets big. A simple way is to define a service factory that takes the infra and returns the service. 

What do we want to initialise early.
- db, because of prepared statement. 
- Anything that we want to fail fast

Why lazy load
- there are a series of builds that requires the same dependency, and we do not want to trouble ourselves with the sequence of initialisation (which can be considered a tight coupling).


## Repository
Select the fields required for query/exec. 
Map the database columns back into entity when necessary.

This hides the implementation details for querying the db. We only define the interfaces of the database operations as well as their input and output. Note that the input and output can be the entity domain, there are two ways to write a repository
```
Users. All, withId, withEmail...fluent way
Repository, getById, getByEmail, getUsers
Users, fetchAll, fetchOneById, fetchOne...find,
List, one, all, create, update, delete
```
It doesn’t matter which way you choose, but be consistent. 

Having an interface layers helps us solve a lot of issue, easier to mock
- testing is easy, since we can mock the repo layer
- Swapping implementation is easy, like in memory vs Postgres vs sql. This only works as if the interface is really the same. Changing to a nosql might require a different interface. 


## Microservices best practices


### When dealing with multiple associations

- Operations should obey single responsibility principles, e.g. updating an entity should not affect the associations (deletion, creation)
- Update should not include delete operation, and vice versa
- When deleting multi associations, use the database cascade
- When dealing with operations such as uniqueness, do not check on the application level - instead, do it at the database layer. This is because when the services scales beyond a single node, and there are concurrent activities, the uniqueness is not longer achievable (multiple inserts at the same time)



- microservices are usually composed of different layers (controllers, service, repository).
- each layers usually have different responsibility, as well as different input and output that is required on order to operate
- sometimes they are the same, which means we will be creating duplicate input and output types
- Different layers, different input and output
If controller, one can stick with request response
For services, stick with dtos, they should produce an entity output
For repository, stick with entity or partial entity with meta (pagination, filtering and sorting), for output stick with entity and error

- different input and output calls for different validation
- for most scenario, we want to assume that all inputs are required, and default values should be populated
- service layer must have a validation, because they may be reused by another service
- Validation happens at top layers but can be multiple. E.g. controller validation for request response, and service validation. Why duplicate validationat service, because sometimes we want to reuse the service, and we need to validate the response. that is why controller request has to be mapped to service dto.
