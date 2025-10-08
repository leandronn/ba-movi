# BA-MOVI — README interno

> **Propósito:** guía operativa para desarrollo, pruebas, despliegue y operación de BA-MOVI.
> **Alcance:** resumen técnico + “cómo correrlo”, **contratos** y **SLOs** mínimos, **eventos**, **seguridad**, **monitoreo** y **checklists**.

---

## 0. TL;DR para arrancar

```bash
# 1) Clonar y preparar
git clone <repo>
cd ba-movi
cp .env.example .env        # completar secretos locales

# 2) Levantar base y dependencias locales
docker compose up -d       # pg + postgis + redis + nats/kafka + minio

# 3) Migraciones y datos semilla
make migrate
make seed

# 4) Iniciar servicios core en modo dev
make dev-api               # api gateway + servicios dominio
make dev-web               # backoffice
make dev-mobile            # hot reload app campo (opcional)

# 5) Story de prueba
make demo-programado       # crea corte de ejemplo y publica en portal-sandbox
```

---

## 1. Arquitectura rápida

**Canales**

* **App campo**: credencial QR, geocercas, checklists, evidencias, offline-first.
* **Backoffice**: tablero RT, planificación, permisos, publicación, BI.
* **Portal**: mapa, cronograma, filtros por zona, accesibilidad WCAG.

**Servicios de dominio**

* **IAM** · **Cortes_Obras** · **Workflow** · **Reglas** · **Dimensionamiento** · **Geocercas_Checklists** · **Evidencias** · **Publicacion** · **Waze_Publisher** · **Waze_Ingest** · **Sugerencias** · **Notificaciones** · **Auditoria** · **Observabilidad**.

**Plataformas**

* **OLTP_Geo** (PostGIS) · **Cache_RT** (Redis) · **Bus_Eventos** (Kafka o NATS) · **Objetos** (S3-like) · **BI_Lake** (Azure Data Lake o equivalente).

**Topología mínima (Mermaid)**

```mermaid
flowchart LR
  subgraph Canales
    app[App campo]
    back[Backoffice]
    portal[Portal]
  end

  subgraph Backend
    api[API Gateway]
    co[Cortes y Obras]
    wf[Workflow]
    rg[Reglas]
    dm[Dimensionamiento]
    gc[Geocercas Checklists]
    ev[Evidencias]
    pb[Publicacion]
    wzp[Waze Publisher]
    wzi[Waze Ingest]
    sg[Sugerencias]
    nt[Notificaciones]
    iam[IAM]
    au[Auditoria]
  end

  subgraph Datos
    db[OLTP Geo]
    ca[Cache RT]
    bus[Bus Eventos]
    s3[Objetos]
    bi[BI Lake]
  end

  app -- grpc-rest-proto --> api
  back -- rest-ws --> api
  portal -- rest --> api

  api --> iam
  api --> co --> db
  api --> wf --> bus
  api --> rg
  api --> dm
  api --> gc
  api --> ev --> s3
  api --> pb
  pb --> wzp
  wzi --> bus
  sg --> bus
  api --> nt
  api --> au
  back <-- ws-diffs --> ca

  bus --> sg
  bus --> wf
  bus --> pb
  back <-- ws --> api
```

---

## 2. Repos y carpetas

```
/apps
  /backoffice         # Next.js
  /portal             # SSR portal
  /mobile             # Flutter app campo
/services
  /api-gateway
  /iam
  /cortes-obras
  /workflow
  /reglas
  /dimensionamiento
  /geocercas-checklists
  /evidencias
  /publicacion
  /waze-publisher
  /waze-ingest
  /sugerencias
  /notificaciones
  /auditoria
/infrastructure
  /terraform
  /kubernetes
  /docker
  compose.yaml
/specs
  openapi/            # contratos REST
  proto/              # contratos gRPC
  events/             # esquemas bus (json/avro/proto)
/db
  migrations/
  seeds/
/ops
  dashboards/
  runbooks/
  alerts.yml
```

---

## 3. Configuración y secretos

`.env.example` (local):

