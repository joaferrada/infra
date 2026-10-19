# Matriz de funcionalidades AndesStay

Alcance: autenticacion, reservas, estados, mensajeria y administracion de RabbitMQ, catalogo de unidades, auditoria y reporteria. Kafka/streaming queda para la etapa siguiente (ver final). Diagrama de servicios y puertos: [arquitectura.md](arquitectura.md).

**Estados:** Implementada = codigo + evidencia verificable · Parcial = funciona con una limitacion indicada · No implementada.

Repos: `reservations` = ms-andesstay-reservations (rama `feature/rabbitmq`), `bff` = ms-andesstay-bff (rama `feature/rabbitmq`), `frontend` = frontend-andesstay (rama `feature/rabbitmq`), `notify` = ms-andesstay-notify, `rabbitadmin` = ms-andesstay-rabbitadmin, `catalog` = ms-andesstay-catalog, `audit` = ms-andesstay-audit, `report` = ms-andesstay-report (estos tres en rama `main`), `infra` = infra (este repo).

## 1. Autenticacion y seguridad

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 1 | Login con Microsoft Entra ID (MSAL, popup) | Implementada | frontend | commits `74bf612`, `b285e79`; `src/app/app.component.ts` |
| 2 | Resource Server JWT (issuer Entra ID) | Implementada | bff, reservations | `bff/security/SecurityConfig.java`; reservations commit `61c6c3d` |
| 3 | Mapeo del claim `roles` a `ROLE_*` | Implementada | bff, rabbitadmin | `JwtAuthConverter.java` (bff y rabbitadmin `e3e4e20`) |
| 4 | Autorizacion por rol en BFF con los 4 roles (`Admin`, `Operador_Dominio`, `Cliente_Dominio`, `Auditor`) | Implementada | bff | `@PreAuthorize` en `ReservationsBffController`; `ReservationsBffControllerSecurityTest` (200/403 por rol) |
| 5 | Reenvio del JWT BFF -> reservations | Implementada | bff | commits `f1b467d`, `5987de7` |
| 6 | Autorizacion por rol dentro de reservations; el Cliente solo ve/actualiza sus reservas (`huespedEmail` = principal; ajena = 404) | Implementada | reservations | `@EnableMethodSecurity` + `JwtAuthConverter`; `ReservationAccessTest`, `ReservationServiceTest.listarConClienteSoloDevuelveLasSuyas` |
| 7 | CORS en AWS API Gateway (HTTP API) para `localhost:4200` y `joaferrada.github.io`; el BFF no configura CORS | Implementada | infra (API Gateway), bff | README del BFF "CORS y seguridad en API Gateway"; captura API Gateway > CORS |
| 8 | Seguridad de la API admin: JWT + rol Admin, apagable con `andesstay.security.enabled` | Implementada | rabbitadmin | `AdminApiSecurityTest` (401 / 403 / 200), commit `8101d01` |

