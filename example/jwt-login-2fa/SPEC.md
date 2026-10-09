# SPEC: Login con JWT y autenticación de dos factores (2FA)

**Estado:** Aprobada <!-- Borrador | En revisión | Aprobada -->

<!-- PARA LA PERSONA
Copia esta plantilla como SPEC.md en una carpeta de la funcionalidad.
Pide al agente que la complete contigo.
SPEC.md define qué debe cumplirse; PLAN.md desarrolla cómo implementarlo;
TASKS.md organiza los pasos de ejecución.
-->

<!-- PARA EL AGENTE
- Lee las instrucciones del proyecto. Inspecciona el
  repositorio para comprobar el comportamiento actual.
- Completa esta spec con la persona: investiga lo comprobable y consulta las
  decisiones pendientes. Haz pocas preguntas por vez y actualiza las respuestas.
- No inventes requisitos ni exclusiones. Distingue propuestas de decisiones
  confirmadas y marca como PENDIENTE lo que aún no esté resuelto.
- Aplica las consideraciones backend relevantes sin ampliar el alcance automáticamente.
- No incluyas diseño de clases, tablas, componentes, archivos o algoritmos:
  esos detalles pertenecen a PLAN.md. Sí registra restricciones explícitas del pedido.
- Mantén el documento breve y proporcional a la funcionalidad. Conserva los comentarios.
- Un documento completo no está aprobado automáticamente. Solicita aprobación
  antes de marcarlo como Aprobada. No implementes durante esta etapa.
-->

## Qué construimos y para quién

<!-- Qué necesidad resolvemos, quién tiene esa necesidad y qué podrá hacer.
Describe el objetivo en lenguaje de producto. -->

Un usuario registrado y con el correo verificado necesita poder autenticarse
en SplitFlow. Hoy el servicio no tiene ningún mecanismo de login: no emite
tokens, no verifica credenciales y no protege ningún endpoint.

Con esta funcionalidad el usuario podrá iniciar sesión con email y contraseña
y, al completar un segundo factor de verificación por correo electrónico,
recibirá un **access token JWT** de corta duración y un **refresh token** para
renovarlo sin volver a autenticarse. Los consumidores del API podrán adjuntar
el access token en cada petición para acceder a recursos protegidos.

## Situación actual

<!-- Comportamiento actual relevante, limitación que queremos resolver y
comportamientos existentes que deben conservarse. No describas la arquitectura. -->

- El servicio permite registrar usuarios (`POST /user/`) y verificar el correo
  con un código OTP de 6 dígitos (`POST /auth/verify-email`).
- La tabla `session` ya existe en la base de datos con columnas para
  `refresh_token_hash`, `ip_address`, `device_info`, `revoked`, `revoked_at`,
  `last_used_at`, `created_at`, `updated_at` y `expires_at`. No hay código de
  producción que la use todavía.
- El enum `OtpPurpose` ya incluye el valor `LOGIN_2FA`, lo que indica que el
  diseño contempla OTPs de propósito específico por paso.
- No existe ningún endpoint de login, emisión de tokens ni renovación de
  sesión. Spring Security está configurado con `STATELESS` y sin JWT.
- El patrón de error y de éxito (`ApiSuccessResponse<T>` / `ApiErrorResponse`)
  y la lógica de OTP (límite de 5 intentos, expiración en 15 minutos,
  transacción con `noRollbackFor`) son el estándar del proyecto y deben
  conservarse en esta funcionalidad.

## Dentro del alcance

<!-- Requisitos concretos, con identificadores estables para vincularlos a
criterios, decisiones del plan y tareas. -->

- **RF-01 — Validación de credenciales:** El sistema debe verificar que el
  email existe, que el usuario tiene el correo verificado y que la contraseña
  coincide con el hash almacenado. Si alguna condición falla, debe responder
  con un mensaje genérico que no revele si el email existe o no.

- **RF-02 — Envío del código 2FA por correo:** Si las credenciales son
  correctas, el sistema genera un código OTP de propósito `LOGIN_2FA` y lo
  publica en el outbox para que el servicio de notificaciones lo envíe al
  correo del usuario.

- **RF-03 — Verificación del código 2FA:** El usuario envía el código recibido.
  El sistema verifica que el código sea correcto, que no haya expirado y que no
  se hayan superado los intentos permitidos. En ese momento emite los tokens.

