# analisis-abstraccion.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 2.

## TODO 2: Análisis de la Abstracción

### 1. Migración de broker: ¿qué archivos cambiarían?

Solo el Component YAML (`pubsub-redis.yaml` o un `pubsub-*.yaml` nuevo). Ahí cambia `spec.type`, de `pubsub.redis` a `pubsub.kafka` u otro, y las entradas de `spec.metadata` por las del nuevo broker. El `metadata.name: order-pubsub` se queda igual.

El código Python no cambia: publisher y subscriber solo usan `localhost:3500` y el nombre `order-pubsub`, y no hay ningún SDK de Redis en `requirements.txt`. Después de aplicar el YAML, el sidecar lo recarga solo (hot reload); si esa función estuviera desactivada, bastaría con un rollout restart de los pods.

### 2. Inventario de 5 brokers soportados por Dapr

| Broker | `spec.type` en el YAML |
|---|---|
| Redis | `pubsub.redis` |
| Apache Kafka | `pubsub.kafka` |
| RabbitMQ | `pubsub.rabbitmq` |
| Azure Service Bus Topics | `pubsub.azure.servicebus.topics` |
| AWS SNS/SQS | `pubsub.aws.snssqs` |

En todos cambia `spec.type` y, además, la metadata de conexión propia de cada broker (`brokers` en Kafka, `connectionString` en RabbitMQ y Azure Service Bus, `accessKey` en AWS SNS/SQS en lugar de `redisHost`).

### 3. Manejo seguro de secretos en producción

Con el building block de Secrets. En vez de escribir la contraseña en el YAML, la entrada de `spec.metadata` apunta a un secreto con `secretKeyRef` (campos `name` y `key`), y el valor real vive en un Secret Store: en Kubernetes un `Secret`, o también Vault o AWS Secrets Manager.

```yaml
spec:
  metadata:
    - name: redisPassword
      secretKeyRef:
        name: redis-secret
        key: redis-password
auth:
  secretStore: kubernetes
```

`auth` va al mismo nivel que `spec`. Con Kubernetes `secretStore` se puede omitir porque es el valor por defecto. Así el YAML puede estar en Git sin exponer nada, y el secreto se rota o se protege con RBAC sin tocar el Component.
