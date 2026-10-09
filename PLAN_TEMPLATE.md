# PLAN: [nombre de la funcionalidad]

**SPEC de referencia:** [ruta a SPEC.md]
**Versión de la spec revisada:** [commit, versión o fecha que permita identificarla]
**Contrato de API afectado:** [ruta a OpenAPI/schema, o "Nuevo endpoint" / "No aplica"]
**Estado:** Borrador <!-- Borrador | En revisión | Aprobado -->

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
| [PENDIENTE] | [PENDIENTE] | [PENDIENTE] |

**Convenciones y patrón de referencia:** [PENDIENTE]

## Solución propuesta

<!-- Explica el enfoque y sus motivos. Describe las responsabilidades y el
recorrido de datos desde la petición hasta la respuesta y/o efectos secundarios
(persistencia, eventos, integraciones). Usa un diagrama si aporta claridad. -->

[PENDIENTE]

## Contrato de API

<!-- Define o referencia el contrato expuesto o modificado. Si ya existe una
definición formal (OpenAPI/JSON Schema/protobuf), referencia la ruta en lugar
de copiarla y describe solo el diff. -->

| Método | Ruta | Autenticación / autorización | Códigos de estado esperados | Requisito relacionado |
| --- | --- | --- | --- | --- |
| [GET/POST/PUT/PATCH/DELETE] | [/ruta] | [Rol, scope o público] | [200/201/400/404/409/422/500...] | [RF-XX] |

- **Esquema de request:** [PENDIENTE o referencia]
- **Esquema de response:** [PENDIENTE o referencia]
- **Compatibilidad hacia atrás:** [¿Rompe el contrato actual? ¿Requiere versionado nuevo? ¿Quién consume este endpoint hoy?]

## Módulos y componentes afectados

<!-- Si el proyecto está modularizado, identifica los módulos afectados, sus
responsabilidades y la dirección de sus dependencias. Respeta los límites
existentes y justifica cualquier módulo o dependencia nueva. Si no está
modularizado, describe las carpetas o componentes afectados sin introducir
modularización fuera del alcance; marca la tabla de módulos como No aplica. -->

| Módulo | Existe / nuevo | Responsabilidad y cambios | Dependencias afectadas |
| --- | --- | --- | --- |
| [Módulo y ruta] | [Existente / propuesto] | [Qué cambia] | [Qué módulos utiliza o pasan a depender de él] |

<!-- Distingue lo que se reutiliza, modifica o crea. Las rutas nuevas son propuestas.
Señala impacto sobre modelos, contratos o componentes compartidos. Indica la capa
(controller/handler, servicio o caso de uso, repositorio o acceso a datos,
integración externa) para detectar si se está filtrando lógica de negocio a la
capa de transporte. -->

| Componente o ruta | Capa | Acción | Cambio y responsabilidad | Requisito relacionado |
| --- | --- | --- | --- | --- |
| [PENDIENTE] | [Handler / Servicio / Repositorio / Integración] | [Reutilizar / modificar / crear] | [PENDIENTE] | [RF-XX] |

## Datos y contratos

<!-- Completa solo lo aplicable. Si un punto no aplica, indica el motivo. -->

- **Modelos de dominio y su relación con el contrato de entrada y salida:** [PENDIENTE]
- **Identificadores, relaciones y restricciones:** [PENDIENTE]
- **Origen de los datos y transformaciones (entrada → dominio → persistencia/salida):** [PENDIENTE]
- **Persistencia, consultas y actualizaciones:** [PENDIENTE]
- **Consistencia entre servicios o fuentes de datos** (si el endpoint depende de otra API, otro servicio o un caché): [PENDIENTE o "No aplica"]
- **Compatibilidad y migraciones de datos existentes:** [PENDIENTE]

## Seguridad y validación de entrada

<!-- Qué se valida antes de procesar la petición y qué datos requieren
tratamiento especial. -->

- **Validación y sanitización de inputs:** [PENDIENTE]
- **Autenticación y autorización requeridas:** [PENDIENTE]
- **Datos sensibles (PII, secretos) y su tratamiento en almacenamiento y logs:** [PENDIENTE]
- **Rate limiting / throttling:** [PENDIENTE o "No aplica"]

