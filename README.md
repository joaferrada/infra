# AndesStay - infra

Infraestructura local y documentacion transversal de AndesStay.

```
compose.yml               todo el backend: cluster RabbitMQ 4 de 2 nodos + 7 microservicios
rabbitmq/rabbitmq.conf    peer discovery classic_config
.env.example              variables del compose (copiar a .env, que no se versiona)
docs/arquitectura.md              servicios, puertos y flujos
docs/arquitectura-mensajeria.md   topologia, contrato de mensajes, ACK/NACK/DLQ
docs/matriz-funcionalidades.md    matriz de funcionalidades
```

## Levantar todo

Requisitos: Docker Desktop y el wallet de Oracle ADB descomprimido (por defecto en `C:/oracle/wallet_andesstay`). Los repos `ms-andesstay-*` deben estar como carpetas hermanas de `infra`, porque cada imagen se construye desde `../ms-andesstay-<servicio>`.

```powershell
cd infra
copy .env.example .env      # completar RESERVATIONS_DB_PASSWORD, CATALOG_DB_PASSWORD y AUDIT_DB_PASSWORD
docker compose up -d --build
docker compose ps           # todos deben quedar (healthy)
```

Orden de arranque (por `depends_on` + healthchecks de `/actuator/health`): rabbit1 -> rabbit2 -> catalog, audit, notify, rabbitadmin -> reservations -> report, bff.

| Variable | Default | Uso |
|---|---|---|
| `RABBITMQ_USERNAME` / `RABBITMQ_PASSWORD` | `guest` / `guest` | usuario del broker (nodos y microservicios) |
| `RABBITMQ_ERLANG_COOKIE` | `andesstay-local-dev-cookie` | cookie compartida por los 2 nodos |
| `WALLET_DIR` | `C:/oracle/wallet_andesstay` | wallet montado en `/wallet` (solo lectura) |
| `RESERVATIONS_DB_PASSWORD` | - | usuario `ANDESSTAY` |
| `CATALOG_DB_PASSWORD` | - | usuario `ANDESSTAY_CATALOG` |
| `AUDIT_DB_PASSWORD` | - | usuario `ANDESSTAY_AUDIT` |
| `RABBITADMIN_SECURITY_ENABLED` | `true` | `false` deja `/api/admin/**` sin token (solo pruebas locales) |
| `JAVA_TOOL_OPTIONS` | `-XX:MaxRAMPercentage=75` | opciones de la JVM (cada contenedor tiene `mem_limit: 512m`) |

## Servicios

| Servicio | Contenedor | Puerto host | URL |
|---|---|---|---|
| bff | `andesstay-bff` | 8080 | http://localhost:8080 |
| reservations | `andesstay-reservations` | 8081 | http://localhost:8081 |
| notify | `andesstay-notify` | 8083 | http://localhost:8083 |
| rabbitadmin | `andesstay-rabbitadmin` | 8084 | http://localhost:8084 |
| catalog | `andesstay-catalog` | 8085 | http://localhost:8085 |
| audit | `andesstay-audit` | 8086 | http://localhost:8086 |
| report | `andesstay-report` | 8087 | http://localhost:8087 |
| rabbit1 | `andesstay-rabbit1` | 5672 / 15672 | AMQP `localhost:5672`, Management UI http://localhost:15672 |
| rabbit2 | `andesstay-rabbit2` | 5673 / 15673 | AMQP `localhost:5673`, Management UI http://localhost:15673 |

Salud de cada microservicio: `GET http://localhost:<puerto>/actuator/health` (sin token).

Dentro de la red `andesstay-net` los servicios se llaman por nombre: RabbitMQ en `rabbit1:5672,rabbit2:5672`, y `http://reservations:8081`, `http://catalog:8085`, `http://audit:8086`, `http://report:8087`.

## Logs y verificacion

```powershell
docker compose logs -f notify                              # logs de un servicio
docker exec andesstay-rabbit1 rabbitmqctl cluster_status   # Running Nodes: rabbit@rabbit1, rabbit@rabbit2
curl.exe -u guest:guest http://localhost:15672/api/queues  # las 6 colas q.cmd.* y sus DLQ
```

Reconstruir un solo servicio despues de un cambio: `docker compose up -d --build reservations`.

## Bajar todo

```powershell
docker compose down        # detiene y elimina los contenedores
docker compose down -v     # ademas borra los volumenes de datos de RabbitMQ
```

## Demo local de mensajeria

Con todo arriba, notify y reservations declaran la topologia y notify empieza a consumir. Para probar sin token, levantar rabbitadmin con `RABBITADMIN_SECURITY_ENABLED=false` en `.env` y usar los ejemplos del README de `ms-andesstay-rabbitadmin`. El frontend (`frontend-andesstay`) usa la URL del API Gateway de AWS de `src/enviroments/enviroment.ts`.