## 2. Reservas

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 9 | CRUD `/api/reservations` (JPA; Oracle ADB, H2 solo en tests) | Implementada | reservations | commits `306bbc2`, `952a780` |
| 10 | Logica de negocio en `ReservationService` | Implementada | reservations | commit `690dbb6` |
| 11 | Proxy CRUD en BFF | Implementada | bff | commits `2cbd307`, `0368ce7` |
| 12 | Listado, creacion y eliminacion en el frontend | Implementada | frontend | `src/app/reservations.component.ts` |
| 13 | Despliegue del frontend en GitHub Pages | Implementada | frontend | `.github/workflows/deploy.yml`, https://joaferrada.github.io/frontend-andesstay/ |
| 52 | Datos del huesped (`huespedNombre`, `huespedEmail`) y `creadoPor` en la reserva: el Cliente los toma del token (claim `name` + principal), Admin/Operador los ingresan en el body con `@NotBlank`/`@Email` (400 si faltan); emails y vouchers usan `huespedEmail`/`huespedNombre` | Implementada | reservations | `dto/ReservationCreateRequest`, `security/HuespedResolver`; `HuespedResolverTest`, `ReservationControllerSecurityTest.adminSinEmailDelHuespedRecibe400ConMensajeClaro`, `EventEnvelopeSerializationTest` |

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
| 46 | README por microservicio (como correr, endpoints, roles, variables, fallos, curl) | Implementada | todos | `README.md` de reservations (`6a6f7f5`, `b9ab9c7`), notify (`47c3252`), rabbitadmin (`bd68361`), bff (`5f0c960`), catalog (`d09394a`), audit (`f0d054f`), report (`8ccf17d`) |
| 47 | Diagrama de topologia y estrategia ACK/NACK/DLQ | Implementada | infra | `docs/arquitectura-mensajeria.md` (commit `6fa872b`) |
| 48 | Tests sin broker, sin OCI y sin servicios externos que pasan con `mvnw.cmd clean package` | Implementada | reservations (99), notify (18), rabbitadmin (31), bff (103), catalog (59), audit (28), report (31) | salida de `BUILD SUCCESS` |
| 64 | Diagrama de servicios y puertos (frontend, API Gateway, bff, reservations, notify, rabbitadmin, catalog, audit, report, RabbitMQ, Oracle ADB) | Implementada | infra | `docs/arquitectura.md` (commit `ed8adb5`) |

## 7. Persistencia

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 49 | Persistencia de reservas en Oracle Autonomous Database (OCI Always Free) con wallet mTLS; credenciales por variables de entorno (`DB_URL`, `DB_USERNAME`, `DB_PASSWORD`) | Implementada | reservations | `application.yaml`, `ojdbc11-production` en `pom.xml`, README "Base de datos Oracle en OCI"; captura de la tabla `RESERVATIONS` en Database Actions (`SELECT * FROM RESERVATIONS`) |

## 8. Identidad y API Gateway

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 50 | App roles en Entra ID: `Admin`, `Operador_Dominio`, `Cliente_Dominio`, `Auditor` (claim `roles`) | Implementada | infra (Entra ID), bff, reservations | captura Entra ID > App registrations > App roles; tabla de roles en README del BFF y de reservations |
| 51 | JWT authorizer `entra-jwt` en AWS API Gateway (HTTP API): issuer `sts.windows.net/<tenant>/`, audience `api://<clientId>` | Implementada | infra (API Gateway) | README del BFF "CORS y seguridad en API Gateway"; captura API Gateway > Authorization; request sin token -> 401 desde API Gateway |
| 53 | Principal del JWT para tokens Entra v1: primer claim no vacio entre `preferred_username`, `upn`, `unique_name`, `email`, `sub` | Implementada | reservations | `security/JwtAuthConverter`; `ReservationAccessTest.tokenV1SinPreferredUsernameUsaUpnYNoSub` |

## 9. Catalogo, auditoria y reporteria

Integracion **REST sincronica** con el JWT del usuario propagado (sin eventos ni exchanges nuevos de RabbitMQ).

