# Database Connection Pool

A database connection pool keeps a reusable set of database connections so the application does not create a new physical connection for every request.

A pool should be selected and tuned together with the database driver, database limits, workload, and application framework. The fastest pool in a benchmark is not automatically the best choice for every system.

## Quick Comparison

| Pool | Best fit | Strengths | Trade-offs |
| --- | --- | --- | --- |
| **HikariCP** | Most modern Java applications | Very fast, small API, low overhead, strong defaults, widely adopted | Fewer legacy features; configuration is intentionally compact |
| **Apache Commons DBCP2** | Apache-based applications and compatibility-focused systems | Mature, familiar, extensive configuration, good validation options | Usually more overhead than HikariCP; more settings to maintain |
| **c3p0** | Older applications requiring legacy JDBC behavior | Long history, statement pooling, connection testing, recovery features | Larger footprint and generally less attractive for new services |
| **Tomcat JDBC Pool** | Applications already running on Tomcat | Flexible, feature-rich, integrates naturally with Tomcat | Less compelling when Tomcat integration is not needed |
| **Agroal** | Quarkus and Jakarta applications | Lightweight, modern, good integration with Quarkus and Narayana | Smaller ecosystem outside its main frameworks |
| **Oracle UCP** | Oracle Database applications | Oracle-specific features, fast failover, RAC and FAN integration | Best value is limited to Oracle environments; vendor-specific |
| **Vibur DBCP** | Teams wanting a small, diagnostic-friendly pool | Lightweight, leak detection, useful monitoring features | Smaller community and ecosystem than the mainstream choices |
| **Vert.x Pool** | Reactive Vert.x applications | Designed for asynchronous and reactive workloads | Primarily useful inside the Vert.x ecosystem |

## HikariCP

HikariCP is the usual default for a new Java service using JDBC. It has a small implementation, low allocation overhead, and a focused configuration model.

### Advantages

- Excellent throughput and latency for common JDBC workloads.
- Small dependency and low runtime overhead.
- Commonly supported by Spring Boot and other modern frameworks.
- Clear lifecycle and timeout settings.
- Good operational behavior when the database or network becomes unhealthy.

### Example Configuration

```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.validation-timeout=5000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.leak-detection-threshold=0
```

The exact values should come from measurements and database limits. Do not enable leak detection permanently at a low threshold because it can create noisy logs.

## Apache Commons DBCP2

DBCP2 is a solid general-purpose pool with a broad configuration surface. It is useful when an application already depends on Apache Commons components or needs features and behavior familiar from older deployments.

Choose DBCP2 when:

- Compatibility with an existing configuration matters more than minimal overhead.
- The team needs its validation, eviction, or abandoned-connection features.
- The application already standardizes on Apache Commons libraries.

For a new service with no compatibility constraint, HikariCP is usually simpler to operate.

## c3p0

c3p0 is a mature pool often found in older Hibernate and Java EE systems. It can still be appropriate when replacing it would create unnecessary migration risk, but it is rarely the first choice for a new application.

Choose c3p0 when:

- Existing behavior depends on c3p0-specific settings.
- The application needs its legacy recovery or statement-pooling behavior.
- A gradual migration is safer than changing the pool and driver together.

Plan a migration assessment before using c3p0 in a new service.

## Tomcat JDBC Pool

Tomcat JDBC Pool is a practical option for applications deployed on Tomcat. It offers many controls for validation, abandoned connections, interceptors, and pool behavior.

Choose it when Tomcat integration or its interceptor features are important. Otherwise, a smaller pool such as HikariCP may reduce configuration and operational complexity.

## Agroal

Agroal is designed for modern Jakarta and Quarkus deployments. It provides a lightweight pool with strong integration into those runtimes and transaction environments.

Choose Agroal when:

- The service is built with Quarkus.
- Jakarta transaction integration is a priority.
- The team wants the pool supported as part of its platform rather than configured independently.

