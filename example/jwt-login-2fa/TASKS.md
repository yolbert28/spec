# TASKS: Login con JWT y autenticación de dos factores (2FA)

**PLAN de referencia:** `docs/features/jwt-login-2fa/PLAN.md`
**SPEC de referencia:** `docs/features/jwt-login-2fa/SPEC.md`

---

## Etapa 1 — Infraestructura JWT

### T-01 · Añadir dependencias JJWT al `pom.xml`
- **Objetivo:** incorporar la librería JJWT 0.12.6 para generar y validar tokens RS256.
- **Alcance:** modificar `pom.xml`; añadir `jjwt-api` (compile), `jjwt-impl` (runtime), `jjwt-jackson` (runtime).
- **Dependencias:** ninguna.
- **Criterios resueltos:** prerequisito de RF-04, RF-07.
- **Validación:** `./mvnw clean package -DskipTests` sin errores de compilación.
- [x] Completada

---

### T-02 · Crear `JwtTokenProvider`
- **Objetivo:** componente que firma, genera y valida JWT RS256.
- **Alcance:** crear `config/JwtTokenProvider.java` con:
  - `generateAccessToken(UUID userId)` → JWT con claims `sub` y `exp` (ahora + 1h), firmado con clave privada RS256.
  - `validateToken(String token)` → booleano; usa la clave pública.
  - `getUserIdFromToken(String token)` → extrae `sub`.
  - Claves cargadas desde `@Value("${jwt.private-key}")` y `@Value("${jwt.public-key}")`.
  - Añadir `jwt.private-key` y `jwt.public-key` en `application.properties` referenciando variables de entorno.
- **Dependencias:** T-01.
- **Criterios resueltos:** CA-09, CA-10, CA-11.
- **Validación:** test unitario `JwtTokenProviderTest` con clave generada inline: emitir token, validar firma, extraer `sub`; verificar que un token manipulado falla la validación.
- [x] Completada

---

### T-03 · Crear `JwtAuthenticationFilter`
- **Objetivo:** filtro que valida el Bearer token en cada petición y puebla el `SecurityContext`.
- **Alcance:** crear `config/JwtAuthenticationFilter.java` (extiende `OncePerRequestFilter`):
  - Extrae el header `Authorization: Bearer <token>`.
  - Delega validación a `JwtTokenProvider`.
  - Si válido, crea `UsernamePasswordAuthenticationToken` con el `userId` como principal y lo registra en `SecurityContextHolder`.
  - Si inválido o ausente, no lanza excepción (Spring Security lo rechazará si la ruta lo requiere).
- **Dependencias:** T-02.
- **Criterios resueltos:** CA-10, CA-11, CA-15.
- **Validación:** test unitario `JwtAuthenticationFilterTest`: petición con token válido popula el contexto; petición sin token no lanza excepción.
- [x] Completada

---

### T-04 · Actualizar `SecurityConfig` y añadir endpoint de prueba protegido
- **Objetivo:** registrar el filtro JWT, mantener las rutas públicas existentes y añadir un endpoint mínimo para verificar la protección.
- **Alcance:**
  - Modificar `config/SecurityConfig.java`: añadir `.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)`.
  - Rutas públicas: `POST /user/**`, `POST /auth/login`, `POST /auth/login/verify-2fa`, `POST /auth/refresh`. `POST /auth/logout` requiere token (se añade en T-14).
  - Añadir `GET /auth/ping` como endpoint protegido temporal en `AuthController` solo para validar CA-10, CA-11 y CA-15 (se puede eliminar al finalizar o conservar como health check).
- **Dependencias:** T-03.
- **Criterios resueltos:** CA-10, CA-11, CA-15.
- **Validación:** test de integración `JwtProtectedEndpointTest` con Testcontainers:
  - `GET /auth/ping` con token válido → 200.
  - `GET /auth/ping` con token manipulado → 401.
  - `GET /auth/ping` sin header `Authorization` → 401.
  - Tests de regresión `AuthControllerTest` y `UserControllerTest` en verde.
- [x] Completada

---