- **RF-04 — Emisión de tokens JWT:** Al completar el 2FA satisfactoriamente,
  el sistema emite un **access token** (JWT firmado, corta duración) y un
  **refresh token** (opaco, almacenado como hash en la tabla `session`).

- **RF-05 — Renovación del access token:** El sistema acepta un refresh token
  válido y emite un nuevo access token **y un nuevo refresh token**, invalidando el
  anterior (rotación en cada renovación).

- **RF-06 — Revocación de sesión (logout):** El usuario puede invalidar su
  sesión. El sistema marca el refresh token como revocado.

- **RF-07 — Protección de endpoints existentes:** Los endpoints que no sean
  de registro, verificación de correo o inicio de sesión deben requerir un
  access token válido en `Authorization: Bearer`. Actualmente ningún endpoint
  existente requiere protección porque no hay recursos autenticados
  implementados; esta restricción se aplica a cualquier endpoint nuevo que
  se añada tras esta funcionalidad.

- **RF-08 — Protección contra fuerza bruta en el login:** Se aplica rate
  limiting por IP en el endpoint `POST /auth/login`. El umbral y los detalles
  de configuración se definen en PLAN.md; el comportamiento observable es que
  las peticiones que superen el límite reciben `429 Too Many Requests`.

## Fuera de alcance

<!-- Exclusiones acordadas, no deducidas por el agente. Si no hay exclusiones
adicionales, indícalo tras revisarlo con la persona. -->

- Recuperación de contraseña (el propósito `PASSWORD_RESET` ya está en el enum
  pero no es parte de esta entrega).
- 2FA mediante aplicación TOTP (Google Authenticator, Authy, etc.). Esta
  entrega usa únicamente OTP por correo electrónico.
- OAuth2 / inicio de sesión con terceros (Google, GitHub, etc.).
- Gestión de múltiples sesiones activas por usuario desde el frontend
  (visualización, revocación individual desde UI).
- La inscripción al 2FA es obligatoria para todos los usuarios verificados;
  no existe opción de omitirlo.

## Flujo del cliente

<!-- Cómo se inicia, qué peticiones hace y qué resultado obtiene.
Incluye endpoints afectados y alternativas relevantes. -->

**Paso 1 — Iniciar sesión:**
1. El cliente envía `POST /auth/login` con `email` y `password`.
2. Si las credenciales son correctas y el correo está verificado, el servidor
   responde con `200 OK` e indica que se ha enviado un código 2FA al correo.
   El cuerpo no incluye ningún token todavía.
3. Si las credenciales son incorrectas (o el correo no existe), el servidor
   responde con `401 Unauthorized` y un mensaje genérico.
4. Si el correo no está verificado o las credenciales no son válidas (incluyendo
   email inexistente), el servidor responde con `401 Unauthorized` y el mismo
   mensaje genérico en todos los casos.

**Paso 2 — Verificar el código 2FA:**
5. El cliente envía `POST /auth/login/verify-2fa` con `email` y `otp_code`.
6. Si el código es correcto y no ha expirado, el servidor responde con `200 OK`
   y devuelve el `access_token` (JWT), el `refresh_token` (opaco) y sus
   tiempos de expiración.
7. Si el código es incorrecto, el servidor responde con `400 Bad Request`. No
   se expone el número de intentos restantes en la respuesta.
8. Si se superan los intentos permitidos, el servidor responde con
   `429 Too Many Requests` (misma convención que `verify-email`).
9. Si el código ha expirado, el servidor responde con `400 Bad Request` y un
   mensaje de expiración.

**Paso 3 — Acceder a recursos protegidos:**
10. El cliente adjunta el access token en el encabezado
    `Authorization: Bearer <token>` en cada petición.
11. Si el token es válido, el servidor procesa la petición normalmente.
12. Si el token ha expirado o es inválido, el servidor responde con
    `401 Unauthorized`.

**Paso 4 — Renovar el access token:**
13. El cliente envía `POST /auth/refresh` con el `refresh_token`.
14. Si el refresh token es válido y no está revocado, el servidor emite un
    nuevo access token y un nuevo refresh token, invalidando el anterior.
15. Si el refresh token no es válido o está revocado, el servidor responde con
    `401 Unauthorized`.

**Paso 5 — Cerrar sesión:**
16. El cliente envía `POST /auth/logout` con el `refresh_token`.
17. El servidor revoca la sesión y responde con `200 OK`.

## Datos y reglas de negocio

<!-- Información que necesita el usuario, campos obligatorios, validaciones,
límites y reglas como duplicados u orden de presentación. Describe significado
y comportamiento, sin diseñar tablas, DTO, DAO ni almacenamiento. -->

