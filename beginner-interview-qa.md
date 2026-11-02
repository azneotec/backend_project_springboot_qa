# Backend Interview Prep: Beginner Questions and Answers

Based on the notes in `backend_project_springboot_notes` (Spring Boot e-commerce microservices), cross-checked against current Spring practice. Questions 1 to 3 were first asked at an advanced level and are rewritten here in beginner form. Where the lecture notes differ from current practice, the difference is flagged with **Note**.

---

## 1. Spring startup and wiring

**Q: In simple terms, what happens when a Spring Boot application starts? What do `@SpringBootApplication`, a bean and dependency injection mean, and why is constructor injection preferred?**

**Answer**

- `SpringApplication.run(...)` creates the **application context**, which is the IoC container. The container creates and connects the objects of the application.
- `@SpringBootApplication` combines three annotations:
  - `@SpringBootConfiguration` (marks the class as a configuration class).
  - `@ComponentScan` (finds classes annotated `@Component`, `@Service`, `@Repository`, `@RestController` in the same package and below).
  - `@EnableAutoConfiguration` (adds ready-made configuration based on the libraries on the classpath, for example an embedded Tomcat when the web starter is present).
- A **bean** is an object created and managed by the container.
- **Dependency injection (DI)**: a class declares what it needs (for example a `ProductService`), and the container supplies it. The class does not create the dependency with `new`. This is **inversion of control**.
- Third-party classes such as `RestTemplate` cannot be annotated, so a `@Configuration` class exposes them with a `@Bean` method. One shared instance is then injected everywhere.
- **Constructor injection is preferred** because:
  - Required dependencies are visible in the constructor.
  - Fields can be `final` (immutable).
  - An object can never exist half-wired.
  - Unit tests can pass a mock directly, with no container.
  - Circular dependencies fail fast at startup.
- After the beans are created, the embedded web server starts and the application accepts requests.

**codex answer:**
When a Spring Boot application starts, it creates the application context, discovers and configures beans, wires their dependencies, and starts the embedded web server if it is a web application; `@SpringBootApplication` combines configuration, component scanning, and auto-configuration based on the libraries and settings present. A bean is an object managed by Spring, and dependency injection means Spring supplies the objects it needs; constructor injection is preferred because it makes required dependencies explicit, allows `final` fields, and makes testing easier.

---

## 2. Lazy loading failure

**Q: What does lazy loading mean for a list of roles inside a `User` entity? Why can accessing the list after the database session closes fail?**

**Answer**

- With lazy loading, Hibernate does not load the roles when the user is loaded. A placeholder (a proxy or collection wrapper) stands in, and the SQL query runs the first time the list is accessed.
- The placeholder needs an **open persistence session** to run that query. Once the session or transaction has ended, access throws `LazyInitializationException`.
- In the authentication flow, the user is loaded in one place and the authorities are read later, outside the session, so the failure appears.
- **Note:** the lecture fixes this with `fetch = FetchType.EAGER`. It works, but it is a global setting that taxes every query on the entity. Better options:
  - Read the roles inside a `@Transactional` service method.
  - Use a **fetch join** or `@EntityGraph` in the specific query that needs the roles.
  - Use a **DTO projection**.

**codex answer:**
Lazy loading means a user’s roles are fetched from the database only when the list is first accessed. If the list has not been loaded before the Hibernate session closes, accessing it can throw a `LazyInitializationException` because Hibernate no longer has an active session to fetch the roles.

---

## 3. Test doubles

**Q: Why is a real database or external service avoided in a unit test? In simple terms, what are a mock, a stub and a fake?**

**Answer**

- Real dependencies make tests **slow, flaky and not repeatable** (network down, data changed) and **not isolated** (a failure in the database looks like a failure in the controller).
- Test doubles replace the real dependency. Definitions in the lecture notes:
  - **Mock:** returns hard-coded values.
  - **Stub:** can react to state changes (for example the count rises after an add).
  - **Fake:** a lightweight working version, such as an in-memory map acting as a database.