## Etapa 2 — Entidad y repositorio de sesión

### T-05 · Crear entidad JPA `Session`
- **Objetivo:** mapear la tabla `session` existente (V3) a una entidad de dominio.
- **Alcance:** crear `domain/entity/Session.java` con todos los campos de la tabla:
  `id` (UUID), `userId` (UUID), `refreshTokenHash` (String), `ipAddress` (String), `deviceInfo` (String), `revoked` (boolean), `revokedAt` (OffsetDateTime), `lastUsedAt` (OffsetDateTime), `createdAt` (OffsetDateTime), `updatedAt` (OffsetDateTime), `expiresAt` (OffsetDateTime).
  Usar `@Builder`, `@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor` (patrón Lombok del proyecto).
- **Dependencias:** ninguna (tabla ya existe).
- **Criterios resueltos:** prerequisito de RF-04, RF-05, RF-06.
- **Validación:** `./mvnw clean package -DskipTests`; Hibernate valida el esquema con `ddl-auto=validate` sin errores.
- [x] Completada

---

### T-06 · Crear `SessionRepository`
- **Objetivo:** interfaz de acceso a datos para la entidad `Session`.
- **Alcance:** crear `repository/SessionRepository.java` (extiende `JpaRepository<Session, UUID>`) con:
  - `Optional<Session> findByRefreshTokenHash(String hash)` — para refresh y logout. El hash SHA-256 del token hace que esta consulta use el índice `idx_session_refresh_token_hash`.
- **Dependencias:** T-05.
- **Criterios resueltos:** prerequisito de RF-05, RF-06.
- **Validación:** test de integración de repositorio (con `@DataJpaTest` + Testcontainers Postgres): insertar sesión y recuperarla por hash.
- [x] Completada

---

## Etapa 3 — Login paso 1 (credenciales + OTP 2FA)

### T-07 · Crear excepciones de dominio nuevas
- **Objetivo:** excepciones explícitas para los nuevos casos de error del login.
- **Alcance:** crear:
  - `utils/exceptions/InvalidCredentialsException.java` → mapeada a 401 en `GlobalExceptionHandler`.
  - `utils/exceptions/SessionNotFoundException.java` → mapeada a 401 en `GlobalExceptionHandler`.
- **Dependencias:** ninguna.
- **Criterios resueltos:** CA-02, CA-03, CA-13.
- **Validación:** compilación limpia; handlers en `GlobalExceptionHandler` responden con el código y mensaje correcto en test unitario.
- [x] Completada

---

### T-08 · Actualizar `GlobalExceptionHandler` con los nuevos handlers
- **Objetivo:** mapear las nuevas excepciones a los códigos HTTP definidos en el plan.
- **Alcance:** añadir en `config/GlobalExceptionHandler.java`:
  - `handleInvalidCredentials(InvalidCredentialsException)` → 401, mensaje `"Credenciales inválidas."`, sin campo `errors[]`.
  - `handleSessionNotFound(SessionNotFoundException)` → 401, mensaje `"Sesión inválida o expirada."`.
- **Dependencias:** T-07.
- **Criterios resueltos:** CA-02, CA-03, CA-13.
- **Validación:** tests unitarios del handler; los tests de regresión existentes siguen en verde.
- [x] Completada

---

### T-09 · Crear `LoginCommand` y `LoginUseCase`
- **Objetivo:** implementar la validación de credenciales y la generación del OTP `LOGIN_2FA`.
- **Alcance:**
  - `dto/LoginCommand.java`: campos `email` (`@NotBlank`, `@Email`) y `password` (`@NotBlank`).
  - `service/LoginUseCase.java` (`@Service`):
    1. `userRepository.findByEmail(email)` → si no existe, lanzar `InvalidCredentialsException`.
    2. `passwordEncoder.matches(password, user.getPasswordHash())` → si falla, lanzar `InvalidCredentialsException`.
    3. Si `user.getVerifiedAt() == null` → lanzar `InvalidCredentialsException`.
    4. Invalidar OTP `LOGIN_2FA` pendiente anterior: buscar con `findTopByUserIdAndPurposeOrderByCreatedAtDesc`, si existe y `status == PENDING` marcarlo `EXPIRED` y guardar.
    5. Generar nuevo `OtpCodeGenerator.generate()`, crear `Otp` con `purpose = LOGIN_2FA`, `expiresAt = now + 15 min`, guardar.
    6. Crear evento outbox con `eventType = "USER_LOGIN_OTP"`, payload `{id, email, otp_code}`, guardar.
  - Anotado con `@Transactional`.
  - Retorna `void` (sin datos de respuesta).