**Login (`POST /auth/login`):**
- `email`: obligatorio, formato email válido.
- `password`: obligatorio, sin restricciones de complejidad en este endpoint
  (la validación de fortaleza ya ocurrió en el registro).
- El mensaje de error ante credenciales incorrectas, cuenta inexistente o
  correo no verificado debe ser idéntico para no revelar si el email existe.

**Código 2FA:**
- El código OTP reutiliza el mecanismo existente: 6 dígitos numéricos,
  expiración en 15 minutos, máximo 5 intentos.
- Se almacena con `purpose = LOGIN_2FA`.
- Si el usuario inicia sesión de nuevo mientras hay un código 2FA pendiente
  vigente, el OTP anterior se invalida (estado `EXPIRED`) y se genera uno nuevo.

**Verify 2FA (`POST /auth/login/verify-2fa`):**
- `email`: obligatorio, se usa para encontrar al usuario y su OTP activo.
- `otp_code`: obligatorio, 6 dígitos.

**Access token:**
- JWT firmado con **RS256** (par de claves pública/privada). Esto permite que
  otros microservicios de SplitFlow validen el token sin compartir la clave privada.
- Contiene los claims: `sub` (user UUID) y `exp`. No se incluyen claims adicionales
  en esta entrega.
- Duración: **1 hora**.

**Refresh token:**
- Token opaco generado aleatoriamente; se almacena como hash en `session`.
- La sesión registra `ip_address`, `device_info`, `expires_at`.
- Duración: **7 días**.
- Se implementa **rotación**: cada vez que se usa el refresh token para renovar
  el access token, se emite un nuevo refresh token y el anterior queda revocado.

**Logout (`POST /auth/logout`):**
- Requiere un access token válido en `Authorization: Bearer` **y** el `refresh_token`
  en el cuerpo.
- Solo se revoca la sesión correspondiente al refresh token recibido; otras
  sesiones activas del mismo usuario no se ven afectadas.

**Rate limiting en el login:**
- Se aplica rate limiting por IP en `POST /auth/login`.
- Cuando se supera el límite, el servidor responde con `429 Too Many Requests`.
- El umbral concreto (peticiones por minuto/hora) se define en PLAN.md.
- La respuesta ante rate limit no revela información sobre el estado de la cuenta.

## Comportamiento y casos alternativos

<!-- Añade escenarios relevantes. Marca No aplica con su motivo cuando corresponda. Expresa resultados, no mecanismos técnicos. -->

| Situación | Comportamiento esperado |
| --- | --- |
| Email o contraseña incorrectos | 401 con mensaje genérico: no indica si el email existe ni si la contraseña está mal |
| Correo sin verificar | 401 con el mismo mensaje genérico que credenciales incorrectas |
| Usuario inexistente | 401 con el mismo mensaje genérico que credenciales incorrectas |
| Código 2FA incorrecto | 400, incrementa el contador de intentos |
| Código 2FA expirado (tiempo) | 400 con mensaje de expiración |
| Código 2FA agotado (5 intentos) | 429, OTP marcado como EXPIRED, igual que en `verify-email` |
| Access token expirado | 401; el cliente debe usar el refresh token para renovarlo |
| Access token manipulado o inválido | 401 con mensaje genérico |
| Refresh token revocado | 401; el cliente debe iniciar sesión de nuevo |
| Refresh token inexistente o inválido | 401 con mensaje genérico |
| Logout con refresh token válido | 200, sesión revocada |
| Logout con refresh token ya revocado | 200 OK idempotente |
| Petición a endpoint protegido sin token | 401 |
| Dos peticiones de login simultáneas para el mismo usuario | Se invalida el OTP anterior y se genera uno nuevo para la segunda petición |
| Cambio de contraseña (futuro) | Fuera de alcance; se documenta aquí que al implementarlo deberá invalidar sesiones activas |

**Puntos de la guía no aplicables y motivo:**
- *Idempotency y duplicate prevention (logout)*: pendiente de decisión sobre si el logout es idempotente.
- *Long-running / background work*: no aplica; el envío del OTP se delega al outbox de forma síncrona como en `RegisterUseCase`.
- *Cancellation and client disconnects*: no aplica; las operaciones son de corta duración.
- *Internacionalización*: no aplica en esta entrega; los mensajes de error siguen el mismo patrón en español ya establecido.

## Restricciones del pedido

