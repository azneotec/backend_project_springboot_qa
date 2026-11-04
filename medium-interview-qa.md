# Backend Interview Prep: 10 Medium-Level Questions and Answers

Based on the notes in `backend_project_springboot_notes` (Spring Boot e-commerce microservices), cross-checked against current Spring practice. Difficulty sits between the beginner set and the advanced set: each question asks for a mechanism, a trade-off and a failure mode, not only a definition. **Note** marks points where the lecture notes differ from current practice.

| # | Topic |
|---|---|
| 1 | Bean scopes and thread safety |
| 2 | Calling an external API: generics, DTOs, timeouts |
| 3 | The N+1 select problem |
| 4 | Inheritance mapping trade-offs |
| 5 | `@Transactional` mechanics |
| 6 | Test slices and realistic databases |
| 7 | Local JWT validation, claims, revocation |
| 8 | Offset vs keyset pagination |
| 9 | Payment webhooks, idempotency, reconciliation |
| 10 | Kafka partitions, consumer groups, delivery guarantees |

---

## 1. Bean scopes and thread safety

**Q: Spring beans are singletons by default. What does that imply for a `@Service` class that keeps a field such as a running counter or a cached list? What goes wrong if a prototype-scoped bean is injected into a singleton, and how is that fixed? What happens with a circular constructor dependency?**

**Answer**
- **Singleton scope** means one instance per application context. That single instance is shared by all request threads, so any mutable instance field is shared mutable state: race conditions, lost updates, leaked data between users.
- Rules: keep service beans **stateless**, keep state in method locals, the database or a cache; where shared state is unavoidable, use thread-safe types (`AtomicLong`, `ConcurrentHashMap`) or proper locking.
- **Prototype in singleton:** the singleton's dependencies are injected **once**, at creation. The prototype is therefore created once and reused forever, defeating its purpose. Fixes:
  - Inject an `ObjectProvider<T>` (or `Provider<T>`) and call `getObject()` each time.
  - Use `@Lookup` method injection.
  - Declare the bean with a scoped proxy (`proxyMode = ScopedProxyMode.TARGET_CLASS`), which is also how request-scoped or session-scoped beans are injected into singletons.
- **Circular constructor dependency** (A needs B, B needs A) cannot be resolved: Spring throws `BeanCurrentlyInCreationException`. Setter or field injection used to hide cycles through early singleton references; since Spring Boot 2.6 circular references are **prohibited by default** (`spring.main.allow-circular-references`). The right fix is redesign: extract a third class, or use events.
- Related point: for singleton beans, thread safety also covers injected collaborators; `RestTemplate` and `RestClient` are thread-safe, so one shared bean is correct.

---

## 2. Calling an external API from a service

**Q: A service fetches all products from a third-party API with `restTemplate.getForObject(url, List<FakeStoreProductDto>.class)`. Why does that not compile or work as intended, and what are the working alternatives? What else must be configured before this call is production-ready?**

**Answer**
- Generic type arguments are erased at runtime, and a class literal cannot carry them (`List<X>.class` is not valid Java; `List.class` loses the element type). With raw `List.class`, Jackson deserializes the elements as `LinkedHashMap`, and a `ClassCastException` appears later when the elements are used as DTOs.
- Working alternatives:
  - Fetch an **array**: `FakeStoreProductDto[]` (the approach in the notes), then convert with `Arrays.asList` or a stream.
  - Use `exchange(...)` with a `ParameterizedTypeReference<List<FakeStoreProductDto>>() {}`, which captures the generic type through an anonymous subclass.
  - The same `ParameterizedTypeReference` is accepted by `RestClient` (`.body(new ParameterizedTypeReference<...>() {})`) and by `WebClient`.
- Then map each DTO to the internal model (`toProduct()`), so the external schema never leaks into the domain.
- **Production readiness:**
  - Set **connect and read timeouts**. The default `RestTemplate` configuration can wait indefinitely, so one slow upstream call can exhaust the server's threads.
  - Handle errors: a 4xx or 5xx response throws `HttpClientErrorException` or `HttpServerErrorException`; an empty body can return `null` and cause a `NullPointerException` in the mapping step.
  - Add retries with backoff for idempotent calls and a circuit breaker (for example Resilience4j) so a failing dependency does not cascade.
  - Use `getForEntity` or `exchange` when status code or headers are needed.
- **Note:** `RestTemplate` is in maintenance mode; `RestClient` (synchronous) or `WebClient` (reactive) is the current guidance for new code.

---

## 3. The N+1 select problem

**Q: A page lists 50 users with their roles, and the SQL log shows 51 statements. Explain what is happening, why switching the association to `FetchType.EAGER` does not reliably solve it, and give three better fixes with one caveat each.**