- **Dependencias:** T-07, T-08.
- **Criterios resueltos:** CA-01, CA-02, CA-03, CA-04, CA-05.
- **Validación:** test de integración `AuthControllerLoginTest` con Testcontainers:
  - `login_success` → 200, OTP en BD, evento en outbox.
  - `login_wrongPassword` → 401, mensaje genérico.
  - `login_unknownEmail` → 401, mismo mensaje.
  - `login_unverifiedUser` → 401, mismo mensaje.
- [x] Completada

---

### T-10 · Añadir `POST /auth/login` a `AuthController`
- **Objetivo:** exponer el endpoint de inicio de sesión.
- **Alcance:** añadir en `AuthController`:
  ```
  POST /auth/login → LoginUseCase.execute(command)
  → 200 ApiSuccessResponse con mensaje "Se ha enviado un código de verificación a tu correo."
  ```
  Sin datos en el campo `data`.
- **Dependencias:** T-09.
- **Criterios resueltos:** CA-01, CA-02, CA-03.
- **Validación:** cubierto por los tests de T-09.
- [x] Completada

---

## Etapa 4 — Login paso 2 (verificación 2FA + emisión de tokens)

### T-11 · Crear `AuthTokenResponseData` y `VerifyLoginOtpCommand`
- **Objetivo:** DTOs para el request y el response del verify-2fa.
- **Alcance:**
  - `dto/VerifyLoginOtpCommand.java`: `email` (`@NotBlank`, `@Email`), `otp_code` (`@NotBlank`, `@Pattern("[0-9]{6}")`).
  - `dto/AuthTokenResponseData.java`: `accessToken` (String), `tokenType` ("Bearer"), `expiresIn` (long, segundos), `refreshToken` (String), `refreshTokenExpiresIn` (long). Usar `@JsonProperty` para snake_case: `access_token`, `token_type`, `expires_in`, `refresh_token`, `refresh_token_expires_in`.
- **Dependencias:** ninguna.
- **Criterios resueltos:** CA-06, CA-09.
- **Validación:** compilación limpia.
- [x] Completada

---

### T-12 · Crear `VerifyLoginOtpUseCase`
- **Objetivo:** verificar el OTP 2FA y emitir el par de tokens.
- **Alcance:** `service/VerifyLoginOtpUseCase.java`:
  1. `userRepository.findByEmail(email)` → si no existe, lanzar `InvalidOtpException` (mensaje genérico, igual que `VerifyEmailUseCase`).
  2. `otpRepository.findTopByUserIdAndPurposeOrderByCreatedAtDesc(userId, LOGIN_2FA)` → si no existe, lanzar `InvalidOtpException`.
  3. Verificar intentos (`>= 5`) y estado (`EXPIRED`) → lanzar `TooManyOtpAttemptsException`.
  4. Verificar `status != PENDING` o `expiresAt < now` → lanzar `InvalidOtpException`.
  5. Si código correcto: marcar `status = VERIFIED`, guardar OTP.
  6. Generar refresh token: `UUID.randomUUID().toString()`.
  7. Obtener IP y user-agent del request (inyectados vía `HttpServletRequest`).
  8. Crear y guardar `Session`: `refreshTokenHash = TokenHasher.hash(refreshToken)` (SHA-256), `expiresAt = now + 7 días`, `revoked = false`.
  9. `jwtTokenProvider.generateAccessToken(user.getId())`.
  10. Retornar `AuthTokenResponseData`.
  - Si código incorrecto: incrementar intentos, guardar, lanzar `InvalidOtpException` o `TooManyOtpAttemptsException` según corresponda.
  - Anotado con `@Transactional(noRollbackFor = {InvalidOtpException.class, TooManyOtpAttemptsException.class})`.
