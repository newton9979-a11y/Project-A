# DevOps Microservices Portfolio Project

A small e-commerce-style system built as three Python/FastAPI microservices,
containerized with Docker, and designed to be deployed to AWS EKS in later phases.

## Architecture

```
                 ┌────────────────┐
   client  ───▶  │  user-service  │───▶ user-db (Postgres)
                 └────────────────┘
                          ▲
                          │ REST (sync check: does user exist?)
                          │
                 ┌────────────────┐        publishes         ┌───────────────────────┐
   client  ───▶  │ order-service  │ ───────────────────────▶ │  RabbitMQ (fanout)     │
                 └────────────────┘   order.created event     └───────────────────────┘
                          │                                              │
                          ▼                                              ▼
                     order-db (Postgres)                     ┌────────────────────────┐
                                                               │ notification-service   │
                                                               │ (consumes & "sends")   │
                                                               └────────────────────────┘
```

- **user-service**: create/list/get users. Postgres-backed.
- **order-service**: create/list/get orders. Calls user-service synchronously (REST)
  to validate the user exists, then publishes an `order.created` event to RabbitMQ.
- **notification-service**: consumes `order.created` events and "sends" a notification
  (currently just logs it — swap in SES/SNS/Twilio later).

This intentionally demonstrates **both** communication patterns you'll be asked about
in interviews: synchronous REST between services, and asynchronous event-driven
messaging via a queue.

## Running locally

Requires Docker and Docker Compose.

```bash
docker compose up --build
```

Services will be available at:
- user-service: http://localhost:8001/docs
- order-service: http://localhost:8002/docs
- notification-service: http://localhost:8003/healthz
- RabbitMQ management UI: http://localhost:15672 (guest/guest)

### Try it out

```bash
# create a user
curl -X POST http://localhost:8001/users \
  -H "Content-Type: application/json" \
  -d '{"email":"jane@example.com","full_name":"Jane Doe","password":"secret123"}'

# copy the returned "id", then create an order for that user
curl -X POST http://localhost:8002/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id":"<paste-id-here>","item_name":"Keyboard","quantity":1,"total_price":49.99}'
```

Watch the `notification-service` logs (`docker compose logs -f notification-service`) —
you should see a "Notification sent" line appear right after the order is created.

## Health checks

Every service exposes `/healthz`. These are already wired into Docker Compose's
`healthcheck` blocks and are what Kubernetes liveness/readiness probes will target
in Phase 2.

## Roadmap (see full plan for details)

- [x] Phase 1: Dockerize + Docker Compose (this repo)
- [ ] Phase 2: Kubernetes manifests, local cluster (kind/minikube)
- [ ] Phase 3: Terraform (VPC, EKS, ECR), deploy to AWS
- [ ] Phase 4: CI/CD with GitHub Actions + ArgoCD (GitOps)
- [ ] Phase 5: Observability — Prometheus, Grafana, Loki
- [ ] Phase 6: Helm charts, HPA, docs polish

## Repo structure

```
devops-microservices/
├── docker-compose.yml
├── .env.example
├── README.md
└── services/
    ├── user-service/
    │   ├── Dockerfile
    │   ├── requirements.txt
    │   └── app/
    │       ├── main.py
    │       ├── models.py
    │       ├── schemas.py
    │       ├── database.py
    │       └── routers/users.py
    ├── order-service/
    │   ├── Dockerfile
    │   ├── requirements.txt
    │   └── app/
    │       ├── main.py
    │       ├── models.py
    │       ├── schemas.py
    │       ├── database.py
    │       ├── clients.py       # calls user-service
    │       ├── events.py        # publishes to RabbitMQ
    │       └── routers/orders.py
    └── notification-service/
        ├── Dockerfile
        ├── requirements.txt
        └── app/
            ├── main.py
            └── consumer.py       # RabbitMQ consumer
```
