# PLAN: Login con JWT y autenticación de dos factores (2FA)

**SPEC de referencia:** `docs/features/jwt-login-2fa/SPEC.md`
**Versión de la spec revisada:** Aprobada — 2026-09-15
**Contrato de API afectado:** Nuevos endpoints (no existe contrato previo)
**Estado:** Aprobado <!-- Borrador | En revisión | Aprobado -->

<!-- PARA LA PERSONA
Copia esta plantilla como PLAN.md junto a la SPEC.md aprobada.
Este documento define la solución técnica. Una vez revisado, el agente puede
derivar TASKS.md con tareas, dependencias y comprobaciones.
-->

<!-- PARA EL AGENTE
- Lee la SPEC.md aprobada y las instrucciones del proyecto.
  Si falta un documento necesario o la spec no está aprobada, indícalo antes de avanzar.
- Inspecciona el repositorio. Referencia rutas verificadas y distingue las nuevas propuestas.
- Propón una solución proporcional al alcance y coherente con el proyecto.
  Reutiliza lo existente y justifica nuevas dependencias o cambios de arquitectura.
- Distingue hechos, decisiones confirmadas y propuestas. Consulta las decisiones
  no resueltas; haz pocas preguntas por vez y actualiza el plan con las respuestas.
- Referencia los requisitos y criterios por su ID, sin copiar toda la spec.
- Si una decisión cambia el comportamiento o alcance, vuelve a la spec y solicita
  confirmación. No resuelvas una duda de producto mediante una suposición técnica.
- Conserva estos comentarios. No implementes durante la planificación.
- Solicita aprobación antes de marcar el plan como Aprobado. La autorización
  para implementar debe ser explícita; no se deduce del estado de los documentos.
-->

## Contexto técnico verificado

<!-- Qué existe hoy y cómo participa en la funcionalidad. -->

| Componente o archivo existente | Ruta verificada | Responsabilidad y uso previsto |
| --- | --- | --- |
| `AuthController` | `controller/AuthController.java` | Controller existente en `/auth`; se añadirán los nuevos endpoints aquí |
| `VerifyEmailUseCase` | `service/VerifyEmailUseCase.java` | Referencia directa de implementación: misma lógica OTP (5 intentos, 15 min, `noRollbackFor`) |
| `RegisterUseCase` | `service/RegisterUseCase.java` | Referencia del patrón outbox; `buildPayload` para serializar eventos al outbox |
| `OtpRepository` | `repository/OtpRepository.java` | Acceso a OTPs por `userId` y `purpose`; se reutiliza tal cual |
| `OutboxRepository` | `repository/OutboxRepository.java` | Persiste eventos para el relé; se reutiliza tal cual |
| `UserRepository` | `repository/UserRepository.java` | `findByEmail` y `existsByEmail` disponibles; se reutiliza |
| `Otp` / `OtpPurpose.LOGIN_2FA` / `OtpStatus` | `domain/entity/` | Entidad y enum ya contemplan el propósito `LOGIN_2FA` |
| Tabla `session` (V3) | `db/migration/V3__create_session_table.sql` | Columnas `refresh_token_hash`, `revoked`, `revoked_at`, `expires_at`, `ip_address`, `device_info`, `last_used_at` ya definidas |
| `GlobalExceptionHandler` | `config/GlobalExceptionHandler.java` | Se añadirán handlers para las nuevas excepciones de dominio |
| `SecurityConfig` | `config/SecurityConfig.java` | Se modificará para añadir el filtro JWT y proteger los endpoints |
| `ApiSuccessResponse<T>` / `ApiErrorResponse` | `dto/` | Formato de respuesta estándar; se reutilizan sin cambios |
| `OtpCodeGenerator` | `utils/OtpCodeGenerator.java` | Generador de 6 dígitos numéricos; se reutiliza |

**Convenciones y patrón de referencia:** `RegisterUseCase` + `VerifyEmailUseCase` son la referencia de implementación para todos los casos de uso nuevos. Uso de `@Transactional(noRollbackFor = ...)` para persistir intentos fallidos sin revertir. Outbox como único canal de publicación de eventos de dominio.

