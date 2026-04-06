# troubleshoot-deployment

Claude Code plugin for autonomous Kubernetes deployment diagnostics. Analyzes cluster state, identifies root causes, and generates incident post-mortems — all read-only, no mutations.

## Usage

```
/troubleshoot-deployment cms
/troubleshoot-deployment cms client-service
```

## How It Works

The plugin executes an 11-phase diagnostic flow:

1. **Load Skills** — reads 9 skill definitions for diagnostic patterns
2. **Input Parsing** — namespace and optional service target
3. **Cluster Overview** — `kubectl get all`, identify unhealthy resources
4. **Pod Diagnostics** — `kubectl describe pod`, events, restart analysis
5. **Log Analysis** — container logs, error pattern matching
6. **Config Inspection** — ConfigMap verification, env var cross-reference
7. **Network Diagnostics** — Services, endpoints, DNS
8. **Resource Analysis** — CPU/memory limits, OOMKilled detection
9. **Image Diagnostics** — pull status, tag verification
10. **Root Cause Synthesis** — correlate findings into diagnosis
11. **Post-Mortem Generation** — structured incident report

## Skills

| Skill | Focus |
|-------|-------|
| cluster-overview | High-level namespace health, pod STATUS/READY triage |
| pod-diagnostics | Container states, exit codes, events, probe config |
| log-analysis | Spring Boot stack traces, MongoDB errors, OOM indicators |
| config-inspection | ConfigMap values, env var verification, URI format |
| network-diagnostics | Service endpoints, port mapping, DNS |
| resource-analysis | Memory limits vs JVM requirements, OOMKilled detection |
| image-diagnostics | ImagePullBackOff, minikube image cache |
| root-cause-synthesis | Decision tree, evidence correlation, diagnosis format |
| post-mortem | Structured incident report: timeline, root cause, prevention |

## Installation

Skills and command are installed to `~/.claude/`:

```bash
cp -r skills/* ~/.claude/skills/
cp commands/troubleshoot-deployment.md ~/.claude/commands/
```