- **Note:** the common industry definitions differ:
  - **Stub:** supplies canned answers.
  - **Mock:** pre-programmed with expectations, and the test checks that the expected calls happened.
  - **Fake:** a working shortcut implementation.
  - The lecture's "mock" is closer to a stub, and its "stub" is closer to a fake.
- With Mockito, `when(service.getProduct(1L)).thenReturn(product)` is stubbing, and `verify(...)` is mock-style behavior verification.
- In recent Spring Boot versions, `@MockBean` is deprecated in favor of `@MockitoBean`.

**codex answer:**
Unit tests avoid real databases and external services to stay fast, isolated, and repeatable, without depending on network connections or external state.

- a **stub** returns preset answers,
- a **mock** lets the test verify expected interactions, and
- a **fake** is a simplified working implementation, such as an in-memory repository.

---

## 4. Authentication vs authorization

**Q: What is the difference between authentication and authorization? Give one e-commerce example of each.**

**Answer**

- **Authentication:** verifying who the caller is. Example: logging in with email and password to place an order.
- **Authorization:** deciding what the verified caller may do. Example: only a user with the admin role can change the stock quantity of a product.
- Authentication comes first. Authorization is checked on every request, on the server (hiding a button in the UI is not protection).
- Role-based access control (RBAC) gives users roles, and roles carry permissions.
- Statuses: **401** for missing or invalid credentials, **403** for a valid identity without permission.

**codex answer:** Authentication checks who you are—for example, verifying a customer’s email and password when they log in. Authorization checks what you are allowed to do—for example, allowing only administrators to change product prices.

**4a. Q: What is checked at login, and what is returned afterwards?**

