# Making the services "cloud-ready"

> **In plain words:** code that works on a laptop is not automatically ready for Kubernetes.
> Kubernetes needs to ask each service *"are you alive?"* and *"can you take traffic?"*, it needs
> to collect numbers (metrics) and logs from it, and it needs the service to stop politely when a
> new version replaces it. Every service has exactly those things, is packaged as a small, secure
> container image, and the whole shop also runs on a laptop with Docker Compose.

## 1. The checklist every service now meets

| # | Requirement | Why Kubernetes / production needs it | How we did it |
|---|---|---|---|
| 1 | **Liveness probe** `GET /health` | If it fails repeatedly, Kubernetes restarts the container | Simple "I'm alive" answer, never touches the DB |
| 2 | **Readiness probe** `GET /ready` | While it fails, Kubernetes stops sending traffic (no restart) | Checks the MongoDB connection |
| 3 | **Metrics** `GET /metrics` | Prometheus scrapes numbers: request rate, errors, latency, CPU, memory | prom-client (Node), Micrometer (Java), prometheus-client (Python) |
| 4 | **JSON logs on stdout** | Loki can search fields like `service`, `level`, `status` | pino (Node), Spring structured logging (Java), stdlib JSON formatter (Python) |
| 5 | **No secrets in logs** | Logs are read by many people/tools | `Authorization` and `Cookie` headers redacted |
| 6 | **Graceful shutdown** | Rolling updates must not cut off users mid-request | On `SIGTERM`: stop accepting, finish in-flight requests, close DB, exit |
| 7 | **Fail fast** | A pod that can never work should crash loudly, not sit "Running" | Exit if the first DB connection fails |
| 8 | **Container image** | Same artifact runs everywhere | Multi-stage Dockerfile, non-root user, no dev tools/npm/pip in the final image |
| 9 | **Vulnerability gate** | Don't ship known-vulnerable code | Trivy: 0 fixable HIGH/CRITICAL in every image |

**Liveness vs readiness — the most common beginner mistake:** if `/health` checked the database,
a 30-second database blip would make Kubernetes *restart every pod at once*. Restarting doesn't
fix a database, it only adds downtime. So: liveness = "process is OK", readiness = "dependencies OK".

## 2. Per-service highlights

| Service | Notable changes |
|---|---|
| api-gateway | `/api/health` asks every backend's `/ready` (Cloudflare uses it to decide whether a whole cloud is healthy); `/api/notification/*` no longer public; CORS decided only by the gateway (`CORS_ORIGINS`), backend CORS headers stripped |
| catalog, notification | Mongoose 5 → 6 (CRITICAL CVE), jsonwebtoken 8 → 9; notification: fixed 3 bugs (bad JWT hung the socket, message to unknown user crashed, HTML injection in the order e-mail) |
| commerce (Python) | Runs under **gunicorn** (production server) instead of Flask's dev server; 1 worker × 8 threads so metrics stay consistent |
| identity (Java) | **Spring Boot 3.3 → 4.0** (40 CVEs → 0); Actuator probes on `/health` + `/ready`; a new test boots the app with the *real* production config (it caught 2 bugs that only appeared in the container) |
| client (React) | nginx image, SPA fallback, cache headers; `API_URL=/api` is relative so **one image works in every cloud** |
| seed-db | Exits `1` on failure (Kubernetes Job shows *Failed*); admin credentials from environment variables (a Secret), not command-line arguments |

## 3. Lessons worth remembering

1. **Run the real container, not only unit tests.** Identity's tests passed, but the container
   crashed: the test config file *shadowed* the production one, so production config was never
   tested. Fix: a test that loads the real config, and a container smoke test every time.
2. **Upgrading a framework is the real security fix.** 40 vulnerabilities disappeared by moving
   identity off an end-of-life Spring Boot; a few more needed explicit patch versions (`pom.xml`).
3. **Read scanner output by *where* the problem is.** Half the findings were in npm/pip inside
   the base image — tools the app never uses at runtime. Deleting them was the fix.
4. **Small security details matter:** credentials in logs, CORS headers passed through a proxy,
   internal endpoints accidentally public. Each was found by testing the running system.

## 4. Redo it yourself

```bash
# per service
npm test   |   python -m pytest   |   ./mvnw verify          # tests + coverage gate
docker build -t <service>:local .                             # image
docker run --rm -v //var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.74.0 \
  image --severity HIGH,CRITICAL --ignore-unfixed <service>:local   # vulnerability scan

# whole shop
see platform-deployments/local/README.md
```