<!-- Condiciones ya impuestas: compatibilidad, límites de alcance, requisitos
de accesibilidad o rendimiento medibles, o una tecnología expresamente exigida.
Ejemplo: Usar PostgreSQL puede ser una restricción; el diseño de entidades va en PLAN.md.
No conviertas una preferencia del agente en una restricción. -->

- El mecanismo de 2FA debe usar OTP por **correo electrónico** (igual que la
  verificación de email). No se implementa TOTP ni SMS en esta entrega.
- Se deben emitir **tokens JWT** (no sesiones en cookie, no OAuth2).
- La tabla `session` ya existe en la base de datos y debe usarse para
  almacenar los refresh tokens; no se puede crear una tabla alternativa.
- El límite de intentos OTP (5) y la expiración (15 min) se reutilizan del
  mecanismo existente a menos que se decida explícitamente otro valor.
- Los mensajes de error en el login deben ser **genéricos** (no revelan si el
  email existe), siguiendo los principios de seguridad del API Guidelines.
- El refresh token se almacena como hash **SHA-256** (hex) en `session.refresh_token_hash`.
  SHA-256 es suficiente porque el token es un UUID v4 de 128 bits de entropía; no
  se requiere hash lento (BCrypt) para valores de alta entropía generados aleatoriamente.
- Se debe mantener la compatibilidad con los endpoints existentes
  (`POST /user/`, `POST /auth/verify-email`); esta funcionalidad no los modifica.

## Criterios de aceptación

<!-- Resultados observables que permitan decidir si se cumple cada requisito.
Incluye los casos alternativos acordados. No uses Funciona correctamente.
Repite el formato según sea necesario. -->

- **CA-01 · RF-01:** Dado un usuario registrado y verificado, cuando se envía
  `POST /auth/login` con email y contraseña correctos, entonces el servidor
  responde con `200 OK`, sin access token en el cuerpo, e indica que se ha
  enviado un código al correo.

- **CA-02 · RF-01:** Dado cualquier email y contraseña incorrectos (o email
  inexistente), cuando se envía `POST /auth/login`, entonces el servidor
  responde con `401 Unauthorized` y el mismo mensaje genérico en todos los
  casos (sin indicar si el email existe).

- **CA-03 · RF-01:** Dado un usuario con correo sin verificar, cuando se envía
  `POST /auth/login` con credenciales correctas, entonces el servidor responde
  con `401 Unauthorized` y el mismo mensaje genérico que en CA-02.

- **CA-04 · RF-02:** Dado que el login es exitoso (CA-01), cuando se inspecciona
  la tabla `otp`, entonces existe un registro con `purpose = LOGIN_2FA`,
  `status = PENDING` y `expires_at` dentro de 15 minutos.

- **CA-05 · RF-02:** Dado que el login es exitoso, cuando se inspecciona la
  tabla `outbox`, entonces existe un evento pendiente de tipo `USER_LOGIN_OTP`
  (o el nombre que se acuerde) con el `otp_code` en el payload.

- **CA-06 · RF-03:** Dado un código 2FA válido y no expirado, cuando se envía
  `POST /auth/login/verify-2fa` con email y código correctos, entonces el
  servidor responde con `200 OK`, `access_token` (JWT), `refresh_token` (opaco)
  y sus respectivos tiempos de expiración.

- **CA-07 · RF-03:** Dado un código 2FA incorrecto, cuando se envía
  `POST /auth/login/verify-2fa`, entonces el servidor responde con `400` y el
  contador de intentos del OTP se incrementa en 1.

- **CA-08 · RF-03:** Dado que se han realizado 5 intentos fallidos en el 2FA,
  cuando se envía el quinto intento incorrecto, entonces el servidor responde con
  `429 Too Many Requests` y el OTP queda con `status = EXPIRED`.

- **CA-09 · RF-04:** Dado el `access_token` emitido en CA-06, cuando se decodifica
  (sin verificar firma), entonces contiene al menos los claims `sub` (user ID)
  y `exp` (tiempo de expiración esperado).

- **CA-10 · RF-04:** Dado el `access_token` emitido en CA-06, cuando se envía
  una petición a un endpoint protegido con `Authorization: Bearer <token>`,
  entonces el servidor responde con `200 OK`.

- **CA-11 · RF-04:** Dado un access token manipulado o expirado, cuando se
  envía una petición a un endpoint protegido, entonces el servidor responde
  con `401 Unauthorized`.

