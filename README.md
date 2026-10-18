# AndesStay - infra

Infraestructura local y documentacion transversal de AndesStay.

```
mq/compose.yml       cluster RabbitMQ 4 de 2 nodos (rabbit1 + rabbit2)
mq/rabbitmq.conf     peer discovery classic_config
mq/.env.example      credenciales y Erlang cookie (copiar a .env, que no se versiona)
docs/arquitectura-mensajeria.md   topologia, contrato de mensajes, ACK/NACK/DLQ
docs/matriz-funcionalidades.md    matriz de funcionalidades hasta la EP3
```

## Levantar RabbitMQ

```powershell
cd infra\mq
copy .env.example .env      # opcional: cambiar usuario, clave y cookie
docker compose up -d
docker exec andesstay-rabbit1 rabbitmqctl cluster_status   # Running Nodes: rabbit@rabbit1, rabbit@rabbit2
```

| Nodo | AMQP | Management UI |
|---|---|---|
| rabbit1 | `localhost:5672` | http://localhost:15672 |
| rabbit2 | `localhost:5673` | http://localhost:15673 |

Usuario por defecto `guest` / `guest` (solo local). Los microservicios se conectan a ambos nodos con:

```powershell
$env:RABBITMQ_ADDRESSES="localhost:5672,localhost:5673"
```

Para bajar todo: `docker compose down` (agregar `-v` para borrar los volumenes de datos).

## Orden de arranque para la demo

1. `infra/mq`: `docker compose up -d`
2. `ms-andesstay-notify` (8083): declara la topologia y empieza a consumir
3. `ms-andesstay-reservations` (8081) y `ms-andesstay-bff` (8080)
4. `ms-andesstay-rabbitadmin` (8084), con `ANDESSTAY_SECURITY_ENABLED=false` para probar sin token
5. `frontend-andesstay`: usa la URL de `src/enviroments/enviroment.ts` (API Gateway de AWS). Para la demo local de mensajeria basta con rabbitadmin + notify