## Oracle Universal Connection Pool

Oracle UCP is the strongest choice for systems that depend on Oracle-specific capabilities, such as RAC-aware behavior, Fast Connection Failover, or runtime connection labeling.

Choose UCP for Oracle workloads that need those features. For an Oracle application using only ordinary JDBC connections, compare its operational benefits against the portability and simplicity of HikariCP.

## Reactive and Non-JDBC Applications

A JDBC connection pool is not automatically suitable for a reactive database client. Reactive applications should use the pool supplied by their client ecosystem, such as Vert.x Pool or an R2DBC-compatible pool.

Avoid wrapping blocking JDBC calls in a reactive API and assuming that makes the database access non-blocking. The driver and pool must both support the intended execution model.

## How to Choose

### Select HikariCP when

- The application is a new Java service using JDBC.
- Low overhead and predictable behavior matter.
- The framework has first-class HikariCP support.
- There is no vendor-specific requirement.

### Select DBCP2 when

- Existing Apache Commons configuration should be retained.
- The application needs its broader compatibility and validation options.
- Migration risk is more important than peak pool efficiency.

### Select c3p0 when

- The application already relies on c3p0-specific behavior.
- A replacement would be part of a larger, riskier migration.

### Select Tomcat JDBC Pool when

- The service is tightly coupled to Tomcat.
- Its interceptors or abandoned-connection controls solve a real operational need.

### Select Agroal when

- The application runs on Quarkus or a Jakarta platform where Agroal is the native choice.

### Select Oracle UCP when

- Oracle RAC, FAN, failover, or other Oracle-specific features are required.

## Pool Sizing Notes

Pool size is a concurrency limit, not a performance score. A larger pool can increase database contention, memory use, lock waits, and request latency.

Start with a conservative size and measure:

- Database CPU, I/O, active sessions, and lock waits.
- Pool active, idle, pending, and timeout counts.
- Query latency and application request latency.
- Transaction duration and the number of connections held per request.

A useful starting point for a small service is often between 5 and 10 connections, followed by load testing. The final limit must fit within the database's connection budget:

```text
sum(pool sizes across application instances)
  + administrative and background connections
  <= database connection limit
```

Do not set `maximumPoolSize` to the database's entire connection limit. Leave capacity for migrations, monitoring, administration, replicas, and other services.

## Important Settings

| Setting | Purpose | Guidance |
| --- | --- | --- |
| Maximum pool size | Upper bound on concurrent connections | Size from workload and database capacity |
| Minimum idle | Connections kept ready | Use a stable value for predictable traffic; avoid excessive idle connections |
| Connection timeout | How long a request waits for a connection | Keep finite and aligned with the request timeout |
| Maximum lifetime | Recycle age of a connection | Set below infrastructure or database connection limits when required |
| Idle timeout | Reclaim unused connections | Useful for bursty workloads; avoid constant churn |
| Validation timeout | Maximum time for connection validation | Keep shorter than connection timeout |
| Leak detection | Reports connections held too long | Enable temporarily while investigating suspected leaks |

## Operational Checklist

- Use the database vendor's supported JDBC driver version.
- Keep pool acquisition timeouts finite; never wait indefinitely.
- Set a connection lifetime lower than any load balancer, proxy, or database-enforced lifetime when those components terminate old connections.
- Monitor pool exhaustion separately from slow SQL.
- Return connections in a `finally` block or use try-with-resources.
- Keep transactions short and avoid network calls while holding a connection.
- Configure credentials through secrets management rather than source-controlled files.
- Test database restart, network interruption, and pool recovery behavior.
- Load test with the same number of application instances used in production.

## Recommendation

For a new Java/JDBC application, start with **HikariCP** unless the framework, database vendor, reactive model, or existing deployment requires another pool. Choose the smallest pool that meets the measured concurrency target, then verify the result with database and application metrics.