- **CA-12 · RF-05:** Dado un `refresh_token` válido, cuando se envía
  `POST /auth/refresh`, entonces el servidor responde con `200 OK` y un nuevo
  `access_token` válido.

- **CA-13 · RF-05:** Dado un `refresh_token` revocado o inválido, cuando se
  envía `POST /auth/refresh`, entonces el servidor responde con `401 Unauthorized`.

- **CA-14 · RF-06:** Dado un `refresh_token` válido, cuando se envía
  `POST /auth/logout`, entonces el servidor responde con `200 OK` y el registro
  de sesión queda con `revoked = true`.

- **CA-15 · RF-07:** Dado una petición a un endpoint protegido sin encabezado
  `Authorization`, entonces el servidor responde con `401 Unauthorized`.

- **CA-16 · RF-08:** Dado que se supera el límite de peticiones por IP en
  `POST /auth/login`, cuando se envía una petición adicional, entonces el
  servidor responde con `429 Too Many Requests` sin revelar información sobre
  el estado de la cuenta.

## Cómo se comprueba el comportamiento

<!-- Una fila por criterio: escenario y resultado que debemos comprobar.
La selección de tests, herramientas, comandos y evidencias se desarrolla en PLAN.md.
No marques los criterios como superados durante la especificación. -->

| Criterio | Condiciones y pasos | Resultado esperado |
| --- | --- | --- |
| CA-01 | Usuario verificado; `POST /auth/login` con credenciales correctas | 200, sin token, mensaje de OTP enviado |
| CA-02 | Credenciales incorrectas o email inexistente | 401, mensaje genérico idéntico en ambos casos |
| CA-03 | Usuario no verificado, credenciales correctas | 401, mismo mensaje genérico que CA-02 |
| CA-04 | Inspección de BD tras CA-01 | OTP con `purpose=LOGIN_2FA`, `status=PENDING` |
| CA-05 | Inspección de BD tras CA-01 | Evento en outbox con OTP code |
| CA-06 | OTP válido; `POST /auth/login/verify-2fa` | 200, `access_token` y `refresh_token` presentes |
| CA-07 | OTP incorrecto (primer intento) | 400, `attempts` = 1 en BD |
| CA-08 | Quinto intento incorrecto de OTP | 429, OTP con `status=EXPIRED` |
| CA-09 | Decodificar el JWT emitido | Claims `sub` y `exp` presentes con valores esperados |
| CA-10 | Access token válido en endpoint protegido | 200 |
| CA-11 | Token manipulado o expirado en endpoint protegido | 401 |
| CA-12 | Refresh token válido; `POST /auth/refresh` | 200, nuevo `access_token` |
| CA-13 | Refresh token revocado; `POST /auth/refresh` | 401 |
| CA-14 | Refresh token válido; `POST /auth/logout` | 200, `revoked=true` en BD |
| CA-15 | Petición a endpoint protegido sin `Authorization` | 401 |
| CA-16 | IP supera el límite de peticiones en `POST /auth/login` | 429, sin información sobre el estado de la cuenta |

## Decisiones pendientes

<!-- Al resolverlas, actualiza las secciones afectadas. Escribe Ninguna cuando
no queden pendientes funcionales ni restricciones por decidir. -->

Ninguna. Todas las decisiones han sido resueltas:

| ID | Decisión |
| --- | --- |
| DP-01 | Algoritmo JWT: **RS256** |
| DP-02 | Access token: **1 hora** |
| DP-03 | Refresh token / sesión: **7 días** |
| DP-04 | Rotación del refresh token: **sí, en cada renovación** |
| DP-05 | OTP pendiente ante segundo login: **invalidar el anterior, generar uno nuevo** |
| DP-06 | Correo sin verificar: **401 genérico** (igual que credenciales incorrectas) |
| DP-07 | Fuerza bruta: **rate limiting por IP**, umbral definido en PLAN.md |
| DP-08 | Logout idempotente: **200 OK** aunque el token ya esté revocado |
| DP-09 | Claims JWT: solo `sub` y `exp` en esta entrega |
| DP-10 | Logout: requiere **access token en `Authorization`** y `refresh_token` en el cuerpo |

<!-- ANTES DE SOLICITAR APROBACIÓN
Comprueba que el alcance está acordado, los flujos son coherentes, los puntos
backend relevantes están cubiertos y cada requisito tiene criterios comprobables.
Resuelve las dudas y los marcadores pendientes. Mantén el diseño técnico en PLAN.md.
-->
