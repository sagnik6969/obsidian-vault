
### Monolithic App
- 1 application handles everything.
- for example in an e commerce app single backend handles payments , orders, notifications all. 

### Disadvantage:
- Overloads IDE. 
- Takes a lot of time to run tests in ci since all the test-cases should run.
- If payments has high load. payments service cant scale independently. whole application has to scale.
- Veri difficult to maintain

### Microservices:
- Application is decided into smaller backend apps. 
- 

### Advantage of Micro-service
- Easy to manage each application separately. 
- Each app can scale independently.

### Disadvantage of micro-services
- If micro-services are not divided properly it may cost a lot of money. 
- Microservices should be loosely coupled. 
- If systems are tightly coupled, a service has to communicate with a lot of other services to complete a request then latency can go up because here different services are communicating over HTTP instead of direct function calls in monolithic architecture.
- Latency can increase if we don't divide the monolithic system properly. 
- Difficult to debug issues if request is dependent of a number of micro-services where exactly the issue occurred. 
- Transaction management is very difficult. If a request is processed my 2 micro-services with there own DB then it is very difficult to manage transaction (If update in one table successful and 2nd table fails).

# Different steps of breaking monolithic apps into micro services
## Decomposition:
there are different patterns based on which we can decompose monolithic apps into small micro-services. 
### Decompose Patterns
#### 1. Decompose by business capability:
This is the most common and intuitive approach. It aligns the architecture directly with the organization's business model and structure. A business capability is something a business does to generate value.

- **How it works:** You identify the core functions of the business and create a service for each. These services are largely autonomous and own their specific data.
    
- **Example:** In an e-commerce platform, you would have separate services for `Order Management`, `Inventory`, `Shipping`, and `Customer Accounts`.

#### 2. Decompose by subdomain:
Rooted in Domain-Driven Design (DDD), this pattern is highly effective for complex, enterprise-level systems where "business capabilities" might be too broad.

- **How it works:** You break the overarching domain into subdomains and map them to "Bounded Contexts." Each bounded context becomes a microservice.

#### 3. Decompose by Resource or Technology Requirements

Sometimes, the boundaries are dictated by infrastructure rather than pure business logic.

- **How it works:** Services are split based on their specific compute, memory, or language requirements.
    
- **Example:** You might have a lightweight, user-facing API built in Node.js or React that requires fast response times but low compute. Meanwhile, you have an AI inference engine or data processing pipeline built in FastAPI and Python that requires heavy GPU utilization. Separating these allows you to orchestrate and scale them independently on a platform like Kubernetes without wasting resources.

#### 4. The Strangler Fig Pattern (For Migrations)

This pattern isn't for greenfield projects; it's a migration strategy for decomposing an existing monolith safely over time.

- **How it works:** You place an API Gateway or routing facade in front of the legacy monolith. You then gradually extract specific functionalities into new, independent microservices. The router sends traffic to the new service for the extracted feature, and routes everything else to the monolith. Over time, the new microservices "strangle" the monolith until it can be decommissioned.

#### 5. Decompose by Volatility (or Change Frequency)

This pattern focuses on developer velocity and minimising deployment risk.

- **How it works:** You isolate the parts of the system that undergo rapid, constant iteration from the highly stable, critical core systems.
    
- **Example:** A system's user interface layer or recommendation algorithm might be tweaked and deployed daily. Conversely, the core financial ledger or payment gateway might only be updated twice a year. Separating them ensures that a bug introduced during a fast-paced UI update doesn't bring down the payment processing system.

## Database:
- Separate database per service.
- Shared database. 

## Communication
- How the services will communicate
## Different methods of communications:
- via api
- via events

## Integration
- How different services are integrated with Frontend. 

## Monitering & Logging

# Interview Question:
- What is advantage and disadvantage of micro-services & Monolith. 
- 