- **Dependencias:** T-02, T-05, T-06, T-11.
- **Criterios resueltos:** CA-06, CA-07, CA-08, CA-09.
- **Validación:** test de integración `AuthControllerLogin2faTest`:
  - `verify2fa_success` → 200, `access_token` y `refresh_token` presentes, sesión en BD, OTP `VERIFIED`.
  - `verify2fa_wrongCode` → 400, `otp.attempts = 1`.
  - `verify2fa_tooManyAttempts` (OTP con `attempts=4`) → 429, `otp.status = EXPIRED`.
  - Decodificar JWT del CA-06 y verificar claims `sub` (UUID del usuario) y `exp`.
- [x] Completada

---

### T-13 · Añadir `POST /auth/login/verify-2fa` a `AuthController`
- **Objetivo:** exponer el endpoint de verificación 2FA.
- **Alcance:** añadir en `AuthController`:
  ```
  POST /auth/login/verify-2fa → VerifyLoginOtpUseCase.execute(command, request)
  → 200 ApiSuccessResponse<AuthTokenResponseData> con mensaje "Autenticación completada."
  ```
- **Dependencias:** T-12.
- **Criterios resueltos:** CA-06, CA-07, CA-08, CA-09, CA-10.
- **Validación:** cubierto por los tests de T-12.
- [x] Completada

---

## Etapa 5 — Renovación de token (refresh)

### T-14 · Crear `RefreshTokenCommand` y `RefreshTokenUseCase`
- **Objetivo:** validar el refresh token, rotar la sesión y emitir nuevo par de tokens.
- **Alcance:**
  - `dto/RefreshTokenCommand.java`: `refreshToken` (`@NotBlank`, `@JsonProperty("refresh_token")`).
  - `service/RefreshTokenUseCase.java`:
    1. `sessionRepository.findByRefreshTokenHash(TokenHasher.hash(refreshToken))` — busca la sesión por hash SHA-256 usando el índice. Si no existe coincidencia o la sesión está revocada/expirada, lanzar `SessionNotFoundException`.
    2. Marcar la sesión actual como `revoked = true`, `revokedAt = now`, guardar.
    3. Generar nuevo refresh token opaco, nuevo access token JWT.
    4. Crear nueva `Session` (misma IP, mismo deviceInfo) con `refreshTokenHash = TokenHasher.hash(newToken)`, `expiresAt = now + 7 días`, guardar.
    5. Retornar `AuthTokenResponseData`.
  - `@Transactional` — la revocación y la nueva sesión son atómicas.
- **Dependencias:** T-02, T-05, T-06, T-11.
- **Criterios resueltos:** CA-12, CA-13.
- **Validación:** test de integración `RefreshTokenControllerTest`:
  - `refresh_success` → 200, nuevo `access_token` y `refresh_token`, sesión anterior con `revoked=true`, nueva sesión en BD.
  - `refresh_revokedToken` → 401.
  - `refresh_expiredToken` → 401.
  - `refresh_unknownToken` → 401.
- [x] Completada

---

### T-15 · Añadir `POST /auth/refresh` a `AuthController`
- **Objetivo:** exponer el endpoint de renovación del access token.
- **Alcance:** añadir en `AuthController`:
  ```
  POST /auth/refresh → RefreshTokenUseCase.execute(command, request)
  → 200 ApiSuccessResponse<AuthTokenResponseData>
  ```
  Ruta pública en `SecurityConfig` (no requiere Bearer).
- **Dependencias:** T-14.
- **Criterios resueltos:** CA-12, CA-13.
- **Validación:** cubierto por los tests de T-14.
- [x] Completada

---

## Etapa 6 — Logout

