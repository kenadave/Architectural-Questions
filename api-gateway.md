**Role of CorrelationIdFilter**

Every HTTP request that enters the gateway gets a unique X-Correlation-Id. 
That ID travels with the request across every service it touches — booking-service, flight-search-service, payment-service — and comes back in the response. 
Without it, when something fails in production you'd have logs scattered across 4 services with no common thread to link them.

# Request Lifecycle Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant Middleware as Correlation Middleware
    participant Downstream as Downstream Service / Handler

    Client->>Middleware: Inbound Request (Check X-Correlation-Id)
    
    Note over Middleware: 1. Read header.<br/>If missing, generate UUID.
    Note over Middleware: 2. Mutate request context<br/>& forward header.
    
    rect rgb(240, 240, 240)
        Note over Middleware: 4. Log on Entry:<br/>Method, Path, Correlation ID
    end

    Middleware->>Downstream: Forward Request with Header
    Downstream-->>Middleware: Return Response (Status Code)

    rect rgb(240, 240, 240)
        Note over Middleware: 4. Log on Exit:<br/>Status, Correlation ID
    end

    Note over Middleware: 3. Add same ID to Response Header
    Middleware-->>Client: Outbound Response (X-Correlation-Id)
```
@Component — marks a class so Spring auto-detects and registers it as a bean during component scan. Spring calls the constructor for you.

@Bean — a method-level annotation inside a @Configuration class. You write the construction logic yourself. 
Used when you need control over how the object is built — third-party classes you can't annotate, conditional wiring, or complex setup.

@Configuration — marks a class as a source of @Bean definitions. 
It's a specialised @Component that tells Spring "this class exists to declare beans, not to be used directly itself."



```text
  application.yml          your filters              built-in
        │                       │                       │
        │  defines where        │  enrich/validate      │  makes the call
        ▼  to send it           ▼  the request          ▼
    uri: http://        CorrelationIdFilter        NettyRoutingFilter
    localhost:8082      SecurityHeadersFilter      → HTTP POST to :8082
                        StripPrefixFilter
```


