# 2. Inventory data ownership

## Status

Proposed

## Context

LOGISTPULSE separa sus capacidades operativas en bounded contexts independientes.

El contexto de Inventory necesita mantener una fuente autoritativa para conocer el inventario disponible de cada tienda y producto, incluyendo stock actual y demanda prevista.

Otros componentes, como el frontend de LOGISTPULSE, necesitan consultar esta información sin acceder directamente a la base de datos del contexto de Inventory.

Es necesario definir qué información pertenece a `inventory-api`, cómo debe exponerse y qué ocurre cuando su dependencia de persistencia falla.

## Decision

`inventory-api` será responsable del ownership de los datos de inventario.

El acceso a estos datos se realizará mediante su API y no mediante acceso directo desde otros componentes a `inventory_db`.

Las operaciones principales son:

- `GET /api/inventory/{store_id}` para consultar inventario.
- `POST /api/inventory/{store_id}/{sku}/adjust` para modificar stock.

El frontend accederá al servicio mediante Nginx como borde de entrada.

## Data ownership

Pertenecen al bounded context de Inventory los datos almacenados en la tabla `inventory`:

- `store_id`
- `sku`
- `item_name`
- `unit`
- `stock`
- `forecast_4h`

La combinación `(store_id, sku)` identifica un producto dentro de una tienda.

La fuente autoritativa observada para estos datos es PostgreSQL, en la base `inventory_db`.

El nivel de riesgo mostrado al usuario (`HIGH`, `MEDIUM`, `LOW`) no se almacena como dato canónico. Es calculado por `inventory-api` a partir de `stock` y `forecast_4h`.

No pertenecen a Inventory los datos propios de Logistics, Operations o Fulfillment.

## Alternatives considered

1. Permitir que otros servicios consulten o actualicen directamente `inventory_db`.

   Se descarta porque rompe el ownership del bounded context y acopla otros componentes al esquema interno de Inventory.

2. Compartir las tablas de inventario entre varios servicios.

   Se descarta porque dificulta establecer una autoridad única sobre el stock y aumenta el acoplamiento entre contextos.

3. Exponer el inventario únicamente mediante la API de `inventory-api`.

   Se selecciona porque conserva un límite claro de responsabilidad y permite evolucionar la persistencia sin exponer su esquema interno.

## Consequences

### Positive

- Existe ownership claro sobre los datos de inventario.
- Otros componentes no necesitan conocer el esquema de PostgreSQL.
- Las actualizaciones de stock pasan por una interfaz controlada.
- El cálculo de riesgo permanece dentro del contexto que conoce stock y demanda.
- Se reduce el acoplamiento entre bounded contexts.

### Negative

- Los consumidores dependen de la disponibilidad de `inventory-api`.
- `inventory-api` depende directamente de PostgreSQL para consultas y ajustes.
- Se introduce un salto adicional a través de Nginx y la API.
- La disponibilidad técnica del proceso no garantiza actualmente que PostgreSQL esté disponible.

## Reliability and failure modes

`inventory-api` depende de PostgreSQL mediante `inventory_db`.

Si PostgreSQL no está disponible:

- las consultas de inventario pueden fallar;
- los ajustes de stock pueden fallar;
- el servicio no puede cumplir su capacidad principal.

El endpoint actual:

`GET /health`

devuelve `UP` sin comprobar una conexión real con PostgreSQL.

Por lo tanto, es posible que el servicio reporte salud técnica mientras una operación real de inventario no funciona. Este comportamiento representa un riesgo de falso verde y debe considerarse en la evolución de los health checks y la observabilidad.

En la implementación revisada no se observa una dependencia activa de Inventory con MQTT, Redpanda o MongoDB.

## Observability evidence

`inventory-api` instrumenta las solicitudes HTTP mediante Prometheus.

Métricas observadas:

- `logistpulse_http_requests_total`
- `logistpulse_http_request_duration_seconds`

El endpoint:

`GET /metrics`

expone las métricas para Prometheus.

También existe:

`GET /health`

como contrato de salud técnica.

El smoke test de LOGISTPULSE verifica que Inventory sea alcanzable mediante `/health/inventory`.

Para demostrar completamente la salud del bounded context será necesario complementar el health técnico con evidencia de que operaciones reales contra PostgreSQL continúan funcionando.

## Security considerations

El frontend accede a Inventory a través de Nginx, que constituye un límite de entrada hacia el servicio.

La operación:

`POST /api/inventory/{store_id}/{sku}/adjust`

modifica stock y, en la implementación revisada, no muestra controles explícitos de autenticación o autorización.

Por lo tanto, una evolución del sistema debería limitar quién puede ejecutar ajustes de inventario y validar el origen de estas solicitudes.

Otros servicios no deben recibir credenciales ni acceso directo a `inventory_db`.