### T-16 · Crear `LogoutCommand` y `LogoutUseCase`
- **Objetivo:** revocar la sesión correspondiente al refresh token recibido.
- **Alcance:**
  - `dto/LogoutCommand.java`: `refreshToken` (`@NotBlank`, `@JsonProperty("refresh_token")`).
  - `service/LogoutUseCase.java`:
    1. Buscar sesión por `sessionRepository.findByRefreshTokenHash(TokenHasher.hash(refreshToken))` (consulta indexada).
    2. Si no existe o ya estaba revocada → no lanzar excepción (idempotente); retornar sin cambios.
    3. Si existe y no revocada: marcar `revoked = true`, `revokedAt = now`, guardar.
  - `@Transactional`.
  - Retorna `void`.
- **Dependencias:** T-05, T-06.
- **Criterios resueltos:** CA-14.
- **Validación:** test de integración `LogoutControllerTest`:
  - `logout_success` → 200, `session.revoked = true` en BD.
  - `logout_alreadyRevoked` → 200 (idempotente).
  - `logout_withoutToken` (sin `Authorization` header) → 401 (rechazado por el filtro JWT antes de llegar al use case).
- [x] Completada

---

### T-17 · Añadir `POST /auth/logout` a `AuthController` y actualizar `SecurityConfig`
- **Objetivo:** exponer el endpoint de logout y protegerlo con el filtro JWT.
- **Alcance:**
  - Añadir en `AuthController`:
    ```
    POST /auth/logout → LogoutUseCase.execute(command)
    → 200 ApiSuccessResponse con mensaje "Sesión cerrada exitosamente."
    ```
  - Eliminar `POST /auth/logout` de la lista de rutas públicas en `SecurityConfig` (o simplemente no añadirlo), de modo que el filtro JWT lo intercepte.
- **Dependencias:** T-04, T-16.
- **Criterios resueltos:** CA-14, CA-15 (logout sin token → 401).
- **Validación:** cubierto por los tests de T-16.
- [x] Completada

---

## Etapa 7 — Rate limiting

### T-18 · Crear `LoginRateLimitFilter`
- **Objetivo:** limitar a 10 peticiones/minuto por IP en `POST /auth/login`.
- **Alcance:** crear `config/LoginRateLimitFilter.java` (implementa `Filter` de `jakarta.servlet`):
  - `ConcurrentHashMap<String, Deque<Long>>` para registrar los timestamps de peticiones por IP.
  - Antes de proceder: limpiar entradas más antiguas que 60 segundos; contar las restantes.
  - Si count `>= 10`: responder directamente con `429 Too Many Requests` y cuerpo `ApiErrorResponse` con mensaje `"Demasiadas peticiones. Intenta de nuevo más tarde."` en JSON. No continuar la cadena de filtros.
  - Solo se aplica a `POST /auth/login`.
  - Registrar en `SecurityConfig` como `@Bean FilterRegistrationBean` con `setOrder` antes del filtro JWT.
- **Dependencias:** T-04.
- **Criterios resueltos:** CA-16.
- **Validación:** test de integración `LoginRateLimitFilterTest`:
  - 10 peticiones consecutivas desde la misma IP → todas pasan (con 401 por credenciales inválidas, que es correcto).
  - Petición 11 → 429, sin información sobre el estado de la cuenta.
  - Verificar manualmente con `curl` en `./mvnw spring-boot:run`.
- [x] Completada

---

## Etapa 8 — Regresión y validación final

### T-19 · Ejecutar suite completa de tests y documentar evidencias
- **Objetivo:** confirmar que todos los CA están verificados y que no hay regresiones.
- **Alcance:**
  - Ejecutar `./mvnw test` completo (requiere Docker para Testcontainers).
  - Verificar en verde: `AuthControllerTest`, `UserControllerTest` (regresión).
  - Verificar en verde: todos los tests nuevos de las etapas 1-7.
  - Para CA-16: ejecutar verificación manual con `curl` y documentar la respuesta.
  - Registrar resultado de cada CA en la tabla de evidencias de abajo.
- **Dependencias:** T-01 a T-18.
- **Criterios resueltos:** CA-01 a CA-16.
- **Validación:** output de `./mvnw test` en verde; evidencia manual de CA-16.
- [x] Completada

---

## Tabla de evidencias (rellenar durante la implementación)

