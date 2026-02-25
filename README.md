# TradeX Microservices Architecture

TradeX is a robust, scalable, and resilient microservices-based trading platform. 
It is engineered with strict production standards, prioritizing financial data integrity, fault tolerance, and comprehensive observability.
The system comprises seven specialized microservices working in concert to deliver a seamless trading experience.

## Repository Map
This platform is composed of 7 isolated microservices:

* [API Gateway](https://github.com/AnandhuSajikumar/tradex-api-gateway)
* [Service Discovery (Eureka)](https://github.com/AnandhuSajikumar/tradex-eureka-server)
* [Central Config Server](https://github.com/AnandhuSajikumar/tradex-config-server)
* [Identity Service](https://github.com/AnandhuSajikumar/tradex-identity-service)
* [Transaction Service](https://github.com/AnandhuSajikumar/tradex-transact-service)
* [Portfolio Service](https://github.com/AnandhuSajikumar/tradex-portfolio-service)
* [Market Service](https://github.com/AnandhuSajikumar/tradex-market-service)

System Architecture

The project is structured into seven distinct microservices, divided into infrastructure and domain services.

Infrastructure Services

1. API Gateway (tradex-api-gateway)
   - How it works: Built using Spring Cloud Gateway. It utilizes dynamic routing configurations fetched from the Central Config Server. It incorporates Resilience4j circuit breakers on all downstream routes and features a global fallback controller.
   - Why it exists: It serves as the single entry point for all client requests, hiding internal system complexity. The integrated circuit breakers prevent cascading failures; if a domain service is unresponsive, the Gateway short-circuits the request and returns a clean, standardized JSON response (e.g., a graceful fallback or cached state) rather than hanging indefinitely.

2. Service Discovery (tradex-eureka-server)
   - How it works: Implemented via Spring Cloud Netflix Eureka. All domain services and the Gateway act as Eureka Clients, registering their instances upon startup.
   - Why it exists: It enables dynamic service registration and discovery. Instead of relying on hardcoded IP addresses, services communicate using logical names. This fundamental decoupling allows the platform to scale horizontally and dynamically balance loads across multiple instances of a service.

3. Centralized Configuration (tradex-config-server)
   - How it works: Developed using Spring Cloud Config. It serves properties from a shared configuration repository to all microservices gracefully upon boot.
   - Why it exists: It externalizes configuration from the codebase. This allows environment-specific settings to be managed and updated centrally without requiring code recompilation or service redeployment, ensuring configuration consistency across the entire cluster.

Domain Services

4. Transaction Service (tradex-transact-service)
   - How it works: Orchestrates trading actions. It integrates deeply with Resilience4j for inter-service communication. It features a strict Idempotency layer using immutable IdempotencyKey database entities and implements a compensation workflow for distributed transactions.
   - Why it exists: It handles the most critical and complex business logic - financial transactions. The Idempotency layer guarantees that duplicate network requests or retries never result in double-charging a user or executing duplicate trades. The mini-saga pattern ensures that if a downstream dependency fails during a trade, local state changes are reliably rolled back to maintain absolute financial consistency.

5. Identity Service (tradex-identity-service)
   - How it works: Manages authentication, user onboarding, and role-based access control. It establishes the precedent for strict, standardized error-handling using custom ApiError formats.
   - Why it exists: It secures the platform. Extracting identity management into its own service isolates sensitive user credentials from general business logic. The standardized exception handling ensures that internal stack traces are never leaked to external clients, presenting only secure and actionable error responses.

6. Portfolio Service (tradex-portfolio-service)
   - How it works: Manages user financial accounts, cash balances, and equity holdings. It processes debit/credit commands issued by the Transaction Service.
   - Why it exists: Separating asset management allows for independent scaling. Portfolio valuation is heavily read-optimized, whereas transaction processing is write-heavy. This separation of concerns prevents high query volumes from impacting execution performance.

7. Market Service (tradex-market-service)
   - How it works: Acts as the pricing engine, providing real-time or cached stock market data.
   - Why it exists: It establishes a single source of truth for market conditions. Isolating pricing logic ensures that the high-throughput nature of querying stock prices does not compete for resources with the core transactional databases.


Cross-Cutting Concerns

Throughout the architecture, specific patterns have been implemented to elevate the platform to strict production standards:

Observability and Distributed Tracing
   - How: All services are instrumented with Spring Boot Actuator, Micrometer Prometheus, and Micrometer Tracing. Trace sampling is set to 100%. Health and Prometheus metric endpoints are exposed globally.
   - Why: In a distributed environment, a single user request traverses multiple services. Injecting traceId and spanId across HTTP boundaries allows developers to follow the complete lifecycle of a request in centralized logging. Prometheus metrics provide the foundation for automated alerting and proactive health monitoring.

Error Handling Protocol
   - How: Custom exception classes immediately map to standardized immutable ApiError payloads.
   - Why: Prevents information leakage and provides front-end consumers with predictable contract models to handle failures gracefully.

Resilience and Self-Healing
   - How: Aside from gateway-level circuit breaking, inter-service clients utilize Resilience4j components. Retries are strictly configured to target only specific server errors or timeouts, explicitly avoiding blind retries on client errors.
   - Why: This defends the network from retry storms. By controlling exactly when and how services retry or fail, the platform remains highly available and fail-safe during temporary infrastructure outages.
