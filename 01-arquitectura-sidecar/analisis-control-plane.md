# analisis-control-plane.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 1.

## TODO 1: Análisis del Plano de Control

### 1. ¿Qué tipo de objeto de Kubernetes es el `dapr-sidecar-injector`?

El `dapr-sidecar-injector` es un **Deployment** que sirve un endpoint HTTP (`/mutate`, puerto 443 a través de un Service) y se registra en el API server mediante un **MutatingWebhookConfiguration**. Debe ser *Mutating* y no *Validating* porque su trabajo no es aprobar o rechazar el Pod, sino **modificarlo**: cuando detecta la anotación `dapr.io/enabled: "true"`, devuelve un parche que añade el contenedor `daprd` al Pod antes de que se guarde en etcd.

El webhook está configurado para la operación `CREATE` sobre `pods` y con `failurePolicy: Ignore`: si el injector está caído, los Pods se crean igualmente, pero sin sidecar.

### 2. Réplicas del `dapr-operator` y comparación con Istio

Con la instalación por defecto en este clúster, el `dapr-operator` corre con **1 réplica** (en producción se usa alta disponibilidad con `global.ha.enabled=true`).

Dapr es más liviano que Istio porque no gestiona el tráfico de red del clúster. Istio (service mesh) intercepta de forma transparente todo el tráfico de cada Pod, y por eso su plano de control debe distribuir configuración de red a todos los proxies. Dapr solo ofrece APIs de aplicación que la app invoca explícitamente, así que su plano de control se limita a inyectar el sidecar, repartir los `Component` y emitir certificados.

Nota: el README menciona `dapr-sentinels`, pero en el clúster el componente real es `dapr-sentry` (autoridad certificadora para mTLS). Además existe `dapr-scheduler`.

### 3. Flujo de inyección del sidecar

Al hacer `kubectl apply` de un Deployment, este crea un ReplicaSet que intenta crear un Pod con un solo contenedor (la app). Antes de guardarlo, el API server, en su etapa de admission, envía el Pod al `dapr-sidecar-injector` según el `MutatingWebhookConfiguration`. El injector revisa si tiene la anotación `dapr.io/enabled: "true"` y, de ser así, responde con un parche que añade el contenedor `daprd`. El API server aplica el parche y guarda en etcd el Pod con dos contenedores. Al arrancar, `daprd` se conecta al `operator` para cargar los `Component` y a `sentry` para obtener su certificado mTLS, y queda listo para comunicarse con la app por `localhost`.

---

## TODO 2: Exploración de los CRDs de Dapr

### 4. CRDs instalados por Dapr

Un CRD (Custom Resource Definition) enseña a Kubernetes un tipo de objeto nuevo, que luego se puede crear con `kubectl apply` como cualquier otro recurso. En este clúster (`kubectl get crds | grep dapr.io`) Dapr instaló:

| CRD | Para qué sirve |
|---|---|
| `components.dapr.io` | Declara qué tecnología concreta implementa cada Building Block (Redis para pub/sub, PostgreSQL para estado, etc.). |
| `subscriptions.dapr.io` | Define suscripciones a tópicos de forma declarativa, como alternativa al endpoint `/dapr/subscribe` del código. |
| `configurations.dapr.io` | Configura el comportamiento del runtime: tracing, mTLS, APIs habilitadas, middleware. |
| `resiliencies.dapr.io` | Define políticas de reintentos, timeouts y circuit breakers para las llamadas. |
| `httpendpoints.dapr.io` | Declara endpoints HTTP externos (que no usan Dapr) para invocarlos con las mismas reglas. |
| `workflowaccesspolicies.dapr.io` | Controla qué app IDs pueden operar sobre qué workflows y programar qué actividades en una app destino. |
| `mcpservers.dapr.io` | Declara la conexión a un servidor MCP (Model Context Protocol) para ejecución durable de herramientas. |

### 5. Relación entre `components.dapr.io` y el concepto de Building Block

El **Building Block** es la capacidad abstracta y su contrato (por ejemplo, Pub/Sub: publicar en un tópico y suscribirse), que es lo que la aplicación consume por la API del sidecar y que no cambia al cambiar de tecnología.

El **Component** es la **implementación concreta**, no la abstracción. Es la forma de declarar qué tecnología respalda a cada Building Block. Esto se ve en `spec.type: pubsub.redis`: `pubsub` nombra el Building Block y `redis` la implementación. Es análogo a una interfaz y sus clases concretas: la app programa contra el contrato y el YAML del `Component` decide qué implementación se inyecta. Migrar de Redis a Kafka implica cambiar `pubsub.redis` por `pubsub.kafka` (y su metadata), no el código de la app.