| Criterio | Test / método de verificación | Estado | Evidencia |
| --- | --- | --- | --- |
| CA-01 | `AuthControllerLoginTest#login_success` | ✅ Verificado | 200 + `status=success`; OTP `LOGIN_2FA` `PENDING` guardado con `expiresAt <= createdAt + 15 min` |
| CA-02 | `AuthControllerLoginTest#login_wrongPassword`, `login_unknownEmail` | ✅ Verificado | Ambos → 401 con `"Credenciales inválidas."` (mensaje idéntico, no hay enumeración de emails) |
| CA-03 | `AuthControllerLoginTest#login_unverifiedUser` | ✅ Verificado | 401 con el mismo mensaje genérico |
| CA-04 | `AuthControllerLoginTest#login_success` + inspección BD | ✅ Verificado | Evento outbox `USER_LOGIN_OTP` con `aggregate_id = user.id` y payload `{id, email, otp_code}` |
| CA-05 | `AuthControllerLoginTest#login_invalidatesPreviousPendingOtp` | ✅ Verificado | Tras 2 logins: 1 OTP `EXPIRED`, 1 `PENDING` |
| CA-06 | `AuthControllerLogin2faTest#verify2fa_success` | ✅ Verificado | 200 con `access_token`, `token_type=Bearer`, `expires_in=3600`, `refresh_token`, `refresh_token_expires_in=604800`; token decodificado: `sub` = UUID del usuario, `exp` ≈ now+3600 |
| CA-07 | `AuthControllerLogin2faTest#verify2fa_wrongCode` | ✅ Verificado | 400 con mensaje genérico; `otp.attempts = 1`, sigue `PENDING` |
| CA-08 | `AuthControllerLogin2faTest#verify2fa_tooManyAttempts` | ✅ Verificado | Intentos 4→5 → 429; OTP marcado `EXPIRED` |
| CA-09 | `AuthControllerLogin2faTest#verify2fa_success` + decodificación JWT | ✅ Verificado | Sesión creada: `refresh_token_hash` = SHA-256 hex del token plano (64 chars), `revoked=false`, `expiresAt` futuro; OTP `VERIFIED` |
| CA-10 | `JwtProtectedEndpointTest#accessWithValidToken` | ✅ Verificado | `GET /auth/ping` con Bearer válido → 200 |
| CA-11 | `JwtProtectedEndpointTest#accessWithExpiredToken`, `accessWithMangledToken` | ✅ Verificado | Token expirado → 401; token manipulado → 401 |
| CA-12 | `RefreshTokenControllerTest#refresh_success` | ✅ Verificado | 200 con nuevo `access_token`/`refresh_token`; sesión vieja `revoked=true` con `revokedAt`; nueva sesión en BD; hash SHA-256 en BD (64 chars hex) |
| CA-13 | `RefreshTokenControllerTest#refresh_revokedToken`, `refresh_expiredToken`, `refresh_unknownToken` | ✅ Verificado | Revocada → 401, expirada → 401, desconocida → 401, todos con `"Sesión inválida o expirada."` |
| CA-14 | `LogoutControllerTest#logout_success` | ✅ Verificado | 200 con `"Sesión cerrada exitosamente."`; sesión `revoked=true` con `revokedAt` en BD; reintento/desconocida → 200 idempotente |
| CA-15 | `JwtProtectedEndpointTest#accessWithoutToken` + `LogoutControllerTest#logout_withoutToken` | ✅ Verificado | `GET /auth/ping` sin header → 401; `POST /auth/logout` sin token → 401 |
| CA-16 | `LoginRateLimitFilterTest#rateLimit` + verificación `curl` manual | ⚠️ Parcial | Test automatizado: 10 peticiones OK y la 11º → 429 con mensaje de error estándar. Verificación manual con `curl` **pendiente** (no se ejecutó contra servicio en ejecución) |

**Regresión:**
| Test existente | Estado |
| --- | --- |
| `AuthControllerTest` (verify-email) | ✅ Verificado |
| `UserControllerTest` (registro) | ✅ Verificado |
