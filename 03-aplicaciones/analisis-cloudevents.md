# analisis-cloudevents.md
# Completa este archivo con tus respuestas al TODO 4 de la Fase 3.

## TODO 4: Análisis de Trazabilidad (CloudEvents)

Evidencia: logs del subscriber tras lanzar el publisher (3 órdenes).

| orderId | id | source | type |
|---|---|---|---|
| 1001 | `bc487015-0ca9-460c-a59c-050b31899fbe` | order-publisher | com.dapr.event.sent |
| 1002 | `2826498e-4979-44d9-a090-e70fcecee674` | order-publisher | com.dapr.event.sent |
| 1003 | `1f711a62-75dc-4c78-8ca4-83aafe6b7703` | order-publisher | com.dapr.event.sent |

### 1. El atributo `id`: ¿quién lo genera y para qué sirve?

Lo genera el sidecar de Dapr del publisher al publicar. El código del publisher solo envía `orderId` y `status`, y aun así cada evento llegó con un UUID distinto. Sirve para identificar un evento de forma única, y eso es lo que permite la idempotencia: los brokers pueden entregar un mismo mensaje más de una vez (entrega al menos una vez), así que el subscriber puede guardar los `id` ya procesados y descartar los repetidos sin duplicar el efecto.

### 2. El atributo `source`: ¿qué valor tiene y de dónde lo toma Dapr?

Vale `order-publisher`. Dapr lo toma de la anotación `dapr.io/app-id` del Pod que publicó, que en `publisher-job.yaml` es `order-publisher`. Por eso el origen del evento lo define la identidad que se le da a la app en el despliegue y no el código.

### 3. El atributo `traceparent` y su relación con OpenTelemetry

Implementa el estándar W3C Trace Context. Su formato es `versión-traceId-spanId-flags` y permite que un identificador de traza viaje con la petición entre servicios, para reconstruir su recorrido completo. OpenTelemetry es el estándar de observabilidad que usa ese mismo formato para propagar el contexto, así que un `traceparent` presente en el evento permite enlazar la traza del publisher con la del subscriber en una herramienta de tracing.

En este laboratorio el campo llegó vacío. Hay evidencia de que no hay tracing configurado (la `Configuration` `daprsystem` no tiene sección `tracing` y los Pods no usan `dapr.io/config`), pero no probé que esa sea la causa. Para poblarlo haría falta una `Configuration` con `spec.tracing.samplingRate` asociada a las apps.

### 4. Valor de negocio para auditoría: atributos más útiles del sobre CloudEvents

- `id`: identifica cada evento de forma única, así se cuenta y se detectan duplicados.
- `source`: dice qué servicio lo emitió, para filtrar por origen.
- `type`: clasifica el tipo de evento.
- `data`: el contenido de negocio (qué orden y en qué estado).
- `traceparent`: si estuviera poblado, permitiría seguir un evento a través de todos los servicios que tocó.

Para filtrar por las últimas 24 h también haría falta la marca de tiempo del evento (el atributo `time` del estándar). Nuestro subscriber no lo imprime, así que no lo verifiqué en estos logs.