```dotenv
# API
API_PORT=8080
API_BASE_URL=http://localhost:8080
JWT_ISSUER=ba-movi-local
JWT_PUBLIC_KEY_PATH=./keys/jwt.pub

# DB geoespacial
PG_URL=postgres://movi:password@localhost:5432/movi
PG_SSLMODE=disable

# Redis y bus
REDIS_URL=redis://localhost:6379
KAFKA_BROKERS=localhost:9092
NATS_URL=nats://localhost:4222

# Objetos
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minio
S3_SECRET_KEY=minio123
S3_BUCKET_EVIDENCIAS=ba-movi-evidencias

# Waze
WAZE_API_URL=https://partner.waze.com/...
WAZE_TOKEN=changeme

# H3
H3_RESOLUTION=8
```

> **Prod:** secretos en KMS/Vault; nunca en repo.

---

## 4. Cómo correr localmente

* **Dependencias**: Docker, Make, Node LTS, Flutter SDK, Java o JVM si aplica.
* **Servicios base:** `docker compose up -d` (PostGIS, Redis, Kafka/NATS, MinIO).
* **Migraciones:** `make migrate`; **seeds:** `make seed`.
* **Iniciar dominio:** `make dev-api` (gateway + dominio), `make dev-web`, `make dev-mobile`.

---

## 5. Contratos — REST y gRPC

**REST base** `https://.../api/v1`

* **Cortes**
  `POST /cortes` · `GET /cortes?bbox&estado&comuna&page&page_size` · `PATCH /cortes/{id}` · `POST /cortes/{id}/desvios` · `GET /cortes/{id}/conflictos`
* **Workflow**
  `POST /workflow/{corte_id}/aprobar` · `.../cerrar` · `.../migrar_a_programado`
* **Reglas**
  `POST /reglas/evaluar`
* **Dimensionamiento**
  `POST /dimensionamiento/calcular`
* **Evidencias**
  `POST /evidencias` (multipart con `hash`) → URL firmada
* **Publicación**
  `POST /publicacion/portal` · `GET /publicacion/estado?corte_id=...`
* **Waze**
  `POST /waze/closures` (create-update-revoke) · `POST /waze/ingest` (batch normalizado)
* **Sugerencias**
  `POST /ops/sugerencias` · `POST /ops/sugerencias/{id}/aplicar`
* **Credencial**
  `GET /credencial/verify?q=payload_qr`
* **Auditoría**
  `GET /auditoria?...`

**gRPC**

* **Ingesta móvil**: paquete delta idempotente con `event_id`, `ts`, `lat`, `lon`, `accuracy`, `speed`, `heading`, `logical_clock`.

**Transversales**

* Auth: OIDC LDAP + JWT Bearer · RBAC por rol/secretaría
* Idempotencia: `Idempotency-Key` en unsafe · `event_id` en ingesta
* Paginación: `page/page_size` o cursor · filtros `bbox, comuna, estado, impacto, updated_since`
* Errores: `{ code, message, details, correlation_id }`
* Caching: `ETag`, `If-None-Match` en GETs

---

## 6. Eventos de dominio

**Topics sugeridos**

* `loc_update` · `geofence_event` · `evidence_added` · `workflow_state_changed`
* `waze_publish_requested|succeeded|failed` · `traffic_ingest`
* `ops_suggestion_emitted|applied` · `sla_timer_fired` · `nc_opened|closed`

**Ejemplos**

```json
// workflow_state_changed
{ "entity":"corte", "id":"C-2025-000123", "from":"APROBADO", "to":"EJECUCION", "ts":"..." }

// traffic_ingest
{ "source":"waze", "cell_id":"8a2a1072abfffff", "ts":"...", "metrics":{"speed_avg_kmh":12.3, "jam_severity":3} }
```

---

## 7. Reglas declarativas y dimensionamiento

* Reglas en **YAML JSON** · versionadas · auditables.
* Inputs: tipología, vía, carriles, horario, entorno, fase, zona.
* Outputs: dotación, puestos, turnos, señalización, ventanas, comunicación, rutas.
* **Explain**: lista de reglas disparadas y por qué.
* Overrides por **corredor planificado alta prioridad**.

---

## 8. Flujos clave

**Programado — E2E**

1. Solicitud → 2. Validaciones conflicto → 3. Condiciones permiso →
2. Dimensionamiento → 5. Aprobación → 6. Publicación anticipada (portal y Waze) →
3. Ejecución y supervisión → 8. Cierre → 9. BI.

**Imprevisto — E2E**

1. Alta express → 2. Permiso contingente → 3. Mitigación →
2. Migrar a programado si persiste → 5. Supervisión y cierre.

