# discovery-service

> The Eureka service registry of the banking platform: every other service registers here, and the API gateway resolves `lb://` route targets against it. Public port **8761**, actuator port **18761**.

## Table of Contents

- [What It Does](#what-it-does)
- [Architecture At A Glance](#architecture-at-a-glance)
- [HTTP API](#http-api)
- [Error Response Shape](#error-response-shape)
- [Database](#database)
- [Kafka Topics](#kafka-topics)
- [Configuration](#configuration)
- [Behaviour Rules](#behaviour-rules)
- [Running It](#running-it)
- [API Quick Reference (curl examples)](#api-quick-reference-curl-examples)
- [Configuration Reference (quick table)](#configuration-reference-quick-table)
- [Troubleshooting](#troubleshooting)
- [Notes and Known Limitations](#notes-and-known-limitations)

## What It Does

The discovery service is the **service registry** of the whole banking platform. It is a standalone
Netflix Eureka server (`@EnableEurekaServer` in
`src/main/java/com/banking/discovery/DiscoveryServiceApplication.java`) that every other Spring Boot
service registers with, and that the API gateway (`api-gateway`, port 8080) queries to resolve a
service name to a host and port for its `lb://` routes.

It owns nothing else. There is **no database**, **no Kafka**, **no Redis** and **no business logic**
in this repository — the whole service is the registry plus its actuator. It is deliberately
configured **not to register with itself and not to fetch from itself**
(`register-with-eureka: false`, `fetch-registry: false`), because it *is* the registry. Its
dashboard at `http://localhost:8761` is the single place where you can see which of the registered
services are currently UP.

The service must therefore start **first**. Every other service in the platform points its
`eureka.client.service-url.defaultZone` at `http://localhost:8761/eureka` and registers there; if
the registry is not up, the clients keep retrying and the gateway cannot resolve any route.

The registry exposes a standard Eureka REST API (`/eureka/apps`, `/eureka/apps/{appName}`, heartbeat
and status endpoints) plus a small HTML dashboard on the public port, and the Spring Boot actuator
on a **separate private port**.

## Architecture At A Glance

```
        ┌───────────────────────────────────────────────┐
        │  discovery-service (Eureka server)            │
        │  public  :8761   dashboard + Eureka REST API  │
        │  private :18761  actuator (health/info/prom)  │
        └───────────────▲───────────────────────────────┘
                        │  register + heartbeat every 30 s
   ┌──────────┬─────────┴────────┬──────────┬───────────┐
   │ account  │ fund-transfer    │ user     │ core      │  … and
   │ :8081    │ :8082             │ :8083    │ banking   │  5 more
   └────┬─────┴─────────┬────────┴────┬─────┴─────┬─────┘
        │                 │             │           │
        └────────┬────────┴─────────────┴───────────┘
                 │  lb:// route resolution
        ┌────────▼──────────────────────────────────┐
        │  api-gateway :8080                        │
        │  registry-fetch-interval-seconds: 5        │
        └───────────────────────────────────────────┘
```

| Service | Depends on | Depended on by |
|---|---|---|
| `discovery-service` (8761) | nothing — it must be up first | `account-service` (8081), `api-gateway` (8080), `charge-fee-service` (8085), `core-banking-service` (8084), `event-bus-service` / `kafka-event-service` (8090), `fund-transfer-service` (8082), `notification-service` (8087), `otp-service` (8086), `user-service` (8083) |
| `banking-web-ui` (3000) | not a Eureka client at all | – |

## HTTP API

The discovery service is not a REST business API. It serves the standard Eureka registry API and
the actuator. Everything below is served by the framework — this repository contains no controllers.

### GET / — Eureka dashboard

The HTML dashboard. It lists every registered application with its instances, their status, their
host and port, and their health check URL. This is the page to open when a service is not visible
from the gateway.

| Response | Type | Notes |
|---|---|---|
| `200` | HTML | Dashboard page |

### GET /eureka/apps — the whole registry

The full registry as JSON. This is what `run-all.sh` polls while it waits for the services to
register.

| Field | Type | Notes |
|---|---|---|
| `applications` | object | Map of application name to registration |
| `applications.*.name` | string | Upper-case Eureka app name, e.g. `ACCOUNT-SERVICE` |
| `applications.*.instance` | array | One entry per registered instance |
| `applications.*.instance[].app` | string | Lower-case `spring.application.name`, e.g. `account-service` |
| `applications.*.instance[].instanceId` | string | |
| `applications.*.instance[].hostName` | string | `localhost` unless the client sets `prefer-ip-address` |
| `applications.*.instance[].port` | object | `{"$": 8081, "@enabled": "true"}` |
| `applications.*.instance[].status` | string | `UP` or `DOWN` |
| `applications.*.instance[].metadata` | object | Carries e.g. `"management.port":"18081"` |
| `applications.*.instance[].healthCheckUrl` | string | e.g. `http://10.112.21.135:18086/actuator/health` |

| Response | Type | Notes |
|---|---|---|
| `200` | JSON | Registry dump |
| `404` | JSON | Only for `/eureka/apps/{unknown-app}` |

### GET /eureka/apps/{appName} — one application

The registration of a single application. The app name is the **upper-case** Eureka name, e.g.
`ACCOUNT-SERVICE`.

| Response | Type | Notes |
|---|---|---|
| `200` | JSON | One application object |
| `404` | JSON | No such application is registered — this is also the answer for `DISCOVERY-SERVICE`, because the registry never registers itself |

### GET /actuator/health, /actuator/health/liveness, /actuator/health/readiness

Actuator on the **private management port 18761** only.

| Response | Type | Notes |
|---|---|---|
| `200` | JSON | `{"status":"UP","groups":["liveness","readiness"]}` |
| `404` | – | On the public port 8761 — the actuator is not exposed there |

### GET /actuator/info

| Response | Type | Notes |
|---|---|---|
| `200` | JSON | `{}` — no info contributors are configured |

### GET /actuator/prometheus

| Response | Type | Notes |
|---|---|---|
| `200` | text | Prometheus exposition format; every metric carries the tag `application="discovery-service"` |

### Error Response Shape

This repository defines **no** custom error body. Errors come from the Eureka framework or from
Spring Boot and use their standard shapes (for example the WebFlux/Actuator error body with
`timestamp`, `path`, `status`, `error`). There is no `ErrorResponse` record in this service.

## Database

**Not applicable.** The discovery service has no datasource, no JDBC driver and no entity classes.
Its only state is the in-memory Eureka registry, which is rebuilt from the clients' heartbeats
within seconds of a restart.

## Kafka Topics

**Not applicable.** The discovery service has no Kafka dependency in `pom.xml` and neither
publishes nor consumes any event. It only knows about service names and network addresses.

## Configuration

### Server and Spring properties

`src/main/resources/application.yml`:

| Name | Default | Meaning |
|---|---|---|
| `server.port` | `8761` | Public port — the Eureka dashboard and the Eureka REST API |
| `spring.application.name` | `discovery-service` | Name used in logs and in the `application` metric tag |
| `eureka.instance.hostname` | `localhost` | Hostname the server advertises for its own instances |
| `eureka.client.register-with-eureka` | `false` | The server never registers itself as a client |
| `eureka.client.fetch-registry` | `false` | The server never fetches a registry from itself |
| `eureka.client.service-url.defaultZone` | `${EUREKA_URL:http://localhost:8761/eureka}` | Zone URL, overridable with `EUREKA_URL` |
| `eureka.server.enable-self-preservation` | `false` | Local development only — stopped services leave the dashboard quickly instead of being kept alive by self-preservation |
| `eureka.server.eviction-interval-timer-in-ms` | `5000` | Eviction sweep every 5 s |
| `management.server.port` | `${MANAGEMENT_PORT:18761}` | Actuator on a **separate** port |
| `management.endpoints.web.exposure.include` | `health,info,prometheus` | Only these three actuator endpoints are exposed |
| `management.endpoint.health.show-details` | `never` | Health returns only the status, no component detail |
| `management.endpoint.health.probes.enabled` | `true` | `/actuator/health/liveness` and `/actuator/health/readiness` are enabled |
| `management.metrics.tags.application` | `${spring.application.name}` | Adds `application="discovery-service"` to every metric |
| `management.metrics.distribution.percentiles-histogram.http.server.requests` | `true` | Enables the latency histogram for HTTP requests |

### Custom `app.*` properties

**None.** This service defines no `app.*` properties at all.

### Environment variables

| Name | Default | Meaning |
|---|---|---|
| `EUREKA_URL` | `http://localhost:8761/eureka` | Zone URL of the registry. Optional — it only matters if you run several registries or move this one |
| `MANAGEMENT_PORT` | `18761` | Port of the actuator. Optional |
| `SERVER_PORT` | `8761` | Port of the public API. `run-all.sh` sets it explicitly to `8761` |

### Environment variables you must set

**None.** Every property in this service has a working default. The service starts with an empty
environment.

## Behaviour Rules

### Which applications register

The registry is **empty of itself**. `GET /eureka/apps` contains no `DISCOVERY-SERVICE` entry and
`GET /eureka/apps/DISCOVERY-SERVICE` answers `404`. The startup log line
`Registering application DISCOVERY-SERVICE with eureka with status UP` comes from the server-side
registry initialising its own in-memory store, **not** from a client registration.

Every other service registers itself. The nine applications that appear in the registry are:

| Eureka app name | `spring.application.name` | Port | Management port |
|---|---|---|---|
| `ACCOUNT-SERVICE` | `account-service` | 8081 | 18081 |
| `API-GATEWAY` | `api-gateway` | 8080 | 18080 |
| `CHARGE-FEE-SERVICE` | `charge-fee-service` | 8085 | 18085 |
| `CORE-BANKING-SERVICE` | `core-banking-service` | 8084 | 18084 |
| `FUND-TRANSFER-SERVICE` | `fund-transfer-service` | 8082 | 18082 |
| `KAFKA-EVENT-SERVICE` | `kafka-event-service` (note: **not** `event-bus-service`) | 8090 | 18090 |
| `NOTIFICATION-SERVICE` | `notification-service` | 8087 | 18087 |
| `OTP-SERVICE` | `otp-service` | 8086 | 18086 |
| `USER-SERVICE` | `user-service` | 8083 | 18083 |

That is **9** registered applications. `banking-web-ui` is **not** a Eureka client at all.

### Eviction and self-preservation

| Rule | Value | Effect |
|---|---|---|
| Self-preservation | **disabled** | A service that stops heart-beating is removed from the dashboard instead of being kept "alive" by the renewal threshold |
| Eviction interval | **5 s** | The registry sweeps for expired leases every 5 seconds |
| Client lease (as observed on the clients) | `renewalIntervalInSecs: 30`, `durationInSecs: 90` | A client that dies disappears from the dashboard roughly 90 s after its last heartbeat |

### The actuator is private

`check-all.sh` asserts exactly this contract: it queries `http://127.0.0.1:18761/actuator/health`
and `http://127.0.0.1:18761/actuator/prometheus`, and then checks that
`http://127.0.0.1:8761/actuator/health` is **not** `200` — it prints `EXPOSED!` if it is.

### The management port is reachable from outside the loopback

The in-file comment on `management.server.port` explains why the actuator is not on `127.0.0.1`:

> `# The other services publish their metrics on 18080-18090. Prometheus runs in`
> `# a container and scrapes them through host.docker.internal, so the metrics`
> `# port has to be reachable from outside the loopback interface.`

That is the same reason Prometheus scrapes `host.docker.internal:18761` for this service.

## Running It

### Prerequisites

- Java 17
- Maven 3.9+ (or the bundled wrapper `./mvnw`)
- **Nothing else.** The discovery service has no database, no Kafka and no Redis dependency, so it
  can be the very first process you start.

### Build and run

```bash
cd discovery-service
mvn clean package -DskipTests
java -jar target/discovery-service-0.0.1-SNAPSHOT.jar
```

Or with the Maven wrapper:

```bash
cd discovery-service
./mvnw spring-boot:run
```

Or as part of the whole platform:

```bash
cd /home/bs01092/Projects/banking-microservices
./run-all.sh
```

`run-all.sh` starts this service first, exports `SERVER_PORT=8761`, waits up to 90 × 2 s for the
port to open and prints `[$name] UP`, or fails with
`[$name] did not open port 8761 within 3 minutes.`

### Environment variables to export

None are required. If you want to change the ports:

```bash
export SERVER_PORT=8761
export MANAGEMENT_PORT=18761
export EUREKA_URL=http://localhost:8761/eureka
```

### How to check it is healthy

```bash
# Actuator on the private port
curl -s http://localhost:18761/actuator/health
# {"status":"UP","groups":["liveness","readiness"]}

# The registry itself
curl -s -H 'Accept: application/json' http://localhost:8761/eureka/apps | head -40

# The dashboard
open http://localhost:8761
```

To check the whole platform:

```bash
cd /home/bs01092/Projects/banking-microservices
./check-all.sh
```

It expects the discovery service to report `UP`, at least 50 metric lines on port 18761, and the
actuator to be **blocked** on the public port.

### How to stop it

```bash
# If you started it with ./run-all.sh
cd /home/bs01092/Projects/banking-microservices
./stop-all.sh

# If you started it by hand
kill <pid>
```

## API Quick Reference (curl examples)

```bash
# The registry as JSON
curl -s -H 'Accept: application/json' http://localhost:8761/eureka/apps

# One application (upper-case Eureka name)
curl -s -H 'Accept: application/json' http://localhost:8761/eureka/apps/ACCOUNT-SERVICE

# The registry deliberately does not contain itself
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8761/eureka/apps/DISCOVERY-SERVICE
# 404

# Health on the private port
curl -s http://localhost:18761/actuator/health
curl -s http://localhost:18761/actuator/health/liveness
curl -s http://localhost:18761/actuator/health/readiness
curl -s http://localhost:18761/actuator/info
curl -s http://localhost:18761/actuator/prometheus | head

# The actuator is NOT on the public port
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8761/actuator/health
# 404

# The dashboard
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8761/
# 200
```

## Configuration Reference (quick table)

| Knob | Default | Change it when |
|---|---|---|
| `server.port` | `8761` | Port 8761 is taken on your machine |
| `management.server.port` | `18761` | Prometheus already scrapes something on that port |
| `eureka.server.enable-self-preservation` | `false` | Never in local development; only for a production registry where a network blip must not empty the registry |
| `eureka.server.eviction-interval-timer-in-ms` | `5000` | You want stopped services to disappear faster or slower |
| `eureka.client.service-url.defaultZone` | `http://localhost:8761/eureka` | You run more than one registry |
| `eureka.instance.hostname` | `localhost` | You run the registry in a container and want a routable hostname |
| `management.endpoint.health.show-details` | `never` | You want the health endpoint to list its components |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `GET /actuator/health` on port 8761 answers `404` | This is the intended behaviour — the actuator is on the private port | Use `http://localhost:18761/actuator/health` |
| `check-all.sh` prints `EXPOSED!` for `discovery-service` | The management port was moved onto the public port, e.g. `MANAGEMENT_PORT=8761` | Unset `MANAGEMENT_PORT` or set it to `18761` |
| A service never shows up in the dashboard | It started before the registry, or it points at the wrong `EUREKA_URL` | Check the client's `eureka.client.service-url.defaultZone`; it must be `http://localhost:8761/eureka` |
| A stopped service is still shown as `UP` | Self-preservation is on (it is off by default here) | Confirm `eureka.server.enable-self-preservation: false` in `application.yml` |
| The dashboard shows `DISCOVERY-SERVICE` | Impossible with the shipped configuration; the entry only appears if `eureka.client.register-with-eureka` was changed to `true` | Set `eureka.client.register-with-eureka: false` back |
| The gateway answers `503 No servers available for service: account-service` | The gateway's copy of the registry is empty or stale | Wait for the next registry fetch (the gateway polls every 5 s), or check that the account service really registered |
| `Port 8761 was already in use` | Another Eureka server, or a leftover process from a previous run | `./stop-all.sh`, or `lsof -i :8761` to find the process |
| The `run-all.sh` wait loop fails with `did not open port 8761 within 3 minutes` | The service crashed on startup | Read `logs/discovery-service.log` |

## Notes and Known Limitations

- **Self-preservation is off.** The in-file comment says *"Local development only: removes stopped
  services from the dashboard quickly."* In a production cluster you would normally leave it on, so
  that a network partition between the registry and the clients does not wipe the registry.
- **The service is a single point of failure.** There is no peer replication; if the registry is
  down, no client can register and the gateway cannot resolve any `lb://` route.
- **The registry is in-memory.** A restart of the discovery service empties the dashboard. The
  clients re-register within their 30 s renewal interval, but there is a short window in which the
  gateway answers `503`.
- **The startup log line `Registering application DISCOVERY-SERVICE with eureka with status UP` is
  misleading.** It comes from the server-side `EurekaServiceRegistry` initialising its own store,
  not from a client registration. `GET /eureka/apps/DISCOVERY-SERVICE` really does answer `404`.
- **The event bus service registers under the name `KAFKA-EVENT-SERVICE`**, not `EVENT-BUS-SERVICE`,
  because its `spring.application.name` is `kafka-event-service`. Look for the former in the
  dashboard.
- **The test class lives in a different package.** `src/test/java/com/banking/discoveryservice/`
  (`discoveryservice`, one word) versus the main package `com.banking.discovery`. The test is a
  plain `contextLoads()` smoke test.
- **`run-all.sh` prints a wrong hint for Kafka.** It says
  *"Start it first: docker compose up -d kafka kafka-init"* and refers to a `kafka-init` container,
  but **neither compose file defines a `kafka-init` service**. `event-bus-service/docker-compose.yml`
  defines `kafka` and `kafka-ui`. The command that actually works is
  `docker compose -f event-bus-service/docker-compose.yml up -d`.
- **The root `run-all.sh` header says "all 9 Java services"** while the `SERVICES` array holds 10.
  The `EXPECTED_SERVICES` calculation (`${#SERVICES[@]} - 1` = 9) is nevertheless correct, because
  the discovery service does not register itself and the web UI is not a client.
- **Maven coordinates**: `com.banking:discovery-service:0.0.1-SNAPSHOT`. Dependencies:
  `spring-cloud-starter-netflix-eureka-server`, `spring-boot-starter-actuator`,
  `micrometer-registry-prometheus`, `spring-boot-starter-test`. Spring Boot 3.5.5, Spring Cloud
  BOM 2025.0.0, Java 17.

# Discovery Service

**Part of the Banking Microservices Project**

Developed and maintained by **Jahid Hasan**

- **Author:** Jahid Hasan
- **GitHub:** [Md-Jahid-Hasan-10](https://github.com/Md-Jahid-Hasan-10)
- **Repository:** [discovery-service](https://github.com/Md-Jahid-Hasan-10/discovery-service)
- **Project:** Banking Microservices
- **Application Port:** `8761`
- **Management/Metrics Port:** `18761`

> The Eureka service registry of the banking platform. Every other service registers here, and the API gateway resolves `lb://` route targets against it.
