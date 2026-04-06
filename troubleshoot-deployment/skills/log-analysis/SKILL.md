# Log Analysis

Rules and patterns for parsing container logs, recognizing error patterns, and extracting diagnostic information from Spring Boot application output.

## Version: 1.0.0

---

## Rules

### LA01: Use `--previous` for CrashLoopBackOff Pods

When a pod is in CrashLoopBackOff, `kubectl logs <pod>` shows logs from the *current* (possibly just-started) container. The crash information is in the *previous* container:

```bash
kubectl logs <pod-name> -n <namespace> --previous --tail=200
```

If `--previous` fails with "previous terminated container not found", the current container's logs may still contain the error if it crashed recently.

### LA02: Use `--tail` to Limit Output

Always use `--tail=200` (or similar) to avoid overwhelming output. Start with the end of the logs — that is where the error usually is.

### LA03: Spring Boot Startup Failure Pattern

A Spring Boot application that fails to start will show:
```
***************************
APPLICATION FAILED TO START
***************************

Description:
<specific error description>

Action:
<suggested fix>
```

This banner is the single most important log output for startup failures. Everything above it is normal startup logging.

### LA04: MongoDB Connection Error Pattern

When the MongoDB URI is wrong or MongoDB is unreachable:
```
org.springframework.data.mongodb.UncategorizedMongoDbException: ...
Caused by: com.mongodb.MongoTimeoutException: Timed out after 30000 ms while waiting for a server that matches...
```

Or with a wrong hostname:
```
com.mongodb.MongoSocketOpenException: Exception opening socket
Caused by: java.net.UnknownHostException: mongo-primary
```

The hostname in `UnknownHostException` tells you exactly what wrong hostname is configured.

### LA05: OOM Has No Application Logs

When a container is OOMKilled, there are usually NO error logs from the application itself — the process is killed by the kernel before it can log anything. If you see an OOMKilled exit code but empty or truncated logs, this confirms the OOM diagnosis.

The logs may show normal Spring Boot startup progress and then just stop mid-line.

### LA06: Port Binding Error Pattern

```
org.springframework.boot.web.server.PortInUseException: Port 8081 is already in use
```

This is rare in Kubernetes (each pod has its own network namespace) but can happen if the SERVER_PORT env var conflicts with something.

### LA07: Configuration Property Error Pattern

```
org.springframework.boot.context.properties.bind.BindException: Failed to bind properties under 'spring.data.mongodb'
```

Or:
```
Could not resolve placeholder 'SOME_VAR' in value "${SOME_VAR}"
```

This means an expected environment variable or configuration property is missing or malformed.

### LA08: Look for the `Caused by` Chain

Java stack traces use a `Caused by:` chain. The ROOT cause is always the LAST `Caused by:` in the chain:

```
org.springframework.beans.factory.BeanCreationException: ...
  Caused by: org.springframework.data.mongodb.UncategorizedMongoDbException: ...
    Caused by: com.mongodb.MongoTimeoutException: Timed out after 30000 ms...
      Caused by: com.mongodb.MongoSocketOpenException: Exception opening socket
        Caused by: java.net.UnknownHostException: mongo-primary    ← THIS IS THE ROOT CAUSE
```

### LA09: Timestamp Analysis

Spring Boot logs include timestamps:
```
2026-04-05T10:15:32.123Z  INFO 1 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8081
2026-04-05T10:15:32.456Z  INFO 1 --- [main] c.e.cms.client.ClientServiceApplication  : Started ClientServiceApplication in 3.2 seconds
```

Compare log timestamps with pod creation time to understand how long the application ran before failing.

### LA10: Distinguish Between Startup and Runtime Failures

- **Startup failure**: Error appears before "Started XXXApplication in X seconds" message. Usually a configuration or dependency issue.
- **Runtime failure**: Error appears after successful startup. Usually a resource issue (OOM), external dependency failure, or application bug.

---

## Common Pitfalls

- **Reading only current container logs**: For CrashLoopBackOff, the current container may have just started and has no errors yet. Always use `--previous`.
- **Stopping at the first exception**: Java stack traces can be verbose. Scroll past the noise to find the `Caused by` chain root.
- **Assuming no logs = no error**: OOMKilled containers are killed by the kernel and produce no application-level error. No logs + exit code 137 = OOM.
- **Ignoring log timestamps**: A Spring Boot app that logs for 3 seconds then dies is likely a startup config issue. One that runs for 5 minutes then dies may be a memory leak or load issue.