## Solución propuesta

El flujo completo de login se implementa en dos pasos consecutivos, siguiendo el patrón use-case del proyecto:

1. **`LoginUseCase`** (`POST /auth/login`): valida credenciales (RF-01), invalida cualquier OTP `LOGIN_2FA` pendiente, genera uno nuevo y lo publica en el outbox (RF-02). Devuelve `200` sin token.

2. **`VerifyLoginOtpUseCase`** (`POST /auth/login/verify-2fa`): verifica el OTP siguiendo exactamente la lógica de `VerifyEmailUseCase` (RF-03). Si es correcto, genera un access token JWT (RS256) y un refresh token opaco, crea un registro en `session` y devuelve ambos tokens con sus tiempos de expiración (RF-04).

3. **`RefreshTokenUseCase`** (`POST /auth/refresh`): busca la sesión por el hash del refresh token recibido, verifica que no esté revocada ni expirada, genera un nuevo par de tokens con rotación (el refresh anterior queda revocado) y persiste la nueva sesión (RF-05).

4. **`LogoutUseCase`** (`POST /auth/logout`): requiere access token válido (verificado por el filtro JWT antes de llegar al use case) y refresh token en el cuerpo. Marca la sesión correspondiente como revocada. Idempotente: si ya estaba revocada responde `200` (RF-06).

5. **Filtro JWT** (`JwtAuthenticationFilter`): intercepta todas las peticiones protegidas, extrae y valida el access token usando la clave pública RS256, y popula el `SecurityContext`. Las rutas `POST /user/**`, `POST /auth/login`, `POST /auth/login/verify-2fa` y `POST /auth/refresh` quedan fuera del filtro (RF-07).

6. **Rate limiting** (`POST /auth/login`): se implementa mediante el filtro `LoginRateLimitFilter`, que mantiene un contador por IP en memoria usando `ConcurrentHashMap` con ventana deslizante de 1 minuto. Umbral: **10 peticiones/min por IP**. Responde `429` al superarlo, sin revelar estado de cuenta (RF-08).

**Recorrido de datos — login completo:**
`POST /auth/login` → `LoginUseCase` → `UserRepository.findByEmail` → `BCryptPasswordEncoder.matches` → `OtpRepository` (invalidar pendiente) → `OtpCodeGenerator` → `OtpRepository.save` → `OutboxRepository.save` → `200 OK`

`POST /auth/login/verify-2fa` → `VerifyLoginOtpUseCase` → `OtpRepository` → validación → `JwtTokenProvider.generateAccessToken` → generar refresh token opaco → `TokenHasher.hash(refreshToken)` → `SessionRepository.save` → `200 OK + tokens`

**Generación de tokens:**
- **Access token:** `io.jsonwebtoken:jjwt-*` (JJWT 0.12.x). Se firma con clave privada RS256 cargada desde PEM en propiedades de entorno. Claims: `sub` (user UUID string), `exp` (ahora + 1 hora).
- **Refresh token:** `UUID.randomUUID().toString()` (token opaco). Se almacena como `TokenHasher.hash(refreshToken)` (SHA-256 en hex) en `session.refresh_token_hash`. SHA-256 es determinista, lo que permite una consulta indexada exacta en lugar de un escaneo por tabla.

## Contrato de API

<!-- Define o referencia el contrato expuesto o modificado. Si ya existe una
definición formal (OpenAPI/JSON Schema/protobuf), referencia la ruta en lugar
de copiarla y describe solo el diff. -->

| Método | Ruta | Autenticación / autorización | Códigos de estado esperados | Requisito relacionado |
| --- | --- | --- | --- | --- |
| POST | `/auth/login` | Público | 200, 400, 401, 429 | RF-01, RF-02, RF-08 |
| POST | `/auth/login/verify-2fa` | Público | 200, 400, 429 | RF-03, RF-04 |
| POST | `/auth/refresh` | Público (solo refresh token) | 200, 400, 401 | RF-05 |
| POST | `/auth/logout` | `Bearer` access token + refresh token en body | 200, 401 | RF-06 |

