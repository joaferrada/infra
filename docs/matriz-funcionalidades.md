# Matriz de funcionalidades AndesStay (hasta EP3)

Alcance: autenticacion, reservas, estados, mensajeria y administracion de RabbitMQ. Kafka, reportes y auditoria quedan para la etapa siguiente (ver final).

**Estados:** Implementada = codigo + evidencia verificable · Parcial = funciona con una limitacion indicada · No implementada.

Repos: `reservations` = ms-andesstay-reservations (rama `feature/rabbitmq`), `bff` = ms-andesstay-bff (rama `feature/rabbitmq`), `frontend` = frontend-andesstay (rama `feature/rabbitmq`), `notify` = ms-andesstay-notify, `rabbitadmin` = ms-andesstay-rabbitadmin, `infra` = infra (este repo).

## 1. Autenticacion y seguridad

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 1 | Login con Microsoft Entra ID (MSAL, popup) | Implementada | frontend | commits `74bf612`, `b285e79`; `src/app/app.component.ts` |
| 2 | Resource Server JWT (issuer Entra ID) | Implementada | bff, reservations | `bff/security/SecurityConfig.java`; reservations commit `61c6c3d` |
| 3 | Mapeo del claim `roles` a `ROLE_*` | Implementada | bff, rabbitadmin | `JwtAuthConverter.java` (bff y rabbitadmin `e3e4e20`) |
| 4 | Autorizacion por rol en BFF (`Admin`, `Operador_Dominio`, `Cliente_Dominio`) | Implementada | bff | `@PreAuthorize` en `ReservationsBffController` |
| 5 | Reenvio del JWT BFF -> reservations | Implementada | bff | commits `f1b467d`, `5987de7` |
| 6 | Autorizacion por rol dentro de reservations | Parcial | reservations | `@PreAuthorize` declarado, pero `SecurityConfig` no tiene `@EnableMethodSecurity`: el control efectivo de roles se hace en el BFF |
| 7 | CORS para localhost y GitHub Pages | Implementada | bff | commit `a7b0ac8` |
| 8 | Seguridad de la API admin: JWT + rol Admin, apagable con `andesstay.security.enabled` | Implementada | rabbitadmin | `AdminApiSecurityTest` (401 / 403 / 200), commit `8101d01` |

## 2. Reservas

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 9 | CRUD `/api/reservations` (JPA + H2) | Implementada | reservations | commits `306bbc2`, `952a780` |
| 10 | Logica de negocio en `ReservationService` | Implementada | reservations | commit `690dbb6` |
| 11 | Proxy CRUD en BFF | Implementada | bff | commits `2cbd307`, `0368ce7` |
| 12 | Listado, creacion y eliminacion en el frontend | Implementada | frontend | `src/app/reservations.component.ts` |
| 13 | Despliegue del frontend en GitHub Pages | Implementada | frontend | `.github/workflows/deploy.yml`, https://joaferrada.github.io/frontend-andesstay/ |

## 3. Estados de la reserva

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 14 | `estado` (enum), `creadaEn`, `actualizadaEn`; toda reserva nueva parte en CREADA | Implementada | reservations | commit `d78e81c`; `ReservationServiceTest.crearSiempreParteEnCreadaYPublica` |
| 15 | Maquina de estados (sin check-in sin confirmar) | Implementada | reservations | `ReservationStatus.java`; `ReservationStatusTest` (17 casos) |
| 16 | `PUT /api/reservations/{id}/status` (200/400/404/409) | Implementada | reservations | commit `690dbb6`; `ReservationServiceTest.checkinSinConfirmarLanza409YNoPublica` |
| 17 | Filtro `GET /api/reservations?status=` | Implementada | reservations, bff | commits `690dbb6`, `49e49c6` |
| 18 | Proxy `PUT /{id}/status` en BFF (Admin u Operador) | Implementada | bff | commit `49e49c6` |
| 19 | Botones de cambio de estado y filtro en el frontend | No implementada | frontend | Pendiente: se desarrolla aparte. El backend ya expone `PUT /{id}/status` y `?status=` via BFF |

