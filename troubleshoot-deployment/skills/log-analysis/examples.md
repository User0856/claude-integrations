# Log Analysis — Examples

## Example 1: Wrong MongoDB URI — `UnknownHostException`

```
$ kubectl logs client-service-6d4f8b7c9-x2k4p -n cms --previous --tail=100

2026-04-05T10:15:30.123Z  INFO 1 --- [main] c.e.cms.client.ClientServiceApplication  : Starting ClientServiceApplication using Java 21.0.2
2026-04-05T10:15:31.456Z  INFO 1 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8081 (http)
2026-04-05T10:15:32.789Z  INFO 1 --- [main] org.mongodb.driver.cluster               : Cluster created with settings {hosts=[mongo-primary:27017]}
2026-04-05T10:15:32.790Z  INFO 1 --- [main] org.mongodb.driver.cluster               : No server chosen by ReadPreferenceServerSelector
2026-04-05T10:16:02.791Z ERROR 1 --- [main] o.s.boot.SpringApplication               : Application run failed

org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'mongoTemplate': ...
    at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:326)
    ...
Caused by: org.springframework.data.mongodb.UncategorizedMongoDbException: Timed out after 30000 ms while waiting for a server
    ...
Caused by: com.mongodb.MongoTimeoutException: Timed out after 30000 ms while waiting for a server that matches ReadPreferenceServerSelector{readPreference=primary}
    ...
Caused by: com.mongodb.MongoSocketOpenException: Exception opening socket
    ...
Caused by: java.net.UnknownHostException: mongo-primary

***************************
APPLICATION FAILED TO START
***************************
```

**Analysis:**
- Cluster settings show `hosts=[mongo-primary:27017]` — this is the configured MongoDB host
- `UnknownHostException: mongo-primary` — DNS cannot resolve `mongo-primary`
- The correct hostname should be `mongodb` (matching the Kubernetes Service name)
- **Root cause:** ConfigMap has wrong MongoDB URI with hostname `mongo-primary` instead of `mongodb`

---

## Example 2: OOMKilled — Truncated Logs

```
$ kubectl logs billing-service-7a2b3c4d5-p8r2s -n cms --previous --tail=100

2026-04-05T10:20:01.100Z  INFO 1 --- [main] c.e.cms.billing.BillingServiceApplication : Starting BillingServiceApplication using Java 21.0.2
2026-04-05T10:20:02.200Z  INFO 1 --- [main] o.s.data.mongodb.config.MongoConfigurat  : ...
2026-04-05T10:20:03.300Z  INFO 1 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initia
```

**Analysis:**
- Logs are truncated mid-line (`Tomcat initia...`) — the process was killed externally
- No Java exception or `APPLICATION FAILED TO START` banner
- Combined with exit code 137 from `kubectl describe pod` → OOMKilled
- The JVM was killed by the kernel OOM killer during startup (Tomcat initialization phase)
- **Root cause:** Memory limit too low. JVM + Spring Boot + Tomcat initialization exceeds the container memory limit.

---

## Example 3: Healthy Startup (for comparison)

```
$ kubectl logs client-service-6d4f8b7c9-x2k4p -n cms --tail=50

2026-04-05T10:10:00.100Z  INFO 1 --- [main] c.e.cms.client.ClientServiceApplication  : Starting ClientServiceApplication using Java 21.0.2
2026-04-05T10:10:01.200Z  INFO 1 --- [main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data MongoDB repositories
2026-04-05T10:10:01.500Z  INFO 1 --- [main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 234 ms
2026-04-05T10:10:02.100Z  INFO 1 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8081 (http)
2026-04-05T10:10:02.300Z  INFO 1 --- [main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2026-04-05T10:10:02.900Z  INFO 1 --- [main] org.mongodb.driver.cluster               : Cluster created with settings {hosts=[mongodb:27017]}
2026-04-05T10:10:03.100Z  INFO 1 --- [main] org.mongodb.driver.connection            : Opened connection to mongodb:27017
2026-04-05T10:10:03.400Z  INFO 1 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8081 (http) with context path '/'
2026-04-05T10:10:03.500Z  INFO 1 --- [main] c.e.cms.client.ClientServiceApplication  : Started ClientServiceApplication in 3.4 seconds (process running for 3.8)
```

**Analysis:**
- `hosts=[mongodb:27017]` — correct MongoDB hostname
- `Opened connection to mongodb:27017` — successful connection
- `Started ClientServiceApplication in 3.4 seconds` — successful startup
- This is what healthy logs look like. Compare against this baseline when diagnosing issues.

---

## Example 4: Readiness Probe — No Errors in Application Logs

```
$ kubectl logs contract-service-5c7d9e8f1-m3n7q -n cms --tail=50

2026-04-05T10:10:00.100Z  INFO 1 --- [main] c.e.cms.contract.ContractServiceApplication : Starting ContractServiceApplication
...
2026-04-05T10:10:03.500Z  INFO 1 --- [main] c.e.cms.contract.ContractServiceApplication : Started ContractServiceApplication in 3.5 seconds
2026-04-05T10:10:33.600Z  WARN 1 --- [http-nio-8082-exec-1] o.s.w.s.r.ResourceHttpRequestHandler : Path with "health/ready" was not found
```

**Analysis:**
- Application started successfully — no errors during startup
- The WARN about `health/ready` path not found confirms the readiness probe is hitting a non-existent endpoint
- The application is healthy — the problem is the probe configuration, not the application