**Esquema de request — `POST /auth/login`:**
```json
{ "email": "string (requerido, formato email)", "password": "string (requerido)" }
```

**Esquema de response — `POST /auth/login` (200):**
```json
{ "status": "success", "message": "Se ha enviado un código de verificación a tu correo." }
```

**Esquema de request — `POST /auth/login/verify-2fa`:**
```json
{ "email": "string (requerido)", "otp_code": "string (requerido, 6 dígitos)" }
```

**Esquema de response — `POST /auth/login/verify-2fa` (200):**
```json
{
  "status": "success",
  "message": "Autenticación completada.",
  "data": {
    "access_token": "string (JWT)",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "string (opaco)",
    "refresh_token_expires_in": 604800
  }
}
```

**Esquema de request — `POST /auth/refresh`:**
```json
{ "refresh_token": "string (requerido)" }
```

**Esquema de response — `POST /auth/refresh` (200):** misma estructura que `verify-2fa`.

**Esquema de request — `POST /auth/logout`:**
Header: `Authorization: Bearer <access_token>`
Body: `{ "refresh_token": "string (requerido)" }`

**Compatibilidad hacia atrás:** todos los endpoints son nuevos. Los endpoints existentes (`POST /user/`, `POST /auth/verify-email`) no se modifican. El cambio en `SecurityConfig` para añadir el filtro JWT no afecta a los endpoints abiertos ya definidos.

## Módulos y componentes afectados

<!-- Si el proyecto está modularizado, identifica los módulos afectados, sus
responsabilidades y la dirección de sus dependencias. Respeta los límites
existentes y justifica cualquier módulo o dependencia nueva. Si no está
modularizado, describe las carpetas o componentes afectados sin introducir
modularización fuera del alcance; marca la tabla de módulos como No aplica. -->

No aplica — proyecto de módulo único sin modularización explícita.

| Componente o ruta | Capa | Acción | Cambio y responsabilidad | Requisito relacionado |
| --- | --- | --- | --- | --- |
| `controller/AuthController.java` | Controller | Modificar | Añadir los 4 nuevos endpoints; delega en los use cases correspondientes | RF-01 a RF-06 |
| `service/LoginUseCase.java` | Servicio | Crear | Valida credenciales, invalida OTP pendiente, genera OTP `LOGIN_2FA`, publica en outbox | RF-01, RF-02 |
| `service/VerifyLoginOtpUseCase.java` | Servicio | Crear | Verifica OTP (igual que `VerifyEmailUseCase`), emite access y refresh token, crea sesión | RF-03, RF-04 |
| `service/RefreshTokenUseCase.java` | Servicio | Crear | Valida refresh token, rota sesión, emite nuevo par de tokens | RF-05 |
| `service/LogoutUseCase.java` | Servicio | Crear | Revoca la sesión por hash del refresh token; idempotente | RF-06 |
| `config/JwtTokenProvider.java` | Config | Crear | Genera y valida JWT RS256; carga clave privada/pública desde configuración | RF-04, RF-07 |
| `config/JwtAuthenticationFilter.java` | Config | Crear | Filtro de Spring Security; valida Bearer token y puebla `SecurityContext` | RF-07 |
| `config/LoginRateLimitFilter.java` | Config | Crear | Filtro previo al dispatcher; limita peticiones por IP a 10/min con ventana deslizante en memoria | RF-08 |
| `config/SecurityConfig.java` | Config | Modificar | Registrar filtros JWT y rate limit; mantener rutas públicas existentes | RF-07, RF-08 |
| `config/GlobalExceptionHandler.java` | Config | Modificar | Añadir handlers para `InvalidCredentialsException`, `SessionNotFoundException`, `TokenExpiredException` | RF-01, RF-05 |
| `domain/entity/Session.java` | Dominio | Crear | Entidad JPA que mapea la tabla `session` existente | RF-04, RF-05, RF-06 |
| `repository/SessionRepository.java` | Repositorio | Crear | `findByRefreshTokenHash`, `findByRefreshTokenHashAndRevokedFalse` | RF-05, RF-06 |
| `dto/LoginCommand.java` | DTO | Crear | Request de `POST /auth/login` con validaciones Bean Validation | RF-01 |
| `dto/VerifyLoginOtpCommand.java` | DTO | Crear | Request de `POST /auth/login/verify-2fa` con validaciones | RF-03 |
| `dto/RefreshTokenCommand.java` | DTO | Crear | Request de `POST /auth/refresh` | RF-05 |
| `dto/LogoutCommand.java` | DTO | Crear | Request de `POST /auth/logout` | RF-06 |
| `dto/AuthTokenResponseData.java` | DTO | Crear | Response con `access_token`, `refresh_token`, `expires_in`, `refresh_token_expires_in` | RF-04, RF-05 |
| `utils/exceptions/InvalidCredentialsException.java` | Utils | Crear | Excepción genérica para credenciales inválidas → 401 | RF-01 |
| `utils/exceptions/SessionNotFoundException.java` | Utils | Crear | Excepción para refresh token no encontrado/revocado → 401 | RF-05, RF-06 |