**Answer**
- **N+1:** one query loads the parent rows (the 1), then each parent's lazy association triggers its own query when accessed (the N). Cost grows linearly with the number of rows, and it is invisible with small test data.
- **Why `EAGER` is not a fix:** the eager setting is a *global* mapping decision. For `findById`, Hibernate can join in the same statement, but for JPQL or derived queries (`findAll`, custom finders) the base query runs first and the eager association is loaded with **follow-up selects per row**, which is the same N+1. It also taxes every query on the entity, including ones that never need roles.
- **Better fixes:**
  1. **Fetch join** in JPQL: `select u from User u join fetch u.roles`. Caveat: fetching a collection together with pagination forces Hibernate to paginate **in memory** (a warning is logged); fetching several `List` collections at once raises `MultipleBagFetchException` (use `Set` or separate queries).
  2. **`@EntityGraph`** on the repository method, which declares what to load per use case while the mapping stays lazy. Same pagination caveat for collections.
  3. **Batch fetching** (`@BatchSize` or `hibernate.default_batch_fetch_size`): lazy collections for many parents load with `where user_id in (...)` in groups. Caveat: still several queries, but a bounded number.
  4. (Also valid) **DTO projection** selecting only the needed columns. Caveat: no managed entity, so no automatic dirty checking.
- **Detection:** enable SQL logging or Hibernate statistics in tests, and assert on statement counts.

---

## 4. Inheritance mapping trade-offs

**Q: A `User` hierarchy has `Mentor` and `Mentee` subclasses. Compare single-table and joined-table mapping on schema, query cost, integrity and polymorphic queries. Why are table-per-class and `@MappedSuperclass` awkward for a hierarchy that must be queried as `User`?**

**Answer**
- **Single table** (`InheritanceType.SINGLE_TABLE`): one table with a discriminator column (`@DiscriminatorColumn`, `@DiscriminatorValue`) and the union of all subclass columns.
  - Pros: no joins, fastest reads, simple polymorphic queries.
  - Cons: many nullable columns (sparse rows), and `NOT NULL` constraints cannot be enforced on subclass-specific columns, which weakens integrity.
- **Joined table** (`InheritanceType.JOINED`): a parent table with shared columns and one child table per subclass whose primary key is also a foreign key to the parent.
  - Pros: normalized, no redundant columns, subclass constraints can be enforced.
  - Cons: a join for each concrete read; polymorphic queries join or left-join all child tables; deeper hierarchies mean more joins.
  - This is the strategy the notes call the "ideal solution".
- **Table per class** (`TABLE_PER_CLASS`): every concrete class has a full table, including inherited columns. A polymorphic query on `User` needs a `UNION` across all tables, shared columns are duplicated, and identifiers must be unique across tables, so `IDENTITY` generation cannot be used.
- **`@MappedSuperclass`:** the parent is *not an entity*. It only shares mapped fields with subclasses, so a query such as "find a user by email" across `Mentor` and `Mentee` is impossible as one polymorphic query, and the parent cannot be a relationship target.
- **Choosing:** joined when subclasses have many specific fields and integrity matters; single table when the hierarchy is shallow, subclass columns are few, and read speed dominates. Lookup by shared columns (login by email) is a strong argument against table-per-class and mapped superclass.

---

## 5. `@Transactional` mechanics

**Q: Explain how `@Transactional` works internally. Give two situations where the annotation is present but no transaction (or no rollback) happens, and say where the transaction boundary should sit in a layered application.**

**Answer**
- Spring wraps the bean in a **proxy** (AOP). When an *external* caller invokes the method, the proxy's interceptor asks the `PlatformTransactionManager` to begin or join a transaction, calls the real method, then commits, or rolls back if an exception escapes.
- **Situation 1, self-invocation:** a method calling another `@Transactional` method of the *same class* goes through `this`, not the proxy, so the annotation on the inner method is ignored. Fix: move the method to another bean, or restructure.
- **Situation 2, checked exceptions:** by default rollback occurs only for `RuntimeException` and `Error`. A checked exception commits the transaction unless `rollbackFor = Exception.class` is declared.
- Other traps: swallowing the exception in a `try/catch` inside the method (the proxy never sees it); calling on a bean created with `new` (no proxy); non-public methods in classic proxy setups.
- **Propagation:** the default `REQUIRED` joins an existing transaction or starts one; `REQUIRES_NEW` suspends the current transaction and starts an independent one (use for audit logging that must survive a rollback). A rollback-only marker set by an inner participating method makes the outer commit fail with `UnexpectedRollbackException`.
- **`readOnly = true`** is a hint that lets Hibernate skip dirty checking and lets some drivers and routers optimize reads.
- **Boundary:** the **service layer**, one transaction per business use case. Not controllers (web concerns) and not individual repository calls (too short, no atomicity across several operations).
- **Keep transactions short:** an open transaction holds a database connection, so remote HTTP calls inside it can exhaust the connection pool.

