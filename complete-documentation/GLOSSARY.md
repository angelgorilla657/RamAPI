# Glossary

Definitions of terms and concepts used throughout RamAPI documentation.

## A

**Adapter**
: A bridge between RamAPI's core routing system and the underlying HTTP server implementation (e.g., Node.js HTTP, uWebSockets.js).

**API (Application Programming Interface)**
: A set of protocols and tools for building software applications.

**Authentication**
: The process of verifying the identity of a user or system.

**Authorization**
: The process of determining what an authenticated user is allowed to do.

## C

**Context (ctx)**
: An object passed to route handlers containing request information and response methods. Replaces `req`/`res` in Express.

**CORS (Cross-Origin Resource Sharing)**
: A mechanism that allows restricted resources on a web page to be requested from another domain.

**CRUD**
: Create, Read, Update, Delete - the four basic operations of persistent storage.

## D

**Distributed Tracing**
: A method of tracking requests as they flow through distributed systems, collecting timing and metadata.

**Dynamic Route**
: A route with parameters (e.g., `/users/:id`) that matches multiple URLs.

## E

**Exporter**
: Component that sends telemetry data (traces, metrics) to external systems (e.g., Jaeger, Prometheus).

## F

**Flow Tracking**
: RamAPI's visualization system that shows the execution timeline of requests.

**Flow Visualization**
: Visual representation of request execution, including waterfall charts and sequence diagrams.

## G

**Graceful Shutdown**
: Process of stopping a server while allowing existing requests to complete.

**GraphQL**
: A query language and runtime for APIs that provides a complete description of the data.

**gRPC**
: High-performance RPC framework using Protocol Buffers.

## H

**Handler**
: A function that processes a request and sends a response.

**Health Check**
: An endpoint that reports the application's operational status.

**HTTP Error**
: An error object with an HTTP status code (e.g., 404, 500).

**HTTPMethod**
: HTTP request methods: GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD.

## J

**Jaeger**
: Open-source distributed tracing system for monitoring microservices.

**JWT (JSON Web Token)**
: A compact, URL-safe means of representing claims to be transferred between two parties.

## M

**Middleware**
: Functions that execute during the request-response cycle, before the final handler.

**Multi-Protocol**
: Support for multiple communication protocols (REST, GraphQL, gRPC) in one application.

## O

**Observability**
: The ability to measure a system's internal state by examining its outputs (traces, logs, metrics).

**OpenTelemetry**
: Open-source observability framework for traces, metrics, and logs.

**OTLP (OpenTelemetry Protocol)**
: Protocol for transmitting telemetry data.

## P

**Profiling**
: Process of measuring performance characteristics to identify bottlenecks.

**Prometheus**
: Open-source monitoring and alerting toolkit.

## R

**Rate Limiting**
: Controlling the rate at which users can make requests to prevent abuse.

**REST (Representational State Transfer)**
: Architectural style for distributed hypermedia systems.

**Router**
: Component that maps URLs to handler functions.

## S

**Sample Rate**
: Percentage of requests to trace (e.g., 0.1 = 10% of requests).

**Schema**
: Definition of data structure and validation rules.

**Span**
: A unit of work within a trace, representing a single operation.

**Static Route**
: A route with a fixed path (e.g., `/users`) that only matches exact URLs.

## T

**Trace**
: A record of a request's journey through a system, composed of spans.

**Trace Context**
: Information propagated through a distributed system to correlate spans.

**Trace ID**
: Unique identifier for a trace (32 hex characters).

**TypeScript**
: Typed superset of JavaScript that compiles to plain JavaScript.

## V

**Validation**
: Process of ensuring data meets specified criteria before processing.

## W

**Waterfall Chart**
: Visual representation showing the timeline of operations in sequence.

## Z

**Zod**
: TypeScript-first schema validation library used by RamAPI.

---

## See Also

- [FAQ](FAQ.md)
- [Core Concepts](core-concepts/request-lifecycle.md)
- [Observability](observability/tracing.md)