## 4. Mensajeria RabbitMQ

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 20 | Topologia: 3 exchanges durables, 6 colas, 9 bindings | Implementada | reservations, notify | `RabbitTopologyConfig.java` (commits `1449db8`, `fba0793`); captura de Management UI > Exchanges/Queues |
| 21 | Colas principales con DLX + dead-letter routing key; DLQ con TTL de 7 dias | Implementada | reservations, notify | Management UI > Queues > features `DLX`, `DLK`, `TTL` |
| 22 | Nombres centralizados en `andesstay.messaging.*` + record `@ConfigurationProperties` | Implementada | reservations, notify | `MessagingProperties.java`, `application.yaml` |
| 23 | Envelope comun (record) y mensajes persistentes JSON con `messageId=eventId` | Implementada | reservations | `EventEnvelope.java`; `EventEnvelopeSerializationTest` (commit `9ab5f34`) |
| 24 | Publicacion por cambio de estado usando `cmd.direct` y `cmd.topic` | Implementada | reservations | `CommandType.planFor`; `RabbitReservationNotificationPublisherTest` |
| 25 | Publisher confirms (correlated), returns y `mandatory=true` | Implementada | reservations, rabbitadmin | `RabbitMessagingConfig.java`; rabbitadmin `POST /messages` responde `confirmed:true` y 422 si no es enrutable |
| 26 | Reserva se guarda aunque RabbitMQ este caido | Implementada | reservations | `RabbitReservationNotificationPublisherTest.brokerCaidoNoPropagaExcepcion` |
| 27 | Dominio desacoplado de RabbitMQ (interfaz `ReservationNotificationPublisher`) | Implementada | reservations | `service/ReservationNotificationPublisher.java` |
| 28 | `@RabbitListener` por dominio (email/, housekeeping/, voucher/, deadletter/) | Implementada | notify | commit `b978895` |
| 29 | ACK manual, prefetch configurable | Implementada | notify | `application.yaml` (`acknowledge-mode: manual`, `NOTIFY_PREFETCH`) |
| 30 | Idempotencia por `eventId` (duplicado -> ACK + log) | Implementada | notify | `ManualAckMessageProcessorTest.duplicadoHaceAckSinReprocesar`; log `[ACK-DUPLICADO]` |
| 31 | Error no recuperable -> `basicNack(requeue=false)` -> DLQ | Implementada | notify | `ManualAckMessageProcessorTest.jsonInvalidoHaceNackSinRequeue`; log `[NACK->DLQ]` |
| 32 | Reintento con `x-retry-count` y `max-attempts=3`, luego DLQ | Implementada | notify | `AckDecisionPolicyTest`, `ManualAckMessageProcessorTest.transitorio*`; header `x-retry-count=2` en la DLQ |
| 33 | Excepcion inesperada tratada como no recuperable | Implementada | notify | `AckDecisionPolicyTest.excepcionInesperadaSeTrataComoNoRecuperable` |
| 34 | Listener de DLQ con log de `x-death` y autoStartup configurable | Implementada | notify | `DeadLetterListener.java`; log `[DLQ] x-death reason=rejected` |
| 35 | Simulacion de fallos (TRANSIENT, failTimes, FATAL) | Implementada | notify | `FailureSimulator.java`; README de notify |
| 36 | Cluster RabbitMQ de 2 nodos (classic_config, misma cookie) | Implementada | infra | `mq/compose.yml` (commit `19cb601`); `rabbitmqctl cluster_status` muestra 2 running nodes; captura Management UI > Overview > Nodes |
| 37 | Health check con estado de RabbitMQ | Implementada | notify, rabbitadmin | `GET :8083/actuator/health` -> `rabbit: UP` |

## 5. Administracion de RabbitMQ

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 38 | `GET/POST/DELETE /api/admin/queues` (+ purga) | Implementada | rabbitadmin | commit `13f22e6`; `AdminApiHttpStatusTest` |
| 39 | `POST/DELETE /api/admin/exchanges` | Implementada | rabbitadmin | `ExchangeAdminController.java` |
| 40 | `POST/DELETE /api/admin/bindings` | Implementada | rabbitadmin | `BindingAdminController.java` |
| 41 | `POST /api/admin/messages` (envelope o `rawBody` malformado) | Implementada | rabbitadmin | `MessageAdminController.java`; demo de JSON invalido -> `q.cmd.email.dlq` |
| 42 | Codigos HTTP 201/202/204/400/404/409/422/503 | Implementada | rabbitadmin | `AdminApiHttpStatusTest` (9 casos) |
| 43 | Logica encapsulada en `RabbitAdminService` (RabbitAdmin) | Implementada | rabbitadmin | commit `9e7f2b4` |
| 44 | Validacion con DTOs + jakarta.validation (`@AmqpName`, sin `amq.`, enum de tipo, `@Positive`) | Implementada | rabbitadmin | `AdminDtoValidationTest` (18 casos), commit `1a97c74` |
| 45 | `@RestControllerAdvice` con errores JSON claros | Implementada | rabbitadmin | `ApiExceptionHandler.java` |

## 6. Documentacion y pruebas

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 46 | README por microservicio (como correr, endpoints, fallos, curl) | Implementada | todos | `README.md` de reservations (`6a6f7f5`), notify (`47c3252`), rabbitadmin (`bd68361`) |
| 47 | Diagrama de topologia y estrategia ACK/NACK/DLQ | Implementada | infra | `docs/arquitectura-mensajeria.md` (commit `6fa872b`) |
| 48 | Tests sin broker que pasan con `mvnw.cmd clean package` | Implementada | reservations (27), notify (18), rabbitadmin (31), bff (1) | salida de `BUILD SUCCESS` |

## Resumen

| Estado | Cantidad | % |
|---|---|---|
| Implementada | 46 | 95,8 % |
| Parcial | 1 | 2,1 % |
| No implementada | 1 | 2,1 % |
| **Total** | **48** | |

## Etapa siguiente (fuera de alcance de la EP3)

| Funcionalidad | Estado |
|---|---|
| Streaming de eventos con Kafka | No implementada (etapa siguiente) |
| Microservicio de reportes (report) | No implementada (etapa siguiente) |
| Microservicio de auditoria (audit) | No implementada (etapa siguiente) |
| Quorum queues (requiere 3 nodos) | No implementada (mejora) |
| Idempotencia persistente (Redis o BD) | No implementada (mejora) |