## Datos y contratos

<!-- Completa solo lo aplicable. Si un punto no aplica, indica el motivo. -->

- **Modelos de dominio y su relación con el contrato de entrada y salida:**
  - `User` (existente): se lee para validar email, `verifiedAt` y `passwordHash`. No se modifica en el login.
  - `Otp` (existente): se crea uno nuevo con `purpose = LOGIN_2FA` en el login; se verifica y marca `VERIFIED` en el 2FA. El anterior pendiente se marca `EXPIRED` en el paso 1.
  - `Session` (nueva entidad JPA sobre tabla existente): se crea en el verify-2fa, se actualiza en refresh (rotación) y se marca revocada en logout. Contiene `refreshTokenHash` (BCrypt), `userId`, `ipAddress`, `deviceInfo`, `expiresAt`, `revoked`, `revokedAt`, `lastUsedAt`.
  - `Outbox` (existente): se crea un evento `USER_LOGIN_OTP` en el paso 1 con payload `{id, email, otp_code}`.

- **Identificadores, relaciones y restricciones:**
  - La sesión se identifica por `refreshTokenHash` (SHA-256 del token recibido).
  - La comparación del refresh token recibido contra el hash almacenado se hace calculando `TokenHasher.hash(token)` y buscando con `findByRefreshTokenHash(hash)` — consulta indexada O(log n).
  - Un refresh token revocado (`revoked = true`) nunca puede volver a usarse.

- **Origen de los datos y transformaciones (entrada → dominio → persistencia/salida):**
  - Login: `LoginCommand` → `User` (BD) → `Otp` creado (BD) → `Outbox` creado (BD) → respuesta sin token.
  - Verify 2FA: `VerifyLoginOtpCommand` → `Otp` validado (BD) → `Session` creada (BD) → `access_token` (generado en memoria) + `refresh_token` opaco → `AuthTokenResponseData`.
  - Refresh: `RefreshTokenCommand` → `Session` buscada por hash → sesión anterior revocada + nueva sesión creada (BD) → nuevo par de tokens.
  - Logout: `LogoutCommand` + access token → `Session` marcada revocada (BD) → 200.

- **Persistencia, consultas y actualizaciones:**
  - `SessionRepository.findByRefreshTokenHash(String hash)`: localiza sesión para refresh y logout. Se beneficia del índice `idx_session_refresh_token_hash` ya creado.
  - `OtpRepository.findTopByUserIdAndPurposeOrderByCreatedAtDesc`: ya existe, se reutiliza para `LOGIN_2FA`.
  - `Session.lastUsedAt` se actualiza en cada operación de refresh.

- **Consistencia entre servicios o fuentes de datos:** El envío del OTP al correo se delega al outbox; si falla el `OutboxPublisher`, el usuario no recibe el código pero la base de datos permanece consistente. El reintento puede hacerse volviendo a llamar a `POST /auth/login`.

- **Compatibilidad y migraciones de datos existentes:** La tabla `session` ya existe (V3). No se requiere ninguna migración Flyway para esta funcionalidad. Las entidades `Otp` y `User` no se modifican.