---

## 6. Test slices and realistic databases

**Q: A controller test is written with `@SpringBootTest` and calls the controller method directly. List two problems with that, name the narrower alternatives and what each loads, and explain why a persistence test should not rely on an in-memory H2 database when production runs another engine.**

**Answer**
- **Problems:**
  1. The full context loads everything (services, repositories, security, messaging), so startup is slow and failures are hard to localize. Differing mock configurations create separate cached contexts.
  2. Calling the Java method directly **skips the web layer**: URL mapping, parameter binding, validation, JSON serialization, status codes and the `@ControllerAdvice` translation are never exercised, so the test proves less than it appears to.
- **Narrower alternatives:**
  - `@WebMvcTest(ProductController.class)` loads the MVC infrastructure only (the named controller, advice, converters, filters, security config). Collaborators are supplied as `@MockitoBean` (the successor of the deprecated `@MockBean`), and requests go through `MockMvc`, which can assert the 404 status and JSON error body.
  - `@DataJpaTest` loads entities, repositories and the `EntityManager`, with each test rolled back.
  - `@RestClientTest` tests the HTTP client code against a mock server.
  - `@JsonTest` tests DTO serialization.
  - Plain JUnit and Mockito tests for service logic, with no Spring at all (fastest).
- **H2 vs production engine:** differences in SQL dialect and functions, JSON and array types, locking and isolation behavior, sequences, collation and case sensitivity, and native queries. Tests can pass on H2 and fail in production.
- **Better:** **Testcontainers** running the same engine and version as production, with the schema built by the real migration tool (Flyway or Liquibase).
- **Edge case:** `@DataJpaTest` wraps each test in a transaction, so lazy associations load fine in the test but fail in production (`LazyInitializationException`), and constraint violations only surface on flush.

---

## 7. Local JWT validation, claims and revocation

**Q: A resource service validates JWTs without calling the user service. Which checks must it perform, how does signing algorithm choice affect key distribution, how do roles get from the token into authorization decisions, and how can a token be revoked before it expires?**

**Answer**
- **Checks:** signature verification with the correct key; `exp` (and `nbf`) time claims, with a small clock skew allowance; `iss` (expected issuer); `aud` (intended audience); and an allow-list of accepted algorithms (reject `alg: none`, and never let the token choose a weaker algorithm).
- **Algorithms:**
  - **HS256** (HMAC): one shared secret signs *and* verifies, so every resource server holds the secret and any of them could mint tokens.
  - **RS256 / ES256** (asymmetric): only the authorization server holds the private key; resource servers need only the **public key**, fetched from a **JWKS endpoint** and selected by the `kid` header, which also enables **key rotation**. This suits one issuer and many resource servers.
- **Roles into authorization:** the issuer adds a claim (the notes use an `OAuth2TokenCustomizer<JwtEncodingContext>` to add roles to the access token). On the resource server, a `JwtAuthenticationConverter` maps the claim to `GrantedAuthority` objects. Note the default authority prefix is `SCOPE_` for the `scope` claim; roles usually need a custom converter and a `ROLE_` prefix for `hasRole(...)`. Endpoint rules are then applied (`requestMatchers(...).hasRole("ADMIN")` or `@PreAuthorize`).
- **Outcomes:** missing or invalid token gives **401** (authentication entry point); valid token with insufficient authority gives **403** (access denied handler). **Note:** the lecture maps a failed token check to 403.
- **Revocation (the cost of statelessness):** the token stays valid until expiry. Mitigations: short access-token lifetime plus a refresh token (which can be revoked server-side); a deny-list keyed by the `jti` claim checked on each request (re-introducing some state); or opaque tokens with introspection for cases needing immediate revocation. Role changes also take effect only when a new token is issued.
- The payload is encoded, not encrypted: no sensitive data in claims.

---

## 8. Offset vs keyset pagination

**Q: An endpoint uses `PageRequest.of(page, size, sort)`. Explain what the database does for a deep page, the correctness issue with concurrent writes, and how keyset pagination addresses both. What are its limitations?**

**Answer**
- **Offset pagination** translates to `ORDER BY ... LIMIT n OFFSET k`. For a deep page the database still has to **read and discard the first k rows**, so cost grows with the page number even when an index is used. A `Page` return type also issues a separate **`COUNT` query** for total pages (a `Slice` avoids it).
- **Correctness:** positions are computed per request. A row inserted before the current position pushes the last row of the previous page onto the next one (duplicate); a deletion causes a skipped row. Without a deterministic order (and a unique tie-breaker such as the primary key) the problem exists even with no writes.
- **Keyset (cursor, "seek") pagination:** the client sends the last-seen sort key and the query becomes `WHERE (created_at, id) > (:lastCreatedAt, :lastId) ORDER BY created_at, id LIMIT n`. An index on the sort columns makes every page cost about the same, and inserts or deletes elsewhere do not shift the window.
  - Spring Data supports this through its scrolling API (`ScrollPosition` / `Window`, Spring Data 3.1+).
