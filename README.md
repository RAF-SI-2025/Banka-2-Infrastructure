# Banka-2-Infrastructure

Kubernetes manifesti za deploy Banka 2 platforme na fakultetski klaster
(`banka-2.radenkovic.rs`). Sve resurse zive u namespace-u `banka-2`.

Ovo je deploy-ready varijanta zasnovana na klaster operatorima (Crunchy
PostgreSQL operator + Envoy Gateway). Raw/portabilni K8s manifesti (StatefulSets,
nginx-ingress, namespace `banka2-tim2`) zive odvojeno u `Banka-2-Backend/k8s/`.

## Manifesti

| Fajl | Sadrzaj |
|------|---------|
| `db.yaml` | Crunchy `PostgresCluster` (Postgres 17, 3 instance + pgBouncer + pgbackrest backup). Jedan fizicki cluster, dve logicke baze: `db` (banka-core) i `trading-db` (trading-service); auto-generisane lozinke u `db-pguser-*` Secret-ima |
| `influxdb.yaml` | InfluxDB (OHLCV time-series) |
| `rabbitmq.yaml` | RabbitMQ broker |
| `backend.yaml` | banka-core Deployment + Service (port 8080) |
| `trading-service.yaml` | trading-service Deployment + Service (port 8082) |
| `notification-service.yaml` | notification-service Deployment + Service (port 8083) |
| `frontend.yaml` | frontend Deployment + Service (port 80) + nginx ConfigMap |
| `route.yaml` | Envoy Gateway `HTTPRoute` — path-based routing za `banka-2.radenkovic.rs` (split na 2 objekta zbog Gateway API limita od 16 rules/objekat) |
| `seed-job.yaml` | banka-core seed Job |
| `trading-seed-job.yaml` | trading-service seed Job |

## Routing

`route.yaml` definise dva `HTTPRoute` objekta na zajednickom parentRef-u
(`banka-gw` u `envoy-gateway-system`), hostname `banka-2.radenkovic.rs`. Envoy
kombinuje sva pravila po longest-prefix-match:

- **`banka-2-trading`** — trgovinski `/api/*` prefiksi (`/api/orders`, `/api/portfolio`, `/api/listings`, `/api/funds`, `/api/tax`, `/api/otc`, ...) → `trading-service:8082`, uz `URLRewrite` koji skida `/api` prefix
- **`banka-2-core`** — Mudrinic alias-i (`/backend`, `/trading-service`, `/notification-service`), `/api/admin/dividends` → trading-service, catch-all `/api/*` → `backend:8080`, SPA catch-all `/` → `frontend:80`

> Napomena: `/api/notifications` NE ide na notification-service (taj servis je
> samo RabbitMQ consumer + Gmail SMTP, bez HTTP REST-a) — endpointi za
> notifikacije zive u banka-core i preuzima ih catch-all `/api → backend`.

## Deploy

```bash
kubectl apply -f db.yaml -f influxdb.yaml -f rabbitmq.yaml
# Sacekaj da Crunchy cluster + baze budu ready, pa:
kubectl apply -f backend.yaml -f trading-service.yaml -f notification-service.yaml -f frontend.yaml
kubectl apply -f route.yaml
kubectl apply -f seed-job.yaml -f trading-seed-job.yaml
```

Deployment-i koriste `imagePullPolicy: Always` i povlaca GHCR image-e (`ghcr.io/raf-si-2025/banka-2-*`),
pa `kubectl rollout restart deployment/<name> -n banka-2` povlaci najnoviji `:latest`.

## Tim

Banka 2025 Tim 2, Racunarski fakultet 2025/26. DevOps: Luka Stojiljkovic.