## Seguridad y validación de entrada

<!-- Qué se valida antes de procesar la petición y qué datos requieren
tratamiento especial. -->

- **Validación y sanitización de inputs:**
  - `LoginCommand`: `@NotBlank` + `@Email` en `email`; `@NotBlank` en `password`. Sin validación de complejidad (la fortaleza se validó en el registro).
  - `VerifyLoginOtpCommand`: `@NotBlank` + `@Email` en `email`; `@NotBlank` + `@Pattern("[0-9]{6}")` en `otp_code`.
  - `RefreshTokenCommand` y `LogoutCommand`: `@NotBlank` en `refresh_token`.
  - El error `401` genérico del login se construye en `LoginUseCase` sin información de qué condición falló (RF-01).

- **Autenticación y autorización requeridas:**
  - `POST /auth/login`, `POST /auth/login/verify-2fa`, `POST /auth/refresh`: públicos; sin token requerido.
  - `POST /auth/logout`: requiere access token válido en `Authorization: Bearer` (validado por `JwtAuthenticationFilter`) más refresh token en el cuerpo (DP-10).
  - Cualquier endpoint futuro no incluido en la lista pública de `SecurityConfig` requerirá access token válido.

- **Datos sensibles (PII, secretos) y su tratamiento en almacenamiento y logs:**
  - El refresh token **nunca** se almacena en claro: solo su hash SHA-256 (hex).
  - La clave privada RS256 se carga desde una variable de entorno o archivo externo, nunca desde el código fuente.
  - Los logs no registran `password`, `otp_code`, tokens ni hashes.
  - El payload del outbox incluye `otp_code` (necesario para el servicio de email); se trata como dato sensible — el publisher no debe logarlo.

- **Rate limiting / throttling:** `LoginRateLimitFilter` limita `POST /auth/login` a **10 peticiones/minuto por IP** usando ventana deslizante en `ConcurrentHashMap`. Responde `429 Too Many Requests` con el formato estándar `ApiErrorResponse`. El acceso al mapa es `synchronized` por IP para evitar condiciones de carrera en entornos multi-hilo. Nota: esta implementación en memoria no es válida en despliegue multi-instancia; escalar a Redis está fuera del alcance de esta entrega y se documenta como deuda técnica.

## Rendimiento y escalabilidad

<!-- Completa solo lo aplicable. -->

- **Impacto en consultas a base de datos:**
  - `idx_session_refresh_token_hash` ya existe (V3); la búsqueda de sesión por hash SHA-256 es O(log n) con una consulta exacta.
  - `idx_session_user_id` ya existe (V3); sin riesgo de N+1 en las operaciones de sesión.
  - `idx_otp_user_id` ya existe (V2); la consulta de OTP pendiente está indexada.
  - El cálculo SHA-256 del refresh token se hace en memoria y tiene coste constante y despreciable.

- **Paginación:** No aplica — ningún endpoint de esta entrega devuelve colecciones.

- **Estrategia de caché:** No aplica en esta entrega. La validación del access token es stateless (solo verifica la firma RS256); no requiere consulta a base de datos por petición.

## Estado, operaciones y errores

<!-- Cómo se implementan los comportamientos aprobados en la spec. -->

- **Transaccionalidad:**
  - `LoginUseCase.execute`: `@Transactional`. Incluye invalidar OTP anterior, crear nuevo OTP y crear evento outbox. Si cualquier operación falla, se revierte todo (el usuario puede reintentar).
  - `VerifyLoginOtpUseCase.execute`: `@Transactional(noRollbackFor = {InvalidOtpException.class, TooManyOtpAttemptsException.class})`, igual que `VerifyEmailUseCase`. Los intentos fallidos se persisten aunque la transacción no sea exitosa.
  - `RefreshTokenUseCase.execute`: `@Transactional`. Revoca la sesión anterior y crea la nueva en la misma transacción; si falla la creación, la revocación se revierte.
  - `LogoutUseCase.execute`: `@Transactional`. Marca `revoked = true`; idempotente (no lanza excepción si ya estaba revocada).

