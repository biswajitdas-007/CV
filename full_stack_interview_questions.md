# The Ultimate Full-Stack Interview Question Bank (5-6 Years Experience)
*Target Stack: Java, Spring Boot, Node.js, React, SQL/NoSQL, System Design (LLD/HLD)*

---

## Part 1: Core Frameworks & Runtimes

### 1. Java & Spring Boot
#### Java Core & Concurrency (5+ YoE Expectations)
1. Explain the internal working of `ConcurrentHashMap` in Java 8+. How does it achieve thread safety without locking the entire map?
2. What is the difference between `reentrantLock` and a standard `synchronized` block? When would you prefer the former?
3. Explain the Java Memory Model (JMM). What are the roles of the Stack, Heap, Metaspace, and the volatile keyword in preventing instruction reordering?
4. How do Virtual Threads (Project Loom) differ fundamentally from platform threads regarding memory footprint and OS thread blocking?
5. How would you diagnose a memory leak in a production Java application? Which tools (e.g., JProfiler, VisualVM, Eclipse MAT) and JVM flags would you use?
6. Explain the difference between G1 GC and ZGC. In what scenarios would you optimize for low latency versus high throughput?
7. How does CompletableFuture handle asynchronous exception chaining? Walk through `exceptionally()`, `handle()`, and `whenComplete()`.
8. What is the difference between `fail-fast` and `fail-safe` iterators? Provide internal structural examples.
9. How do you implement a custom ThreadPoolExecutor? How do you choose the ideal core pool size, max pool size, and queue capacity for an I/O-bound vs. CPU-bound application?
10. Explain the structural behavior of ThreadLocal variables. How can they introduce memory leaks in an application server like Tomcat?

#### Spring Boot & Microservices
11. Explain the Lifecycle of a Spring Bean from instantiation to destruction. Where do `BeanPostProcessor` and `@PostConstruct` fit?
12. What is Bean Circular Dependency in Spring? How does Spring resolve it using three-level caches, and when does it fail?
13. Explain `@Transactional` propagation behaviors. What happens when a method with `REQUIRED` calls a method with `REQUIRES_NEW` or `NESTED`?
14. How does transactional proxying fail due to self-invocation within the same Spring Bean? How do you fix it?
15. Explain the dynamic architectural differences between Spring Cloud Gateway and Netflix Zuul.
16. How do you configure a resilient Circuit Breaker using Resilience4j? Explain the transitions between Closed, Open, and Half-Open states.
17. How does Spring Boot Auto-configuration work under the hood? Explain `@ConditionalOnClass`, `@ConditionalOnMissingBean`, and `spring.factories`.
18. How do you secure microservices using OAuth2 and JWT with Spring Security? How do you manage stateless session validation at the gateway level?
19. What is the difference between `@ControllerAdvice` and local `@ExceptionHandler`? How do you structure a global error response layout?
20. Explain the architectural setup and benefits of Spring WebFlux over standard Spring MVC for high-throughput, non-blocking I/O.

### 2. Node.js Ecosystem
#### Event Loop & Runtime Diagnostics
21. Walk through the 6 phases of the Node.js Event Loop in exact sequential execution order. Where do `process.nextTick()` and microtasks run?
22. How does Node.js handle heavy CPU-bound computational tasks without blocking the main event loop? Compare Worker Threads vs. Clustering vs. Child Processes.
23. Explain how the `cluster` module utilizes round-robin load balancing across multiple CPU cores.
24. Explain V8 Engine Memory Management. What are the limits, how does the Scavenge/Mark-Sweep GC operate, and how do you profile leaks using heap snapshots?
25. What is the difference between `Stream.pipe()` and manual event handling (`on('data')`)? How do you prevent backpressure issues in streams?
26. How do you handle unhandled promise rejections and uncaught exceptions safely in a production Node.js environment without leaving the app corrupted?
27. Compare the performance, security, and scoping behaviors of CommonJS (`require`) versus ES Modules (`import/export`).
28. Explain the difference between `Buffer.alloc()` and `Buffer.allocUnsafe()`. What are the hidden security risks of using the unsafe method?
29. How do Libuv thread pools manage asynchronous file system operations when OS kernels don't provide native async I/O drivers?
30. How would you optimize a Node.js API that processes massive multiline CSV file uploads to keep memory consumption under 50MB?

#### Express / NestJS Framework Architecture
31. How does middleware chaining work in Express under the hood? Explain the mechanical design of the `next()` function call.
32. Explain the execution context sequence of Interceptors, Guards, Pipes, and Exception Filters in a NestJS request lifecycle.
33. How do you build a robust, scalable multi-tenant architectural filter in Express/NestJS to isolate database connections dynamically based on request headers?

