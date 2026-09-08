# Infraestructura Kata E1

Runtime en Google Cloud, proyecto `round-seeker-309101`. No hay Render ni Supabase en el camino.

## Qué corre

| Pieza | Dónde | Para qué |
| --- | --- | --- |
| Cluster `katae1` | GKE Autopilot, `us-central1` | Kubernetes de producción |
| Pod `front` (2 réplicas) | Namespace `katae1` | React + nginx. Expone HTTP y hace proxy de `/api` a Nest |
| Pod `usuarios` (2 réplicas) | Namespace `katae1` | NestJS: login token, health, circuit breaker de RabbitMQ |
| Pod `rabbitmq` (1 réplica) | Namespace `katae1` | Broker AMQP interno. No sale a internet |
| Artifact Registry `katae1` | `us-central1-docker.pkg.dev` | Imágenes `front` y `usuarios` |
| Firebase Auth | Identity Platform | Google, correo y teléfono. Emite JWT |
| GitHub Actions | Repos `pipeelyo/KataE1*` | Build, push de imagen y `kubectl` |

URL pública: [http://35.254.92.206](http://35.254.92.206) o [http://35.254.92.206.sslip.io](http://35.254.92.206.sslip.io) (LoadBalancer HTTP, sin TLS).

## Cómo se habla el tráfico

```
Navegador
  → LoadBalancer :80  (Service front)
    → nginx
      → estáticos React
      → /api/*  → Service usuarios:3000
  → Firebase Auth (HTTPS)  Google / correo / SMS
Nest
  → JWKS de Firebase (HTTPS) para validar el token
  → TCP rabbitmq:5672  (circuit breaker; Nest aún no publica colas)
```

`KataE1front` y `KataE1Usuarios` cambian código e imagen. `KataE1infra` cambia el YAML del cluster (`k8s/gke.yaml`). `kind.yaml` es solo una copia local opcional; hoy no está encendida.

## CI/CD

Push a `main`:

1. Front o usuarios: Docker build → Artifact Registry (tag = SHA) → `kubectl set image` + rollout.
2. Infra: `kubectl apply -f k8s/gke.yaml`.

GitHub entra a GCP con Workload Identity Federation (cuenta `github-deploy@...`), sin JSON key.

## Resiliencia que sí hay

- **Readiness probes.** Kubernetes no manda tráfico a un pod hasta que responde: front `/`, usuarios `/api/health`, RabbitMQ TCP `:5672`.
- **Autopilot.** Si un nodo o un pod muere, GKE crea otro. Con 2 réplicas de front y usuarios, borrar un pod no apaga la URL.
- **nginx.** Timeout corto y `proxy_next_upstream` si una réplica de Nest falla.
- **JWKS de Firebase.** `jose` cachea las llaves; un blip de Google no obliga a bajarlas en cada request.
- **Circuit breaker de RabbitMQ** en Nest (`GET /api/resilience`):
  - `closed`: ping TCP a `:5672` ok.
  - 3 fallos → `open`: deja de spamear al broker 10 s y responde `circuit-open`.
  - Luego `half-open`: un ping de prueba; si entra, vuelve a `closed`.
- `/api/health` **no** depende de RabbitMQ. Si el broker cae, el login sigue vivo.

## Lo que no es resiliente todavía

- RabbitMQ: 1 réplica y **sin disco**. Si el pod muere, se pierden las colas.
- Nest aún no publica ni consume mensajes; el breaker solo vigila que el puerto AMQP responda.
- HTTP sin TLS. Google login funciona por el `authDomain` de Firebase.
- Sin HPA ni PodDisruptionBudget.

## Cómo validar

Credenciales del cluster:

```bash
gcloud container clusters get-credentials katae1 --region=us-central1 --project=round-seeker-309101
```

### 1. Salud

```bash
curl -s http://35.254.92.206/api/health
curl -s http://35.254.92.206/api/resilience
```

Esperado: `{"status":"ok"}` y `rabbit: "up"` con `circuit.state: "closed"`.

### 2. Matar un pod de front o Nest

```bash
kubectl -n katae1 delete pod -l app=front --wait=false
curl -s -o /dev/null -w "%{http_code}\n" http://35.254.92.206/
kubectl -n katae1 get pods -l app=front
```

Esperado: HTTP 200 mientras Autopilot levanta el reemplazo. Lo mismo con `-l app=usuarios` y `/api/health`.

### 3. Circuit breaker (RabbitMQ caído)

```bash
kubectl -n katae1 scale deploy/rabbitmq --replicas=0
sleep 5
curl -s http://35.254.92.206/api/resilience
# repetir 3 veces hasta ver "circuit-open"
curl -s http://35.254.92.206/api/health
kubectl -n katae1 scale deploy/rabbitmq --replicas=1
sleep 15
curl -s http://35.254.92.206/api/resilience
```

Esperado: `/api/resilience` pasa a `degraded` / `circuit-open`. `/api/health` sigue `ok`. Al volver RabbitMQ, el circuito cierra.

### 4. RabbitMQ no es durable

```bash
kubectl -n katae1 delete pod -l app=rabbitmq
```

El pod vuelve, las colas in-memory no. Para persistir haría falta un PVC.