## Rendimiento y escalabilidad

<!-- Completa solo lo aplicable. -->

- **Impacto en consultas a base de datos (índices nuevos, riesgo de N+1):** [PENDIENTE]
- **Paginación** (si el endpoint devuelve colecciones): [PENDIENTE o "No aplica"]
- **Estrategia de caché:** [PENDIENTE o "No aplica"]

## Estado, operaciones y errores

<!-- Cómo se implementan los comportamientos aprobados en la spec. -->

- **Transaccionalidad:** [Qué operaciones deben ser atómicas y cómo se maneja el rollback]
- **Idempotencia:** [Si el endpoint debe soportar reintentos seguros, especialmente en POST/PUT]
- **Concurrencia de acceso a datos:** [Locks, condiciones de carrera o escrituras simultáneas a considerar]
- **Errores, reintentos y prevención de duplicados:** [Mapeo de errores a códigos HTTP y formato de error estándar de la API]
- **Otras consideraciones aplicables y su solución:** [PENDIENTE]

## Dependencias y configuración

<!-- Librerías, servicios, permisos o configuración afectados. Verifica compatibilidad
con el proyecto y justifica las incorporaciones. No agregues dependencias por defecto. -->

- **Librerías o servicios nuevos:** [PENDIENTE]
- **Variables de entorno nuevas o modificadas:** [PENDIENTE]
- **Migraciones de base de datos (y si son reversibles):** [PENDIENTE]
- **Feature flags:** [PENDIENTE o "No aplica"]

## Estrategia de validación

<!-- Una fila por criterio de la spec. Selecciona el método capaz de demostrarlo:
test unitario, integración o prueba manual. No todos requieren todos los métodos.
Identifica tests existentes y separa los nuevos propuestos. Incluye regresiones relevantes.
Un test unitario aislado no demuestra el comportamiento end-to-end del endpoint;
una respuesta 200 no demuestra los efectos secundarios esperados (persistencia,
eventos, notificaciones). -->

| Criterio | Método y test existente o propuesto | Entorno, datos y estado inicial de la base de datos necesario | Evidencia prevista |
| --- | --- | --- | --- |
| CA-01 | [PENDIENTE] | [PENDIENTE] | [PENDIENTE] |
| CA-02 | [PENDIENTE] | [PENDIENTE] | [PENDIENTE] |

**Comprobaciones de regresión:** [PENDIENTE]

**Comandos verificados para compilar y ejecutar tests:** [PENDIENTE]

**Limitaciones del entorno:** [qué no podrá comprobarse aquí y cómo quedará pendiente]

<!-- Esta sección planifica la validación. Durante la implementación, registra
en TASKS.md o en el informe de validación acordado los resultados y evidencias
reales. Distingue pruebas ejecutadas, fallidas, no ejecutadas y bloqueadas.
Compilar o tener tests en verde no sustituye revisar los criterios de la spec. -->

## Orden de implementación

<!-- Etapas y dependencias principales. El desglose ejecutable se escribe en TASKS.md.
Incluye puntos de comprobación para avanzar con cambios pequeños. -->

1. [Etapa, dependencia y comprobación]
2. [Etapa, dependencia y comprobación]
3. [Etapa, dependencia y comprobación]

## Riesgos y decisiones pendientes

<!-- Riesgos concretos de esta solución y cómo se resolverán, sin listas genéricas.
Considera especialmente: breaking changes de contrato, migraciones irreversibles
e impacto en consumidores existentes de la API. Escribe Ninguna en las decisiones
pendientes cuando estén resueltas. -->

- **Riesgos y medidas acordadas:** [PENDIENTE]
- **Decisiones pendientes:** [PENDIENTE]

<!-- ANTES DE SOLICITAR APROBACIÓN
Comprueba que el plan cubre los requisitos, respeta las exclusiones, reutiliza
componentes verificados y permite demostrar todos los criterios de aceptación.
Resuelve dudas y marcadores pendientes. Si la spec cambió, revisa su impacto.
Tras aprobar el plan, deriva TASKS.md con IDs, dependencias, referencias a RF/CA
y comprobaciones. No marques una tarea terminada sin realizar su validación;
si está bloqueada, registra el motivo.
-->