| # | Funcionalidad | Estado | Responsable | Evidencia verificable |
|---|---|---|---|---|
| 54 | Catalogo de unidades (`HABITACION`, `CABANA`, `LODGE`, `DOMO`): CRUD `/api/catalog/units` con filtros `tipo`, `hostal`, `soloDisponibles` y baja logica (`activo=false`) | Implementada | catalog | commit `66a41e8`; `UnitServiceTest.eliminarEsBajaLogica`, `UnitRepositoryTest` |
| 55 | Validaciones de la unidad (`capacidad`, `tarifaNoche`, `cuposTotales` > 0; `cuposDisponibles` entre 0 y el total) con errores JSON 400/404/409/503 | Implementada | catalog | `dto/UnitRequest`, `exception/ApiExceptionHandler`; `UnitRequestValidationTest`, `UnitControllerSecurityTest.bodyInvalidoResponde400ConDetalle` |
| 56 | Cupos: `POST /{id}/reserve` (409 sin cupos o inactiva) y `/{id}/release` (sin superar el total), con `@Version` | Implementada | catalog | `entity/Unit`; `UnitTest`, `UnitServiceTest.reservarSinCuposLanza409YNoGuarda`, `UnitServiceTest.liberarNoSuperaElTotal` |
| 57 | Roles del catalogo: GET los 4 roles; POST/PUT/DELETE solo Admin; reserve/release Admin y Operador | Implementada | catalog | commit `51d1ac6`; `UnitControllerSecurityTest` (41 casos) |
| 58 | Carga de 6 unidades de ejemplo solo si la tabla esta vacia | Implementada | catalog | commit `949d297`; `MsAndesstayCatalogApplicationTests.contextLoadsYCargaLasUnidadesDeEjemploUnaSolaVez` |
| 59 | Auditoria: `POST /api/audit/events` (201; actor = usuario del token), `GET /api/audit/events` con filtros y paginacion (`occurredAt` desc), `GET /api/audit/reservations/{id}/timeline` cronologico; sin PUT ni DELETE | Implementada | audit | commit `b8abc65`; `AuditServiceTest` (6 casos), `AuditControllerSecurityTest` (21 casos, GET solo Admin/Auditor) |
| 60 | Reporteria sin BD: `GET /api/report/kpis?range=last24h\|last7d\|last30d` (por hora, por estado, ocupacion activa, tiempo de ciclo, ingresos confirmados) y `GET /api/report/top-units`; solo Admin | Implementada | report | commits `3ba812c`, `1281479`; `KpiCalculatorTest`, `ReportControllerSecurityTest` (range invalido -> 400) |
| 61 | Tolerancia a fallas en report: reservations caido -> 503; audit caido -> `tiempoCicloPromedioHoras` null con aviso | Implementada | report | `ReportServiceTest.reservationsCaidoLanza503`, `ReportServiceTest.auditCaidoDejaSoloElTiempoDeCicloEnNullConAviso` |
| 62 | Reservations -> catalog: `unidadId` opcional; confirmar descuenta cupo (409 sin cupos, 503 catalog caido), cancelar/checkout lo devuelve; detras de la interfaz `CatalogClient` | Implementada | reservations | commits `1f271a8`, `c8b220d`, `3daa34b`; `ReservationServiceTest.confirmarSinCuposLanza409YNoGuardaNiPublica`, `RestClientsTest.catalogSinCuposSeTraduceA409ConSuMensaje` |
| 63 | Reservations -> audit: `RESERVA_CREADA`, `CAMBIO_ESTADO` (anterior, nuevo, actor) y `RESERVA_ELIMINADA`; si audit cae, `log.warn` y sigue; detras de la interfaz `AuditClient` | Implementada | reservations | commit `3daa34b`; `ReservationServiceTest.transicionValidaGuardaPublicaYAudita`, `RestClientsTest.auditCaidoNoRompeNada` |
| 65 | BFF: ruteo `/api/catalog/**`, `/api/audit/**` (solo GET) y `/api/report/**` con los roles de cada microservicio; todas las URLs internas en `andesstay.services.*.url` | Implementada | bff | commits `809bf3a`, `18bb96f`, `b4ca64a`; `CatalogAuditReportBffSecurityTest` (61 casos), `ServiceUrlsTest` |
| 66 | Schemas Oracle propios para catalog (`ANDESSTAY_CATALOG`) y audit (`ANDESSTAY_AUDIT`) en la misma ADB, secretos por variables de entorno | Implementada | catalog, audit | `application.yaml` y `.env.example` de cada uno; README "Base de datos" |

## Resumen

| Estado | Cantidad | % |
|---|---|---|
| Implementada | 65 | 98,5 % |
| Parcial | 0 | 0 % |
| No implementada | 1 | 1,5 % |
| **Total** | **66** | |

## Etapa siguiente

| Funcionalidad | Estado |
|---|---|
| Streaming de eventos con Kafka | No implementada (etapa siguiente) |
