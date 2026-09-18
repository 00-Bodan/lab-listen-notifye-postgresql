# Experimento mínimo: llenar LISTEN/NOTIFY

Requisito: Docker con Compose. Ejecuta los comandos desde esta carpeta.
Usamos PostgreSQL 17 con una cola de 64 páginas (512 KiB con páginas de 8 KiB).
No publica puertos ni necesita Python.

Validado con PostgreSQL 17.11: uso inicial `0`, cola llena `1` con errores
`too many notifications in the NOTIFY queue`, y uso `0` tras el `COMMIT` del
listener. El siguiente envío funcionó.

## 1. Arrancar

```sh
docker compose up -d --wait
```

## 2. Terminal A: retener la cola

```sh
docker compose exec postgres psql -X -U postgres
```

Ejecuta por separado y deja esta sesión abierta:

```sql
LISTEN demo;
BEGIN;
```

`LISTEN` queda confirmado antes de abrir la transacción. Mientras esta siga
abierta, el listener no puede avanzar y retiene las notificaciones pendientes.

## 3. Terminal B: llenar la cola

```sh
docker compose exec postgres psql -X -U postgres
```

```sql
SHOW max_notify_queue_pages;
SELECT pg_notification_queue_usage() AS uso_inicial;

-- Cada sentencia generada se confirma por separado (autocommit).
-- Continuamos tras los errores esperados para poder medir la ocupación.
\set ON_ERROR_STOP off
SELECT format('SELECT pg_notify(%L, %L);', 'demo', repeat('x', 7000) || g)
FROM generate_series(1, 100) g
\gexec

SELECT round((100 * pg_notification_queue_usage())::numeric, 2) AS porcentaje;
```

Espera errores `too many notifications in the NOTIFY queue` y una ocupación
alta. No tiene que marcar exactamente 100%: la asignación se realiza por páginas
y segmentos. No envuelvas el envío en un único `BEGIN`: queremos conservar los
envíos confirmados antes del primer fallo.

Opcional, desde otra terminal puedes ver los segmentos que respaldan la cola:

```sh
docker compose exec postgres sh -c 'ls -lh "$PGDATA/pg_notify"'
```

El tamaño de esos archivos no equivale exactamente a los bytes de mensajes
pendientes; hay caché y asignación por segmentos.

## 4. Liberar y comprobar recuperación

En la **terminal A**:

```sql
COMMIT;
```

`psql` imprimirá los mensajes pendientes, incluidos sus payloads grandes.
Espera a que vuelva el prompt. En la **terminal B**:

```sql
SELECT pg_notification_queue_usage() AS uso_despues;
SELECT pg_notify('demo', 'vuelve a funcionar');
```

La ocupación debe caer y el envío debe funcionar otra vez. Si la medición aún
no baja, repítela unos instantes después: la limpieza no es instantánea.

## 5. Limpiar

Sal de ambas sesiones con `\q` y elimina el contenedor y sus datos de prueba:

```sh
docker compose down -v
```

## Qué demuestra

Un listener dentro de una transacción puede retener la cola; cuando se llena,
las transacciones que envían notificaciones fallan al confirmar. Al terminar
la transacción del listener, la cola puede liberarse y los envíos recuperarse.

La cola es global para la instancia, aunque las notificaciones se entregan
dentro de cada base de datos. Esta prueba usa una sola base: no demuestra por
sí sola el efecto entre bases ni el antiguo cálculo del límite por wraparound.
Tampoco es un límite de RAM: es una cola respaldada por `pg_notify/` con caché
SLRU. Un cliente desconectado no retiene la cola ni recibe mensajes para
reproducirlos al reconectarse.

En PostgreSQL 17 el límite predeterminado sigue siendo 8 GiB con páginas de
8 KiB; ahora se configura mediante `max_notify_queue_pages`.

Referencias oficiales: [NOTIFY](https://www.postgresql.org/docs/17/sql-notify.html),
[max_notify_queue_pages](https://www.postgresql.org/docs/17/runtime-config-resource.html#GUC-MAX-NOTIFY-QUEUE-PAGES),
[implementación de la cola](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/commands/async.c).