- **Idempotencia:**
  - `POST /auth/logout`: idempotente. Si el refresh token ya estaba revocado o no existe, responde `200 OK` (DP-08).
  - `POST /auth/refresh`: no idempotente por diseño (cada llamada genera un nuevo par de tokens). El cliente no debe reintentar sin el token recibido en la respuesta anterior.

- **Concurrencia de acceso a datos:**
  - Dos llamadas simultáneas a `POST /auth/refresh` con el mismo refresh token: la segunda encontrará la sesión ya revocada (por la primera) y responderá `401`. El índice único no está en el refresh token hash, pero la lógica de "no reutilizar sesión revocada" evita doble emisión.
  - Dos llamadas simultáneas a `POST /auth/login` para el mismo usuario: ambas crean un OTP válido; el más reciente prevalece en `findTopByUserIdAndPurposeOrderByCreatedAtDesc`. El usuario recibirá dos correos; solo el último código será funcional.

- **Errores, reintentos y prevención de duplicados:**

  | Excepción de dominio | Código HTTP | Mensaje al cliente |
  | --- | --- | --- |
  | `InvalidCredentialsException` | 401 | "Credenciales inválidas." |
  | `InvalidOtpException` (existente) | 400 | "El código ingresado es inválido o ha expirado." |
  | `TooManyOtpAttemptsException` (existente) | 429 | "Demasiados intentos. Intenta de nuevo en unos minutos." |
  | `SessionNotFoundException` | 401 | "Sesión inválida o expirada." |
  | `MethodArgumentNotValidException` (existente) | 400 | Lista de errores de campo |

- **Otras consideraciones aplicables:** La clave pública RS256 puede compartirse con otros microservicios de SplitFlow para que validen el access token sin llamar al auth service (validación stateless).

## Dependencias y configuración

<!-- Librerías, servicios, permisos o configuración afectados. Verifica compatibilidad
con el proyecto y justifica las incorporaciones. No agregues dependencias por defecto. -->

- **Librerías nuevas:**
  - `io.jsonwebtoken:jjwt-api`, `jjwt-impl`, `jjwt-jackson` — versión `0.12.6` (la más reciente estable con soporte RS256 nativo). Justificación: librería estándar de facto en el ecosistema Spring Boot para JWT; compatible con Java 25. Se añaden al `pom.xml` con scope `compile` (`jjwt-impl` con scope `runtime`).
  - `com.github.bucket4j:bucket4j-core` — **No se incorpora** en esta entrega. El rate limiting se implementa con `ConcurrentHashMap` en memoria (suficiente para un único nodo). Esto se documenta como deuda técnica para cuando se escale horizontalmente.

- **Variables de entorno nuevas:**
  - `JWT_PRIVATE_KEY_PEM` — clave privada RS256 en formato PEM (sin cabeceras), usada para firmar tokens.
  - `JWT_PUBLIC_KEY_PEM` — clave pública RS256 en formato PEM, usada para verificar firmas.
  - Se añaden con `@Value("${jwt.private-key}")` / `@Value("${jwt.public-key}")` en `JwtTokenProvider`. Las claves se definen en `application.properties` como referencias a variables de entorno: `jwt.private-key=${JWT_PRIVATE_KEY_PEM}` y `jwt.public-key=${JWT_PUBLIC_KEY_PEM}`.

- **Migraciones de base de datos:** Ninguna. La tabla `session` (V3) ya existe y contiene todas las columnas necesarias. Solo se crea la entidad JPA `Session` que la mapea.

- **Feature flags:** No aplica.

## Estrategia de validación

<!-- Una fila por criterio de la spec. Selecciona el método capaz de demostrarlo:
test unitario, integración o prueba manual. No todos requieren todos los métodos.
Identifica tests existentes y separa los nuevos propuestos. Incluye regresiones relevantes.
Un test unitario aislado no demuestra el comportamiento end-to-end del endpoint;
una respuesta 200 no demuestra los efectos secundarios esperados (persistencia,
eventos, notificaciones). -->

