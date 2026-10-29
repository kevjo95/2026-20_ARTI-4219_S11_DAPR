# reflexion-arquitectonica.md
# Completa este archivo con tus respuestas al Reto 3 de la Fase 4.

## Reto 3: Reflexión Arquitectónica

### 1. Sin Dapr: ¿dónde vive el código de TTL y consumer group?

Con el SDK de Kafka directo, el consumer group es el `group.id` en la configuración del consumer, o sea dentro de cada aplicación, y lo mantiene el equipo de la app. El TTL se reparte: la expiración por tópico es `retention.ms`, que se configura en el tópico y es del equipo de plataforma; un TTL por mensaje habría que programarlo en la app (el publisher pone una marca de expiración y el consumer descarta lo vencido).

### 2. El patrón sidecar como frontera de responsabilidad

Producto decide el SLA (las órdenes valen 2 minutos) y plataforma da el mecanismo y lo opera, sin que cada app lo reimplemente. Pero en la prueba apareció un matiz: `ttlInSeconds` se fija por mensaje en la publicación, y con el valor solo en el Component (120) los mensajes llegaron igual 150 s después. Lo confirmé con un control: sin el TTL en la publicación el sobre no lleva `expiration`, y con `?metadata.ttlInSeconds=10` sí. Al mover el TTL al publisher (`?metadata.ttlInSeconds=120`) y repetir la prueba, el subscriber no recibió nada y el sidecar registró `dropping expired pub/sub event`. Si el valor vive en el código del publisher, el equipo de app queda dueño de una regla de negocio, así que hay que acordar quién lo fija y verificarlo con una prueba como esta.

### 3. Limitaciones de la abstracción: ¿migración 100% transparente?

No. El soporte de TTL cambia por broker: Azure Service Bus lo soporta de forma nativa, en Kafka es por tópico (`retention.ms`) y en otros Dapr lo emula en el runtime. Los campos del consumer group también difieren: Kafka tiene `consumerGroup` y `consumerID`, Redis solo `consumerID`, y la parte de namespaces en Redis se nota en cómo se nombra todo: con `{namespace}` el stream pasó a llamarse `defaultorders` y el grupo `order-subscriber-default.order-subscriber`, algo que en otro broker se configuraría y se nombraría de otra manera. El código no cambia al migrar, pero la configuración y el comportamiento sí hay que revisarlos y probarlos.