- **Limitations:**
  - No jumping to an arbitrary page number and no total count without an extra query.
  - Requires a deterministic, indexed, unique-in-combination sort key; multi-column and mixed-direction sorting is more complex.
  - The cursor must be opaque to clients.
- **Elasticsearch:** `from + size` is limited by `index.max_result_window` (10,000 by default); deep traversal uses `search_after`, ideally with a point-in-time snapshot.
- Operational controls: cap the page size and whitelist sortable fields.

---

## 9. Payment webhooks, idempotency and reconciliation

**Q: Describe a payment flow in which the order is created first, the customer pays through a gateway, and the result arrives by redirect and webhook. How is the webhook endpoint secured and made idempotent, and which mismatch scenarios does reconciliation resolve?**

**Answer**
- **Flow:**
  1. Create the order in the shop with status `PENDING`, so there is a record to attach the payment to, even if the customer vanishes.
  2. Create a payment (link or session) at the gateway with the order id as the reference, and send the customer there.
  3. The gateway redirects the customer back to a **callback URL** (for user experience only) and also calls the shop's **webhook** (server to server).
- The redirect can fail (connection drops, browser closed), so the **webhook, together with reconciliation, is the source of truth**.
- **Securing the endpoint:** the URL is public, so verify the **signature** header (HMAC-SHA256 of the *raw* request body with the webhook secret; Razorpay sends it in `X-Razorpay-Signature`) before trusting anything; use HTTPS; optionally confirm the status by calling the gateway API; respond 2xx quickly and do heavy work asynchronously.
- **Idempotency:** gateways retry on timeouts, so the same event can arrive repeatedly and out of order. Store the event or payment id with a unique constraint and ignore repeats; drive the order through a **state machine** (`PENDING → PAID`) so a duplicate `paid` event does not deduct stock or ship twice.
- **Reconciliation** (a scheduled job, a cron) compares the shop's records with the gateway's. The four scenarios in the notes:
  1. Order failed but payment succeeded: refund, or mark the order successful (for essential services such as utility bills).
  2. Both failed: no action.
  3. Order succeeded but payment failed: cancel the order or refund if charged (for example the customer abandoned payment).
  4. Both succeeded: no action.
- Additional production concerns: amount and currency checks (**Note:** Razorpay amounts are in the smallest currency unit, such as paise), refunds and chargebacks, and alerting on unreconciled items.
- Local testing of webhooks uses a tunnel (for example LocalTunnel) to expose a local port.

---

## 10. Kafka partitions, consumer groups and delivery guarantees

**Q: An order service publishes "order placed" events and an email service consumes them. Explain the roles of topics, partitions, keys, consumer groups and offsets; what ordering guarantee exists; and why duplicate emails can still happen and how to prevent them.**

**Answer**
- A **topic** is a named log, split into **partitions**. Each message in a partition has an increasing **offset**.
- **Keys:** messages with the same key go to the same partition. Using the order id (or customer id) as key keeps all events of that entity in order.
- **Ordering guarantee:** only **within a partition**, not across a topic.
- **Consumer group:** consumers sharing a `group.id` split the partitions, and each partition is read by **one** consumer in the group at a time. This gives parallelism and load sharing, capped by the number of partitions (extra consumers sit idle). A different group (for example analytics) receives every message independently. Adding or removing a consumer triggers a **rebalance**.
- **Offsets** record how far a group has read. Offset commit timing decides the guarantee:
  - Commit *after* processing: **at-least-once** (a crash between sending the email and committing causes redelivery, so **duplicates**).
  - Commit *before* processing: **at-most-once** (a crash loses the message).
- **Preventing duplicate emails:** make the consumer **idempotent**: store a processed event id (unique constraint) or check an "email sent" flag before sending. Kafka's idempotent producer and transactions give exactly-once semantics only within Kafka (read-process-write between topics), not for external side effects such as email.
- **Failure handling:** retry with backoff for transient errors, and send "poison" messages that always fail to a **dead-letter topic** so one bad message does not block the partition.
- **Operational points:** monitor consumer lag; define the JSON schema carefully (or use a schema registry) so producers and consumers can evolve independently; the producer can stay fast because it does not wait for the email service, which is the decoupling advantage over a direct call.

---

## Not covered in this set
Docker and Kubernetes, AWS networking and deployment, Spring Cloud (Eureka, gateway), Actuator with Prometheus and Grafana, Elasticsearch internals, and Git. These can be covered in a follow-up set.