| Criterio | Método y test propuesto | Entorno, datos y estado inicial | Evidencia prevista |
| --- | --- | --- | --- |
| CA-01 | Test de integración `AuthControllerLoginTest#login_success` | Testcontainers (Postgres + RabbitMQ); usuario verificado en BD | 200, body sin `access_token`, OTP en BD |
| CA-02 | Test de integración `AuthControllerLoginTest#login_wrongPassword` y `login_unknownEmail` | Usuario existente con password incorrecta; email inexistente | 401 con mismo mensaje en ambos casos |
| CA-03 | Test de integración `AuthControllerLoginTest#login_unverifiedUser` | Usuario sin `verified_at` en BD | 401, mismo mensaje que CA-02 |
| CA-04 | Test de integración CA-01 + inspección de BD | BD tras login exitoso | `otp.purpose=LOGIN_2FA`, `otp.status=PENDING`, `otp.expires_at` en ≤15 min |
| CA-05 | Test de integración CA-01 + inspección de BD | BD tras login exitoso | Registro en `outbox` con `event_type=USER_LOGIN_OTP` y `otp_code` en payload |
| CA-06 | Test de integración `AuthControllerLogin2faTest#verify2fa_success` | OTP válido en BD | 200, `access_token` y `refresh_token` presentes en response |
| CA-07 | Test de integración `AuthControllerLogin2faTest#verify2fa_wrongCode` | OTP con 0 intentos | 400, `otp.attempts=1` en BD |
| CA-08 | Test de integración `AuthControllerLogin2faTest#verify2fa_tooManyAttempts` | OTP con `attempts=4` | 429, `otp.status=EXPIRED` |
| CA-09 | Test de integración CA-06 + decodificación del JWT | JWT emitido en CA-06 | Claims `sub` (UUID del usuario) y `exp` (ahora + 3600s) presentes |
| CA-10 | Test de integración `JwtProtectedEndpointTest#accessWithValidToken` (requiere endpoint protegido de prueba) | Access token válido | 200 |
| CA-11 | Test de integración `JwtProtectedEndpointTest#accessWithExpiredToken` y `accessWithMangledToken` | Token expirado / manipulado | 401 |
| CA-12 | Test de integración `RefreshTokenControllerTest#refresh_success` | Sesión válida en BD | 200, nuevo `access_token` y nuevo `refresh_token`; sesión anterior revocada |
| CA-13 | Test de integración `RefreshTokenControllerTest#refresh_revokedToken` | Sesión con `revoked=true` | 401 |
| CA-14 | Test de integración `LogoutControllerTest#logout_success` | Sesión activa en BD | 200, `session.revoked=true` en BD |
| CA-15 | Test de integración `JwtProtectedEndpointTest#accessWithoutToken` | Sin header `Authorization` | 401 |
| CA-16 | Test de integración `LoginRateLimitFilterTest#rateLimit_exceeded` | 11 peticiones consecutivas al mismo IP | Primera en 429 después del umbral; respuesta sin info de cuenta |

**Comprobaciones de regresión:** Ejecutar `AuthControllerTest` (verify-email) y `UserControllerTest` (registro) para confirmar que los cambios en `SecurityConfig` y `GlobalExceptionHandler` no rompen los endpoints existentes.

**Comandos verificados para compilar y ejecutar tests:**
```bash
# Compilar
./mvnw clean package -DskipTests

# Todos los tests (requiere Docker para Testcontainers)
./mvnw test

# Solo el nuevo conjunto de tests de login
./mvnw test -Dtest="AuthControllerLoginTest,AuthControllerLogin2faTest,RefreshTokenControllerTest,LogoutControllerTest,JwtProtectedEndpointTest,LoginRateLimitFilterTest"

# Regresión endpoints existentes
./mvnw test -Dtest="AuthControllerTest,UserControllerTest"
```

**Limitaciones del entorno:** CA-16 (rate limiting por IP) puede dar resultados no deterministas si el entorno de test recicla conexiones o comparte IP. Se verificará manualmente con `curl` en un entorno local levantado con `./mvnw spring-boot:run`.

