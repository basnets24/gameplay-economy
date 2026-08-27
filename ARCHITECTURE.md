# Gameplay Economy: System Design & Production Architecture

A consolidated reference for the Gameplay Economy platform: a .NET 8 microservice system for a game item economy (catalog, inventory, trading, and identity), deployed to Azure Kubernetes Service. This document is meant to be (a) committed to the repo as living documentation and (b) pasted into a diagramming tool to regenerate the architecture drawing whenever the system changes.

Repositories (GitHub org `dotnetmicroservice001`, a polyrepo with one repo per service):

| Repo | Role |
|---|---|
| [Play.Catalog](https://github.com/dotnetmicroservice001/Play.Catalog) | Item catalog service |
| [Play.Inventory](https://github.com/dotnetmicroservice001/Play.Inventory) | Player inventory service |
| [Play.Trading](https://github.com/dotnetmicroservice001/Play.Trading) | Purchase orchestration (saga) + real-time updates |
| [Play.Identity](https://github.com/dotnetmicroservice001/Play.Identity) | Auth, users, in-game currency |
| [Play.Frontend](https://github.com/dotnetmicroservice001/Play.Frontend) | React SPA client |
| [Play.Common](https://github.com/dotnetmicroservice001/Play.Common) | Shared library (Mongo, MassTransit, JWT, logging, telemetry), published as a NuGet package |
| [Play.Infra](https://github.com/dotnetmicroservice001/Play.Infra) | Infrastructure-as-code: Helm chart, ingress, cert-manager, observability stack |

---

## 1. Service inventory

| Service | Port | Owns data | Sync API | Async role |
|---|---|---|---|---|
| **Catalog** | 5000 | Mongo `items` | REST (`ItemsController`) | Publishes `CatalogItemCreated/Updated/Deleted` |
| **Inventory** | 5004 | Mongo `catalogitems` (replica), `inventoryitems` | REST (`ItemsController`) | Consumes catalog events + `GrantItems`/`SubtractItems` commands; publishes `InventoryItemsGranted/Subtracted`, `InventoryItemUpdated` |
| **Trading** | 5006 | Mongo `catalogitems`/`inventoryitems` (replicas), `users`, `userpurchasestats`, saga state | REST (`PurchaseController`, `StoreController`) + SignalR hub (`/messagehub`) | Runs the `PurchaseStateMachine` saga; consumes catalog/inventory/identity events to keep local read models current |
| **Identity** | 5002 | Mongo (ASP.NET Identity stores: users, roles) | REST (`UsersController`) + Duende IdentityServer endpoints | Consumes `DebitGil`; publishes `GilDebited`, `UserUpdated` |
| **Frontend** | n/a (Nginx) | n/a | Calls all four services through the gateway | SignalR client for live purchase status |

Each backend service is a self-contained bounded context with its own database, and services keep to their own data store. Cross-service data needed for local decisions (for example, Trading pricing a purchase, or Inventory validating a catalog item) is kept as a local, eventually consistent read model built from consumed domain events. `Play.Inventory.Service.Clients.CatalogClient` is a direct REST client to Catalog, used for cases that call for a live lookup.

`Play.Common` is a shared NuGet package that every backend service references for Mongo repository abstraction, MassTransit bus setup (RabbitMQ or Azure Service Bus, switched by config), JWT bearer validation, Seq logging, OpenTelemetry tracing and metrics, and health checks. This keeps cross-cutting behavior consistent across the four services.

---

## 2. Communication patterns

**Synchronous (HTTP/REST):**
- Browser → Emissary Ingress → service, for all normal CRUD/API calls.
- Browser ↔ Trading, over SignalR (WebSocket, upgraded through the ingress) for live purchase status.
- Inventory → Catalog, direct HTTP client (`CatalogClient`), for on-demand lookups outside the event-driven path.

**Asynchronous (MassTransit over RabbitMQ locally, Azure Service Bus in production, toggled by `ServiceSettings:MessageBroker`):**
- *Domain events* (fire-and-forget, pub/sub): `CatalogItemCreated`, `CatalogItemUpdated`, `CatalogItemDeleted`, published by Catalog and consumed by Inventory and Trading to keep local replicas in sync.
- *Commands* (point-to-point, via `EndpointConvention` queue mapping): `GrantItems` / `SubtractItems` → Inventory, `DebitGil` → Identity.
- *Saga orchestration*: Trading's `PurchaseStateMachine` (a MassTransit state machine, state persisted in MongoDB) drives the purchase workflow end-to-end and reacts to `Fault<T>` events for compensation.

### Purchase saga (the core business workflow)

```
PurchaseRequested
  → CalculatePurchaseTotalActivity (priced from Trading's local CatalogItem replica)
  → send GrantItems → Inventory                         [state: Accepted]
      ⤷ on InventoryItemsGranted → send DebitGil → Identity   [state: ItemsGranted]
          ⤷ on GilDebited → Completed, push status via SignalR
          ⤷ on Fault<DebitGil> → send SubtractItems (compensating transaction, rolls back the grant)
                                → Faulted, push status via SignalR
      ⤷ on Fault<GrantItems> → Faulted, push status via SignalR
```

Every completed purchase also runs an inline anomaly check. A running mean and variance (Welford's algorithm) of each user's purchase totals is kept in `userpurchasestats`. A purchase whose amount is more than 3 standard deviations from that user's historical mean gets flagged, tracked by the `purchases_flagged_total` metric, as a lightweight fraud and anomaly signal.

---

## 3. Data & persistence

- **Database-per-service**: each service has its own MongoDB database, named after the service, with its own schema.
- In production, MongoDB runs as **Azure Cosmos DB for MongoDB** (`cosmos-pe`), a managed service.
- Locally (`Play.Infra/docker-compose.yml`), a single `mongo` container backs all services.
- Saga state (`PurchaseState`) is persisted through MassTransit's MongoDB saga repository, so the saga survives service restarts.

---

## 4. Security

- **Duende IdentityServer** (hosted inside Play.Identity) issues OAuth 2.0 / OpenID Connect tokens; the frontend uses Authorization Code + PKCE.
- Scopes are fine-grained per capability: `catalog.readaccess`, `catalog.writeaccess`, `catalog.fullaccess`, `inventory.fullaccess`, `trading.fullaccess`, `IdentityServerApi`.
- Every backend service validates incoming JWTs via `Play.Common.Identity` and enforces policy-based authorization (role + scope claims), defined per-service (e.g., Catalog's `Policies.Read` / `Policies.Write`).
- Identity Server's token-signing certificate is a production X.509 cert, mounted into the pod from a Kubernetes secret (`signing-cert`) issued by cert-manager.
- **Secrets**: Azure Key Vault (`kv-pf-pos`), accessed through **Azure AD Workload Identity**. Each service's Kubernetes ServiceAccount is federated to its own Azure AD application (see the `azure.workload.identity/client-id` annotation per service), so pods fetch secrets using that federated identity directly.
- **Transport security**: cert-manager + a `ClusterIssuer` for Let's Encrypt (HTTP-01 challenge) issues the public TLS cert for `gameplayeconomy.com` / `www.gameplayeconomy.com`, terminated at the Emissary ingress.

---

## 5. Observability

- **Tracing & metrics**: OpenTelemetry instrumentation in every service (HTTP client, ASP.NET Core, MassTransit) exported over OTLP.
- **Jaeger** collects traces (`jaegertracing/all-in-one` locally; the `jaegertracing` Helm chart in the `observability` namespace in AKS).
- **Prometheus** scrapes each service's `/metrics` endpoint (`kube-prometheus-stack` in AKS); **Grafana** (bundled in the same chart) provides dashboards.
- **Seq** ingests structured logs from every service (`datalust/seq` locally and via Helm in AKS).
- **Health checks**: every service exposes `/health/live` and `/health/ready` (Mongo connectivity checked via a custom `MongoDbHealthCheck`), wired directly into Kubernetes liveness/readiness/startup probes.
- All three tools are exposed publicly through the same ingress, path-routed: `/jaeger/`, `/prometheus/`, `/grafana/`.

---

## 6. Production deployment topology (Azure)

| Azure resource | Purpose |
|---|---|
| AKS cluster (`aks-playground`) | Runs every microservice, the ingress, and the observability stack |
| Azure Container Registry (`acrpfpos`) | Docker images **and** OCI-packaged Helm charts |
| Azure Cosmos DB for MongoDB (`cosmos-pe`) | Production datastore for all services |
| Azure Service Bus (`sb-pf-pos`) | Production message broker (MassTransit target) |
| Azure Key Vault (`kv-pf-pos`) | Centralized secrets, read via Workload Identity |
| Azure AD Workload Identity | Federated pod-to-Azure-resource auth, using per-pod identity |

**Ingress**: [Emissary-Ingress](https://www.getambassador.io/) (Ambassador's Envoy-based gateway) is the single entry point, with an HTTP (8080) and HTTPS (8443) `Listener`, a `Host` bound to the public domain, and path-prefix `Mapping`s:

| Path prefix | Routed to |
|---|---|
| `/identity-svc/` | Identity |
| `/catalog-svc/` | Catalog |
| `/inventory-svc/` | Inventory |
| `/trading-svc/` | Trading (WebSocket upgrade allowed, for SignalR) |
| `/jaeger/`, `/prometheus/`, `/grafana/` | Observability stack |
| `/` | Frontend (catch-all) |

> **Note**: each service repo also carries an older, raw `kubernetes/*.yaml` manifest (some still pointing at an earlier `playeconomyapp.azurecr.io` registry and domain). These predate the shared Helm chart described below. The Helm chart, the `acrpfpos` registry, and the `gameplayeconomy.com` domain make up the live deployment path used by the current `cicd.yml` pipelines.

**Per-service deployment**: each backend service is deployed via a **shared, generic Helm chart** (`Play.Infra/helm/microservice`, packaged and pushed to ACR as an OCI chart named `playflow-microservice`), parameterized per service through that service's own `helm/values.yaml`. Each service gets its own Kubernetes namespace, its own `Deployment` + `ClusterIP` `Service` + `ServiceAccount`, and standard resource requests/limits plus startup/liveness/readiness probes. The Frontend uses its own dedicated chart (static Nginx container serving the React build, with runtime config injected via a `ConfigMap`).

---

## 7. CI/CD (per-repo GitHub Actions)

Each service repo has an independent pipeline (`.github/workflows/cicd.yml`) that runs on every push to `main`:

```
push to main
  → auto version bump (git tag, semver patch)
  → [Catalog/Inventory/Trading/Identity only] pack & publish the *.Contracts project
      as a NuGet package to GitHub Packages, so other services can pin a version
      of the events/commands contract
  → az login (OIDC, using a federated GitHub credential)
  → docker build & push image → ACR, tagged with the bumped version
  → az aks get-credentials
  → helm upgrade --install <service> oci://<ACR>/helm/playflow-microservice
      -f ./helm/values.yaml --set image.tag=<version> --wait
```

This makes deploys fully automatic and independent per service. Merging to `Play.Catalog`'s `main` ships only Catalog, on its own release step. The shared `*.Contracts` NuGet packages version breaking event and command shape changes across service boundaries, with each service pinning the version it needs.

---

## 8. Local development

`Play.Infra/docker-compose.yml` stands up the full dependency set for running all services locally: MongoDB, RabbitMQ (with the management UI), Seq, Jaeger, and Prometheus. A developer runs the four .NET services and the frontend directly, with `dotnet run` and `npm start`, against these local containers.

---

## Diagram-ready summary (for pasting into a diagram generator)

**Nodes, grouped:**
- *Client*: Browser
- *Edge*: Emissary Ingress (TLS via cert-manager/Let's Encrypt), routing by path prefix
- *Frontend*: React SPA (Nginx)
- *Services* (each own namespace + own Mongo DB): Identity, Catalog, Inventory, Trading
- *Messaging*: Azure Service Bus / RabbitMQ (MassTransit)
- *Data*: Azure Cosmos DB (Mongo API), one logical database per service
- *Secrets*: Azure Key Vault (via Workload Identity)
- *Observability*: Jaeger, Prometheus + Grafana, Seq (all services push/expose to these)
- *CI/CD*: GitHub Actions → GitHub Packages (NuGet contracts) + ACR (images & Helm chart) → AKS

**Key edges to draw:**
- Browser → Emissary Ingress → {Frontend, Identity, Catalog, Inventory, Trading}
- Browser ⇄ Trading (SignalR/WebSocket, through ingress)
- Catalog → (publish events) → Service Bus → {Inventory, Trading} (consume)
- Trading → (send command) → Service Bus → Inventory (`GrantItems`/`SubtractItems`)
- Trading → (send command) → Service Bus → Identity (`DebitGil`)
- Identity/Inventory → (publish result event) → Service Bus → Trading (`GilDebited`, `InventoryItemsGranted`)
- Each service → its own Cosmos DB database
- Each service → Key Vault (secrets), Jaeger/Prometheus/Seq (telemetry)
- GitHub Actions → ACR (image + Helm chart) → AKS; GitHub Actions → GitHub Packages (Contracts NuGet)

Suggested visual treatment: one layered "production runtime" diagram (client → edge → services → messaging, data, observability), plus a separate small sequence diagram for the purchase saga above. That workflow is worth drawing step by step.