**Sugerencias por tránsito**

* Ingesta Waze → normalización H3 → gap real vs esperado → recomendación `add_agent activate_detour shift_window pause_ops` → botón **Aplicar** → orden operativa.

---

## 9. SLOs y KPIs

**SLOs**

* Evento crítico → tablero **p95 ≤ 300 ms**
* Waze publish **p95 ≤ 2 s**, **success ≥ 99 %**
* Evidencias **p95 ≤ 3 s**, **primer intento ≥ 98 %**
* Ingest drop **≤ 1 % por hora**
* Disponibilidad **99,9 %**

**KPIs clave**

* ingest.latency.ms p50 p95 · ingest.drop.rate · rt.push.latency.ms p50 p95 · rt.ws.sessions.active
* waze.publish.success.rate · waze.publish.latency.ms · waze.reconcile.drift.count
* evidence.upload.success.rate · evidence.upload.latency.ms · portal.ttfb.ms · portal.lcp.ms p75

---

## 10. Monitoreo y alertas

* **Dashboards**: Tiempo Real, Publicación Waze, Evidencias, Workflow SLA NC, Portal Accesibilidad, Seguridad Auditoría.
* **Alertas P1**:

  * `rt.push.latency.p95 > 1500 ms` 5 min → oncall + degradar granularidad H3
  * `waze.publish.success < 95%` 10 min → breaker + DLQ + reconciliación
* **Alertas P2**: `ingest.drop > 2%` 10 min → throttle cliente + backpressure WS

---

## 11. Seguridad

* **AuthN/Z**: OIDC LDAP, JWT, RBAC por rol/secretaría, scopes por recurso.
* **Credencial QR**: firma y revocación, payload mínimo, auditoría de verificaciones.
* **Evidencias**: hash, URL firmada, cifrado en reposo, retención.
* **Headers**: TLS, HSTS, CSP, X-Content-Type-Options, Referrer-Policy.
* **Rate limits** y **circuit breakers** en integraciones externas.
* **Auditoría**: accesos, publicaciones, cambios de estado, verificaciones QR.

---

## 12. Tests

* **Contract tests**: OpenAPI y Proto; validadores de `geometry`, `window`, `bbox`, `H3`.
* **E2E**: historias Programado e Imprevisto.
* **Carga**: ingest 2× y 3× picos; Waze publish ráfagas 100 upd/min.
* **Resiliencia**: caída de Waze simulada → DLQ + reconciliación.

Comandos:

```bash
make test
make test-contracts
make test-load
```

---

## 13. Despliegue

* **Infra como código**: Terraform.
* **Kubernetes**: blue green o canary para ingesta y publisher.
* **CI CD**: contract tests en PR; smoke tests post-deploy; feature flags por zona/fase.
* **Entornos**: dev · staging · prod; datos sintéticos en perf.

---

## 14. Backups y DR

* OLTP snapshot diario + WAL continuo · retención 30 días.
* Objetos con lifecycle y opción WORM para evidencias críticas.
* **RPO ≤ 15 min**, **RTO ≤ 60 min** · prueba DR trimestral.

---

## 15. Glosario rápido

* **H3**: rejilla hexagonal para indexación y suscripción por celdas.
* **NC**: No Conformidad.
* **SLA**: Acuerdo de Nivel de Servicio.
* **TTRO**: Temporary Traffic Regulation Order, compatibilidad de datasets.
* **WCAG**: estándar de accesibilidad web.
* **DLQ**: Dead Letter Queue.

---

## 16. Enlaces internos

* **Especificaciones OpenAPI**: `/specs/openapi/index.yaml`
* **.proto gRPC**: `/specs/proto/`
* **Eventos Bus**: `/specs/events/`
* **Dashboards**: `/ops/dashboards/`
* **Runbooks**: `/ops/runbooks/`
* **Alertas**: `/ops/alerts.yml`
* **Migrations**: `/db/migrations/`
* **Seeds**: `/db/seeds/`

---

## 17. Checklist de PR

* [ ] Contratos actualizados (OpenAPI y Proto)
* [ ] Tests pasan (unit, contract, e2e)
* [ ] Migraciones revertibles y documentadas
* [ ] Métricas y logs agregados
* [ ] Flags configurados por zona/fase
* [ ] Seguridad revisada (scopes, rate limit, secretos)