### 3. Frontend Architecture (React)
#### Advanced Component Engineering & State
34. Explain the React Fiber architecture. How does it break down the reconciliation phase into incremental, interruptible chunks?
35. What is the exact difference between the Render Phase and the Commit Phase in React? Which lifecycle methods or hooks execute in each?
36. Deep dive into React's Diffing Algorithm. How do component keys affect state preservation, unmounting, and element DOM updates?
37. How does React handle Synthetic Event pooling and event delegation at the root container level in React 17+?
38. Explain how `useMemo` and `useCallback` cache values/references internally. How do you avoid performance anti-patterns by overusing them?
39. What are the common causes of React Context performance degradation? How do you optimize large context state trees to prevent global re-renders?
40. How do you implement code-splitting and lazy loading natively using `React.lazy` and `Suspense`? Explain the bundle compilation strategy behind it.
41. Compare React state management architectures: Context API vs. Redux Toolkit vs. Zustand. When does a slice-based approach outperform a single-store proxy?
42. How does `useEffect` cleanup execution work? Explain the memory footprint lifecycle when dealing with active event listeners or open WebSockets.
43. How do you write a production-ready custom hook to handle robust API fetching with automated retry mechanisms, caching, and race-condition cancellation?

#### Micro-Frontends & Performance Optimization
44. Explain the architecture of Micro-Frontends using Webpack 5 Module Federation. How do apps share dependencies dynamically at runtime?
45. How do you audit and optimize Core Web Vitals (LCP, FID, CLS, INP) for a highly interactive, enterprise-grade React SPA?
46. Explain the operational differences, SEO impacts, and infrastructure trade-offs between CSR, SSR (Next.js), and Static Site Generation (SSG).

---

## Part 2: System Design & Architecture

### 1. Low-Level Design (LLD) & Object-Oriented Design (OOD)
#### SOLID Principles & Design Patterns
47. Provide a practical production example where breaking the Interface Segregation Principle (ISP) ruins API client maintainability. How do you fix it?
48. Explain the Liskov Substitution Principle (LSP). How does subclassing a square from a rectangle break LSP? What is the correct object-oriented fix?
49. Implement the Strategy Pattern and Factory Pattern concurrently to build an extensible payment processing module supporting Stripe, PayPal, and Crypto.
50. How does the Observer Pattern differ structurally from a modern Pub/Sub architecture? Show code representations for both.
51. Design a thread-safe, double-checked locking Singleton pattern in Java that is entirely safe against Reflection, Serialization, and Cloning attacks.
52. Write an elegant implementation of the Decorator Pattern to dynamically add logging, encryption, and compression layers onto an I/O data stream.

#### Machine Coding / Practical Architectural Patterns
53. **Design a Concurrent Rate Limiter:** Code a fully functional backend rate limiter class implementing either the Token Bucket or Leaky Bucket algorithm. Ensure thread safety without global blocking.
54. **Design a Thread Pool:** Write a custom implementation of a blocking task queue and thread worker execution loops from scratch.
55. **Design a Parking Lot System:** Provide class blueprints, database schemas, and clean code for a multi-story parking system prioritizing SOLID design.
56. **Design a Movie Ticket Booking Engine (e.g., BookMyShow):** Focus heavily on the exact class structures and database lock orchestration preventing multiple users from booking the identical seat simultaneously.

### 2. High-Level Design (HLD) & Distributed Systems
#### Scalability, Caching, and Message Brokers
57. Explain the CAP Theorem. If a network partition occurs, how do you mathematically choose between Consistency and Availability in a banking app vs. a social media feed?
58. Explain PACELC theorem. How does it expand on CAP when a distributed network is running normally without partitions?
59. How does Consistent Hashing work in horizontal scaling load balancers and distributed databases? Walk through the hash ring mechanics and virtual nodes.
60. Design a multi-tier caching architecture. Explain Cache-Aside, Write-Through, Write-Behind, and Refresh-Ahead patterns. When does each fail?
61. How do you handle cache invalidation at scale? Explain the thundering herd problem, cache avalanche, and cache penetration, and detail structural fixes for each.
62. Deep dive into Apache Kafka Architecture. How do partitions, consumer groups, offsets, and broker replication factors guarantee both high performance and zero data loss?
63. What is the difference between Kafka and RabbitMQ regarding message ingestion? Compare log-centric pull architectures against smart-broker push routing.
64. How do you guarantee exactly-once processing semantics across a complex, distributed microservice architecture utilizing Kafka topics?