- The submitted password is checked against the **stored salted hash**. The password itself is never stored.
- A **salt** is a random value per user, so identical passwords give different hashes. Slow password hashes such as bcrypt or Argon2 are used (Spring's `PasswordEncoder`).
- After success, the server returns a short-lived **token**. The client sends it on later requests, typically in the `Authorization: Bearer <token>` header.
- A failed login should return a generic message that does not reveal whether the username or the password was wrong.

**codex answer:** At login, the server checks the supplied credentials, such as comparing the password against the stored password hash. If they are valid, it typically returns an access token, such as a JWT, which the client sends with later requests to protected endpoints.

---

## 5. Pagination

**Q: What is pagination, why is it preferred over returning all results, and what inputs does a client send? What does sorting add?**

**Answer**

- Pagination returns a large result set in fixed-size slices (pages).
- Reasons: less memory use, smaller responses, faster replies, less load on the database, and protection for the service.
- Inputs: **page number and page size** (or offset and limit). In Spring Data this becomes a `Pageable` (`PageRequest.of(page, size, sort)`), and the repository returns a `Page` with the content plus total elements and total pages. Page numbers are zero-based.
- Sorting adds a **sort field and direction** so that the order, and therefore the page contents, are defined.
- The server should cap the maximum page size and allow sorting only on approved (indexed) fields.

**codex answer:** Pagination returns results in smaller pages, reducing response time, memory use, and network traffic compared with returning everything; the client typically sends a page number and page size, or a cursor and limit. Sorting specifies the order, such as price ascending, with a unique tie-breaker such as product ID to keep the order consistent across pages.

**5a. Q: What can go wrong without a fixed sort order?**

- Rows can appear on two pages or on none. New or deleted rows shift the positions, and an undefined order makes it worse.

**codex answer:** Without a fixed sort order, the database can return rows in a different order on each request, so items may appear on multiple pages or be skipped entirely. Add an explicit `ORDER BY` with a unique tie-breaker, such as the product ID, to make the ordering deterministic.

**5b. Q: Does a database guarantee row order without a sort clause?**

- **No.** Row order is undefined unless an explicit sort is given, even if the data never changes. Pages can overlap or miss rows even on a static table.

**codex answer:** No, a database does not guarantee row order without an explicit `ORDER BY` clause, even if results happen to appear in the same order repeatedly.

**5c. Q: Why can the same query return a different order, and what fixes it?**

- A table is a set of rows, not an ordered list. The order is a by-product of the access path the query planner picks (sequential scan, index scan, parallel scan, cache, replica), and that can change between runs.
- Fix: add `ORDER BY` with a **unique tie-breaker**, for example `createdAt` then `id`. In Spring: `Sort.by("createdAt", "id")`.
- For very large or frequently changing data, **keyset (cursor) pagination** is more stable and faster than large offsets.

**codex answer:** Without `ORDER BY`, the database may return rows in different orders depending on its execution plan, indexes, or parallel processing. Add an explicit `ORDER BY` with a unique tie-breaker, such as the primary key, to guarantee a consistent order.

---

## 6. Elasticsearch

**Q: What is Elasticsearch, and why is it better than a plain `LIKE` query for text search on products? What does a relational database do better?**

**Answer**

- Elasticsearch is a distributed search engine built on an **inverted index**: text is split into normalized terms (by an analyzer), and each term points to the documents containing it.
- Benefits: fast full-text search, relevance ranking (scoring), typo tolerance (fuzzy matching), word variations (stemming), filters and facets, and horizontal scaling.
- A `LIKE '%phone%'` query with a leading wildcard usually cannot use a normal index and scans every row, and it has no ranking or typo handling.
- A relational database is better at **ACID transactions** and consistent multi-record updates (for example creating an order and reducing stock together). It stays the source of truth.
- Elasticsearch is near-real-time, so the data must be synchronized from the database, and search results can lag slightly.
- AWS OpenSearch is the managed variant mentioned in the notes.

**codex answer:** Elasticsearch is a search engine designed to find and rank relevant results from text, making it better suited to product searches with typos, word variations, and relevance ranking than a basic `LIKE` query. A relational database is better at reliably updating related data together, such as recording an order and reducing stock in one transaction.

---

## 7. Webhooks

**Q: What is a webhook, and why is one needed in addition to the redirect after payment?**

**Answer**

- A webhook is an HTTP request the payment gateway sends **directly to the shop's server** when an event happens (for example `payment_link_paid`).
- The redirect depends on the customer's browser and connection. If the connection drops after paying, the shop never hears about it. The webhook is server-to-server and does not depend on the customer.

**codex answer:** A webhook is an automatic message the payment gateway sends directly to the shop’s server when a payment succeeds. Unlike the redirect, it does not depend on the customer’s browser or connection, so the shop can learn that payment succeeded even if the customer never returns to the website.

**7a. Q: What risks come with a public webhook endpoint?**

- **Fake sender:** a forged "payment succeeded" message could trigger shipment without payment. Defense: verify the gateway's **signature** (HMAC with a shared secret) on every request, use HTTPS, and optionally confirm the status by calling the gateway's API.
- **Duplicate delivery:** gateways retry, so the same event can arrive more than once. Defense: **idempotent processing**, storing the event or payment ID and ignoring repeats, so stock is not reduced twice and shipments are not duplicated.
- Respond with 2xx quickly, and do heavy work asynchronously.
- Local testing: a tunnel such as LocalTunnel exposes a local port to the gateway.

**codex answer:** A fake sender could report a successful payment and trick the shop into shipping unpaid goods, so the shop must verify the webhook’s signature. The real gateway may also send the same event repeatedly, so processing must be idempotent—using payment or event IDs to prevent duplicate actions such as shipping twice.

---

## 8. Kafka

**Q: What is Kafka, what are the producer and consumer roles, and what is one advantage over a direct call between services?**

**Answer**

- Kafka is a distributed, durable log of event messages organized into **topics**.
- **Producer:** publishes events (the order service publishes "order placed"). **Consumer:** reads events (the email service reads them and sends the confirmation).
- Advantages over a direct synchronous call:
  - The order service finishes without waiting for the email service (**decoupling**).
  - Events are stored, so they are processed later if the consumer is temporarily down (**durability, buffering**).
  - Failures in the email service do not fail the order.
- Vocabulary: a topic is split into **partitions**, a **consumer group** shares the work among instances of one service (each partition is read by one consumer in the group), and **offsets** track how far a consumer has read.

**codex answer:** 
- Kafka is a system that stores and delivers streams of event messages: the order service is the producer that publishes an “order placed” event, and the email service is the consumer that reads it and sends the confirmation. One advantage is that the order service can finish without waiting for the email service, while the event remains available for processing if the email service is temporarily unavailable.
- Kafka stores events in named topics: the order service is the producer that publishes an event, while email service instances act as consumers in a consumer group to share the work, using offsets to track their reading progress. This lets checkout finish without waiting for email, and stored events can be processed when the email service recovers from an outage.

---

## 9. Gateway vs load balancer

**Q: What does each do, and how do they differ?**

**Answer**

- **API gateway:** the single entry point. It routes requests by path or rule to **different services** and handles shared concerns: authentication, rate limiting, logging.
- **Load balancer:** spreads requests across **identical copies** of one service and avoids unhealthy ones (health checks).
- Typically both are combined: the gateway chooses the service, and load balancing chooses the instance.

**codex answer:** An API gateway is the entry point that routes client requests to the right service and can handle shared tasks such as authentication and rate limiting. A load balancer spreads requests across multiple copies of a service to share the workload and avoid unhealthy instances.

**9a. Q: How are the current addresses of service copies found?**

- Through **service discovery**. The notes use **Eureka**, a service registry.
- At startup, each instance **registers** its application name and address with Eureka and sends periodic **heartbeats**. If heartbeats stop, the instance is removed after the lease expires.
- Callers fetch and cache the registry and call services by name (for example `lb://user-service`). **Client-side load balancing** (Spring Cloud LoadBalancer) then picks an instance. A cached list can be briefly stale.

**codex answer:** This is called service discovery, and Netflix’s component for it is Eureka, a service registry that gateways and other services consult to find available service instances and their current addresses. When a service instance starts, it registers its address with Eureka and sends regular heartbeats to show it is still running.

- If an instance stops sending heartbeats, Eureka eventually marks its registration as expired and removes it from the registry. Callers periodically refresh their cached address lists and use client-side load balancing to choose an instance, retrying another if a cached address no longer works.

---

## 10. Docker

**Q: What is the difference between an image and a container, and how does an image reduce "works on my machine" problems?**

**Answer**

- **Image:** an immutable, layered template that packages the application, the Java runtime, libraries and default settings. It is built from a **Dockerfile** and stored in a registry.
- **Container:** a running instance of an image, with its own isolated process and filesystem (it shares the host kernel).
- The same image runs on the laptop and the server, so the runtime and dependencies are identical.
- Environment-specific settings (database URL, secrets) are passed in at run time, so one image works in every environment.

**codex answer:** A Docker image is a packaged template containing the service, its Java runtime, dependencies, and default settings; a container is a running instance of that image. Using the same image on the laptop and server gives the service a consistent environment, reducing differences that cause “works on my machine” problems.

- A Dockerfile defines the steps for building an image containing the service and its dependencies, and a container is a running instance of that image. The same immutable image is used across environments, with settings such as database addresses and secrets supplied at runtime, keeping the packaged application consistent while allowing configuration to vary.

---

## 11. AWS network security

**Q: What are a VPC and a security group, and how can they ensure that only the application server can talk to the database?**

**Answer**

- **VPC:** a private, isolated virtual network in AWS, divided into subnets with route tables.
- **Security group:** a **stateful** virtual firewall attached to a resource. It lists allowed inbound and outbound traffic, and return traffic is allowed automatically.
- Design:
  - Application server in a **public subnet** (route to an internet gateway).
  - Database (RDS) in a **private subnet**, not publicly accessible.
  - Database security group: allow inbound on the database port (for example 3306 or 5432) **only from the application's security group**, not from IP ranges.

**codex answer:** A VPC is an isolated network in AWS, and a security group is a virtual firewall that controls traffic to and from resources. Put the application server in a public subnet and the database in a private subnet with no public access, then allow inbound database traffic only on its database port from the application server’s security group.

---

## 12. Git

**Q: What is the difference between merging and rebasing, and what does each leave in the commit history?**

**Answer**

- **Merge:** joins two branches and keeps all existing commits. It usually creates a merge commit; if the target branch has not moved, a **fast-forward** merge needs no extra commit. History stays accurate but can look branched.
- **Rebase:** replays the commits of a branch on top of another branch. The commits get **new IDs**, and the history is linear with no merge commit.
- Rule of thumb: never rebase branches that others have already pulled, because history is rewritten and a force push is required.

**codex answer:** Merging joins the branches while preserving their existing commits and usually adds a merge commit showing where they came together, though a fast-forward merge needs no extra commit. Rebasing replays a branch’s commits on top of another branch, giving them new commit IDs and leaving a linear history without a merge commit.

---

## 13. HTTP status codes

**Q: What do 200, 201, 400, 401, 403, 404 and 500 mean, and what does the first digit indicate?**

**Answer**

- **2xx success:** 200 OK (request succeeded, for example a fetch), 201 Created (a new resource was created, for example a new product).
- **4xx client error:** 400 Bad Request (invalid input), 401 Unauthorized (missing or invalid authentication), 403 Forbidden (authenticated but not permitted), 404 Not Found (resource does not exist).
- **5xx server error:** 500 Internal Server Error (unexpected server problem).
- Other classes: 1xx informational, 3xx redirection.
- **Note:** the lecture returns 403 for a failed token check. The standard code is 401, with 403 only for insufficient permission.

**codex answer:**

- **2xx — Success:** **200 OK** means the request succeeded, such as fetching a product; **201 Created** means a new resource was created, such as adding a product.
- **4xx — Client errors:** **400 Bad Request** means the request is invalid, such as missing required product details; **401 Unauthorized** means valid authentication is missing; **403 Forbidden** means the caller lacks permission; **404 Not Found** means the requested resource does not exist.
- **5xx — Server errors:** **500 Internal Server Error** means an unexpected server problem prevented the request from being completed.
- In Spring Boot, use `ResponseEntity` to set status codes and response bodies, and `@RestControllerAdvice` with `@ExceptionHandler` to handle errors consistently across controllers. Return a consistent error body with a code, a clear message, and relevant validation details, while keeping stack traces and other internal details in server logs rather than exposing them in a 500 response.

---

## 14. Exception handling

**Q: How can a Spring Boot API turn a missing product into a clean 404 response without repeating logic in every controller?**

**Answer**

- The service throws a custom exception, such as `ProductNotFoundException`.
- A central class annotated `@ControllerAdvice` (or `@RestControllerAdvice`) contains a method annotated `@ExceptionHandler(ProductNotFoundException.class)` that returns a `ResponseEntity` with status 404 and a consistent error body (message, code, timestamp).
- Also add a catch-all handler for unexpected exceptions that returns a safe 500 without stack traces, handle validation errors (400), and log the underlying error.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorDto> handle(ProductNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorDto(ex.getMessage()));
    }
}
```

**codex answer:** The service throws a custom `ProductNotFoundException` when the product ID does not exist. A central handler marked with `@RestControllerAdvice` catches it using `@ExceptionHandler(ProductNotFoundException.class)` and returns a `ResponseEntity` with status `404 Not Found` and a helpful, consistent error body.

---

## 15. JPA, Hibernate and Spring Data JPA

**Q: What are the three, how do they relate, and what does a repository interface provide?**

**Answer**

- **JPA:** the standard (specification) for mapping Java objects to database tables (ORM).
- **Hibernate:** the most common implementation of JPA. It generates and runs the SQL.
- **Spring Data JPA:** a convenience layer on top that removes boilerplate.
- A repository interface such as `ProductRepository extends JpaRepository<Product, Long>` provides save, find, list, delete and paging methods with no SQL. Spring generates the implementation at startup.
- Query methods can be derived from method names (for example `findByTitleContaining`), or written with `@Query` (JPQL/HQL or native SQL).

**codex answer:**

- JPA is a standard for mapping Java objects to database tables, Hibernate implements that standard, and Spring Data JPA builds on JPA to make database access easier. By defining a repository interface such as `ProductRepository extends JpaRepository<Product, Long>`, a developer gets ready-made operations to save, find, list, and delete products without writing SQL.
- At startup, Spring Data JPA creates a working repository implementation and makes it available for injection into your service. Beyond built-in operations, it can derive queries from method names such as `findByName`, or run custom queries declared with `@Query`, using JPQL or native SQL.

---

## 16. Lazy vs eager loading

**Q: What does each mean for a `User` with a list of `Role` entities? Give one advantage and one disadvantage of each.**

**Answer**

- **Lazy:** roles are loaded only when accessed.
  - Advantage: no unnecessary database work.
  - Disadvantage: `LazyInitializationException` if accessed after the session closes.
- **Eager:** roles are loaded together with the user.
  - Advantage: always available immediately.
  - Disadvantage: extra data and queries on every load, and possible N+1 problems when many users are loaded.
- Default fetch types: lazy for `@OneToMany` and `@ManyToMany`, eager for `@ManyToOne` and `@OneToOne`.
- Preferred approach: keep associations lazy and load what is needed per query (fetch join, `@EntityGraph`, DTO projection).

**codex answer:**

- Lazy loading fetches the roles only when the list is accessed, saving unnecessary database work, but accessing it after the persistence session closes can cause a `LazyInitializationException`. Eager loading fetches the roles when the user is loaded, making them available immediately, but can load unnecessary data and slow down queries.
- Both lazy and eager loading can cause the N+1 problem: loading users triggers additional queries to fetch each user’s roles. When roles are needed, use a JPQL `JOIN FETCH` or an `@EntityGraph` for that query to fetch them together, rather than changing the relationship to eager loading globally.

---

## 17. Unit test structure

**Q: What are the three A's, and why should the expected value be fixed and not calculated in the test?**

**Answer**

- **Arrange:** prepare data and dependencies (create objects, stub mocks).
- **Act:** call the code under test.
- **Assert:** compare the actual result with the expected result (`assertEquals(expected, actual)`).
- The expected value should be a **hard-coded literal**. If it is calculated with the same logic as the code, a bug can be repeated in both and the broken test still passes.
- Other qualities of a good unit test: fast, isolated, repeatable, self-checking, and testing behavior rather than implementation.

**codex answer:**

- The three A’s are **Arrange** (prepare the test data and dependencies), **Act** (run the code being tested), and **Assert** (check the actual result against the expected result). Write the expected value as a fixed value so the test provides an independent check, rather than repeating the calculation and potentially reproducing the same bug.
- During Arrange, replace external dependencies with test doubles, such as Mockito mocks or stubs; during Assert, use framework methods such as JUnit’s `assertEquals` to compare the actual result with a fixed expected value. Good unit tests are fast, isolated from external systems and other tests, and repeatable, producing the same result each time.

---

## 18. JWT

**Q: What is a JWT, what are its three parts, and why is a signature included?**

**Answer**

- A JWT (JSON Web Token) is a compact, self-contained token carrying **claims** (for example user ID, roles, expiry).
- Three parts, base64url-encoded and separated by dots: **header** (token type and signing algorithm), **payload** (claims), **signature**.
- The signature is computed over header and payload with a secret (HMAC) or a private key (RSA, ECDSA). Any service with the secret or the public key can verify it **without calling the user service**.
- Without a signature, anyone could edit the payload (for example set the role to admin) and the server would trust the forged token.
- The payload is only encoded, **not encrypted**, so no sensitive data should be placed in it.
- Use short expiry (`exp`) and manage keys carefully. Self-contained tokens cannot be revoked before expiry unless a deny-list is used.
- Validation also checks expiry, issuer and the accepted algorithm.

**codex answer:**

- A JWT is a compact token carrying claims, such as a user’s identity and permissions, in three parts: a header describing the token and signing algorithm, a payload containing the claims, and a signature used to verify its authenticity and detect changes. Without a verified signature, an attacker could alter the claims—for example, granting themselves admin access—and the server could trust the forged token.
- A signed JWT’s payload is encoded, not encrypted, so anyone holding it can read it; keep sensitive data out and check the `exp` claim to reject expired tokens. Keep signing secrets or private keys secure and rotate them safely; verification uses the shared secret for HMAC or the corresponding public key for asymmetric signatures.

---

## 19. Spring Security filter chain

**Q: What is the purpose of the filter chain, and what happens to a request with no valid token at a protected endpoint?**

**Answer**

- Spring Security installs a chain of servlet filters (configured through a `SecurityFilterChain`) that run **before** the controller. They authenticate the caller and check authorization.
- A request without a valid token is rejected before reaching the controller with **401 Unauthorized**, produced by the `AuthenticationEntryPoint`.

**codex answer:** Spring Security’s filter chain checks who the caller is and whether they have permission to access the requested endpoint before the request reaches the controller. For a token-based REST API, a request without a valid token is rejected with **401 Unauthorized** when it targets a protected endpoint.

**19a. Q: Where is the caller's identity kept, and how does the response differ for a user without the required role?**

- After the token filter succeeds, an `Authentication` object (principal and authorities) is stored in the `SecurityContext`, accessed through `SecurityContextHolder`. It is held per thread and cleared when the request ends.
- An authenticated user without the required role gets **403 Forbidden**, produced by the `AccessDeniedHandler`. A missing or invalid token gets 401.

**codex answer:** Spring Security stores the caller’s identity and authorities in an `Authentication` object inside the `SecurityContext`, accessed through `SecurityContextHolder` for the rest of the request. An authenticated ordinary user accessing an administrator-only endpoint receives **403 Forbidden**, while a request with no valid token receives **401 Unauthorized**.

---

## 20. Environment variables

**Q: Why should database settings come from environment variables, and how does Spring Boot pick them up?**

**Answer**

- The same build can run in every environment with different settings, and secrets stay out of the code repository.
- Two ways Spring Boot reads them:
  - A placeholder in the properties file: `spring.datasource.password=${DB_PASSWORD}`.
  - **Relaxed binding:** an environment variable such as `SPRING_DATASOURCE_URL` maps automatically to `spring.datasource.url` and **overrides** the value in `application.properties`.
- On AWS Elastic Beanstalk, values are set as environment properties of the environment.
- For stronger security, keep secrets in a managed secrets service, and avoid exposing them in logs or diagnostic endpoints.

**codex answer:** Environment variables let the same application use different database settings in each environment and keep passwords out of the code repository. Spring Boot can read them through placeholders such as `spring.datasource.password=${DB_PASSWORD}`, or automatically map a variable such as `SPRING_DATASOURCE_URL` to `spring.datasource.url`, overriding the value in the properties file.

---

## Corrections to the lecture notes (summary)

| Topic             | Lecture notes                     | Current practice                                                      |
| ----------------- | --------------------------------- | --------------------------------------------------------------------- |
| HTTP client       | `RestTemplate`                    | In maintenance mode; prefer `RestClient` or `WebClient`               |
| Failed token      | 403 Forbidden                     | 401 Unauthorized (403 only for insufficient permission)               |
| Test beans        | `@MockBean`                       | `@MockitoBean` in recent Spring Boot versions                         |
| Test double terms | Mock = hard-coded, stub = dynamic | Stub = canned answers, mock = verifies calls, fake = working shortcut |
| Roles loading     | `FetchType.EAGER`                 | Keep lazy; use fetch join, entity graph or DTO                        |
| Payment webhooks  | Basic setup                       | Also verify signatures and process idempotently                       |

## Not yet covered at beginner level

Payment reconciliation, Kubernetes, and Actuator with Prometheus and Grafana. The notes folder contains files on all three.