## Orden de implementación

<!-- Etapas y dependencias principales. El desglose ejecutable se escribe en TASKS.md.
Incluye puntos de comprobación para avanzar con cambios pequeños. -->

1. **Infraestructura JWT** — Añadir dependencias JJWT al `pom.xml`, crear `JwtTokenProvider` (generación y validación RS256), crear `JwtAuthenticationFilter` y registrarlo en `SecurityConfig`. *Punto de comprobación:* CA-10, CA-11, CA-15 con un endpoint de prueba `GET /auth/ping` protegido.

2. **Entidad y repositorio de sesión** — Crear `Session` (JPA sobre tabla V3) y `SessionRepository`. *Punto de comprobación:* test unitario de repositorio; confirmar que `ddl-auto=validate` no lanza error.

3. **Login paso 1** — Crear `LoginCommand`, `LoginUseCase`, excepciones nuevas, añadir `POST /auth/login` a `AuthController` y actualizar `GlobalExceptionHandler`. *Punto de comprobación:* CA-01 a CA-05.

4. **Login paso 2 (2FA + emisión de tokens)** — Crear `VerifyLoginOtpCommand`, `AuthTokenResponseData`, `VerifyLoginOtpUseCase`, añadir `POST /auth/login/verify-2fa`. *Punto de comprobación:* CA-06 a CA-09.

5. **Renovación de token (refresh)** — Crear `RefreshTokenCommand`, `RefreshTokenUseCase`, añadir `POST /auth/refresh`. *Punto de comprobación:* CA-12, CA-13.

6. **Logout** — Crear `LogoutCommand`, `LogoutUseCase`, añadir `POST /auth/logout`. *Punto de comprobación:* CA-14.

7. **Rate limiting** — Crear `LoginRateLimitFilter`, registrarlo en `SecurityConfig`. *Punto de comprobación:* CA-16.

8. **Regresión y validación final** — Ejecutar suite completa de tests, verificar CA-01 a CA-16, documentar evidencias en `TASKS.md`.

## Riesgos y decisiones pendientes

<!-- Riesgos concretos de esta solución y cómo se resolverán, sin listas genéricas.
Considera especialmente: breaking changes de contrato, migraciones irreversibles
e impacto en consumidores existentes de la API. Escribe Ninguna en las decisiones
pendientes cuando estén resueltas. -->

- **Riesgos y medidas acordadas:**
  - *Rate limiting en memoria:* `LoginRateLimitFilter` con `ConcurrentHashMap` no es efectivo en despliegues multi-nodo. Aceptado para esta entrega; se documenta como deuda técnica. La solución escalable (Redis + Bucket4j) se implementará cuando el servicio escale horizontalmente.
  - *Rotación de refresh tokens y reintentos del cliente:* si el cliente pierde la respuesta de `POST /auth/refresh` (p. ej., corte de red), el refresh token anterior estará ya revocado y no podrá volver a usarlo. El cliente deberá iniciar sesión de nuevo. Este comportamiento es aceptado (trade-off de seguridad).
  - *Claves RS256 en variables de entorno:* si las variables no están configuradas al arrancar, la aplicación lanzará una excepción en el contexto de Spring. Se debe documentar en el README el procedimiento de generación de claves y configuración.
  - *Cambio en `SecurityConfig`:* cualquier ruta no listada explícitamente como pública quedará protegida por el filtro JWT. Verificar con tests de regresión que los endpoints existentes siguen siendo accesibles.

- **Decisiones pendientes:** Ninguna.

<!-- ANTES DE SOLICITAR APROBACIÓN
Comprueba que el plan cubre los requisitos, respeta las exclusiones, reutiliza
componentes verificados y permite demostrar todos los criterios de aceptación.
Resuelve dudas y marcadores pendientes. Si la spec cambió, revisa su impacto.
Tras aprobar el plan, deriva TASKS.md con IDs, dependencias, referencias a RF/CA
y comprobaciones. No marques una tarea terminada sin realizar su validación;
si está bloqueada, registra el motivo.
-->