#### Databases, Transactions, & Distributed Operations
65. Explain Database Isolation Levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable). What anomalies (Dirty Read, Non-repeatable Read, Phantom Read) occur in each?
66. How does Multi-Version Concurrency Control (MVCC) operate in relational databases like PostgreSQL to allow non-blocking reads during active updates?
67. Explain Sharding vs. Partitioning. How do you choose an optimal sharding key to completely eliminate hot spot partition issues?
68. Explain SQL vs. NoSQL indexing mechanisms. Contrast B-Trees/B+ Trees used in relational systems with Log-Structured Merge (LSM) Trees in write-heavy NoSQL databases.
69. How do you implement the Saga Pattern (Orchestration vs. Choreography) to manage cross-service distributed transactions across multiple microservices?
70. Explain the Two-Phase Commit (2PC) protocol. Why is it considered an anti-pattern in high-throughput, cloud-native modern microservices?
71. How do you handle distributed race conditions across detached application instances? Compare Optimistic Locking (versioning) vs. Pessimistic Locking vs. Distributed Locking (Redlock via Redis).
72. Design a real-time notification engine capable of handling 50,000 push, email, and SMS notifications per second with strict prioritization queues.
73. Design a globally scalable URL shortening service (like Bitly). Detail the API specs, database choices, base62 encoding logic, and caching layers.
74. Design a high-volume financial ledger system. How do you guarantee absolute auditability, idempotency, and partition tolerance?

---

## Part 3: Databases & Data Engineering

### 1. Relational Databases (SQL - PostgreSQL / MySQL)
75. Write an optimized SQL query to find the 3rd highest-paid employee within each distinct department from an enterprise schema without using loops.
76. Explain the explicit operational execution differences between a `Clustered Index` and a `Non-Clustered Index`. How does this change physical disk storage?
77. How do you profile an expensive, slow-running SQL query? Walk through the interpretation of an `EXPLAIN ANALYZE` output looking for sequential scans vs index scans.
78. What is Connection Pooling? How do you tune maximum pool sizing limits using empirical formulas for production databases like HikariCP?
79. Explain Deadlocks in SQL databases. How does the database engine automatically detect them, and how do you write application code to avoid them?

### 2. Non-Relational Databases & Key-Value Stores (MongoDB / Redis)
80. How does MongoDB store and look up data using BSON documents? Explain the architecture of replica sets, primary elections, and write concerns.
81. Explain Redis data persistence architectures: RDB (snapshots) vs. AOF (Append-Only File). What are the performance and recovery trade-offs of combining both?
82. How do you implement a distributed sliding-window rate limiter using Redis sorted sets (`ZSET`) via atomic Lua scripts?
83. Explain MongoDB indexing strategies. How do compound indexes work, and why does the ordering of fields inside a compound index matter for query optimization?

---

## Part 4: Production Engineering, Devops & Security

### 1. CI/CD, Containerization & Cloud Native Architecture
84. Explain the difference between Docker images and containers. How do copy-on-write file systems and namespaces/cgroups guarantee host isolation?
85. How do you optimize a Dockerfile for a React application to minimize final production image sizes down to under 20MB using multi-stage builds?
86. Explain the operational runtime differences between a Kubernetes Deployment, StatefulSet, DaemonSet, and Generic Pod.
87. How does Kubernetes manage service discovery and internal cluster routing via Kube-Proxy and CoreDNS?
88. Design a comprehensive CI/CD pipeline using GitHub Actions or Jenkins that performs automated security linting, unit testing, sonarqube analysis, containerization, and canary deployment.
89. Explain the technical differences between rolling updates, blue-green deployments, and canary release strategies. How do you roll back safely?

### 2. High Availability, Monitoring, and Security
90. Explain the operational differences between horizontal autoscaling (HPA) and vertical autoscaling (VPA). What metrics should trigger scaling actions?
91. How do you configure centralized application logging and metric tracking using the ELK Stack (Elasticsearch, Logstash, Kibana) or Prometheus and Grafana?
92. What are the key metrics you monitor on a production dashboard to track the health of a distributed microservice layer? (Explain the Four Golden Signals).
93. Explain the architectural setup of Distributed Tracing using OpenTelemetry and Jaeger. How does a unique correlation ID map requests across service networks?
94. Explain the top 3 OWASP Top 10 vulnerabilities (SQL Injection, XSS, CSRF) and detail the concrete full-stack architectural remedies for each.
95. How do you protect an exposed, public-facing Node.js or Spring Boot REST API from Distributed Denial of Service (DDoS) and brute force credential stuffing attacks?
96. Explain the difference between symmetric and asymmetric encryption. How is TLS/SSL initialized under the hood during a client-server web handshake?

---

## Part 5: Behavioral, Leadership & Practical Scenarios

### 1. Behavioral & STAR Method Scenarios
97. Tell me about the most technically complex feature you designed and shipped end-to-end in your 5-6 years career. What were the specific bottlenecks?
98. Describe a scenario where you had a major technical disagreement with a Senior Architect or Tech Lead. How did you structure your data to resolve it?
99. Tell me about a time a critical system crashed in a production environment under your watch. Walk through your immediate debugging, mitigation, and post-mortem analysis.
100. How do you handle managing technical debt while product owners are aggressively pushing for rapid business feature delivery schedules? Provide an example.
