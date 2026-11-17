# Estado — Piscinas Vazquez

- **Rubro**: Piscinas de hormigón armado a medida (Villa Carlos Paz, Córdoba)
- **En producción**: sí · embudo de Kommo `14468572` · rubro en el sistema: `piscinas`
- **Reporte diario por mail**: 10:00, lo manda el flujo compartido "Kuro - WhatsApp Auditor - Reporte Diario".

## Links públicos de los tableros
| Tablero | Link |
|---|---|
| Demanda | https://jadecrepe1761.grafana.net/public-dashboards/a7159e443ec644aabf3bb1392f0db8f7 |
| Gestión y Conversión | https://jadecrepe1761.grafana.net/public-dashboards/f9f8bf4cee6d48e6b0e6aae4daffb9ac |

## Historia
Repo creado el 17/11/2026 al separar los clientes. Lo que sigue es copia de la documentación del repo compartido
(`05_README_INSTALACION.md`) en lo que toca a este cliente. Las secciones generales (que afectan a todos) siguen
allá.

## 18 bis. Clientes de otros rubros (multi-vertical, 24/09)

Hasta el 24/09 el unico cliente en produccion era Caliu (hotel) y la
extraccion de IA estaba cableada a hoteleria. Con el alta de **Piscinas
Vazquez** (venta e instalacion de piscinas) y **Universo Liv** el sistema
pasa a ser multi-rubro. Migracion: `10_migracion_multi_vertical.sql`.

### Que cambio

- `wa_hotels.vertical` — que rubro es el cliente (`hotel`, `piscinas`,
  `generico`, ...).
- `wa_hotels.vertical_config` (`jsonb`) — **la definicion del rubro es
  configuracion, no codigo**: el rol del auditor, las etiquetas del
  transcript, las reglas extra y la lista de campos que la IA tiene que
  extraer (`fields[]`, cada uno `{key, type, desc}`).
- `wa_conversation_audits.vertical` + `vertical_data` (`jsonb`) — donde
  se guardan los campos del rubro. Se eligio una columna `jsonb` por
  vertical en vez de columnas nuevas por rubro, para no ensanchar la
  tabla indefinidamente (ver `CLAUDE.md`).

En el Workflow B cambiaron 7 nodos: los 4 que llevan el rubro resuelto
dentro del estado (`Obtener talks monitorizados`, `Preparar cola de
talks`, `Buscar hotel por pipeline`, `Aplicar resolucion hotel`), el que
arma el prompt (`Construir contexto IA`), el que valida y mapea la
respuesta (`Validar JSON de auditoria`) y el `insert` (`Insertar
auditoria`, ahora con 49 parametros en vez de 47).

En el **Reporte Diario** (`07_...json`) cambiaron 2 nodos: la query de
clientes ahora trae `vertical`, y la tarjeta de KPI "Reservas confirmadas"
dice "Ventas cerradas" para los clientes que no son hoteles. El resto del
mail ya usaba `hotel_name`, asi que no cambia.

### Compatibilidad con los clientes hoteleros

Un cliente sin `vertical_config` (o con `vertical='hotel'`) usa el prompt
historico de hoteleria **sin ningun cambio** y sigue escribiendo en sus
columnas propias; `vertical_data` queda null. Se verifico generando el
prompt con las dos versiones del nodo y comparandolas: el system prompt y
el transcript de un cliente hotelero salen **byte a byte identicos** a
los de antes, asi que las auditorias de Caliu siguen siendo comparables
con las anteriores.

### Dar de alta un cliente de un rubro nuevo

1. `insert into wa_hotels (...)` con `vertical` y `vertical_config`
   (ver los dos ejemplos completos en `10_migracion_multi_vertical.sql`,
   secciones 10.4 y 10.5). Esto solo ya hace que la IA le pida a OpenAI
   los campos de ese rubro — no hay que tocar ningun workflow.
2. Confirmar el `pipeline_id` real del embudo de Kommo de ese cliente
   (ver seccion 10.6 de la migracion). Es el unico dato que no se puede
   deducir desde la base.
3. Clonar los dashboards: Gestion se copia tal cual (no depende del
   rubro); Demanda hay que armarla sobre las claves de `vertical_data`
   de ese rubro. Ver `grafana/README.md`.
4. Publicarlos (Share → Share externally, **lo hace el usuario**) y
   cargar las URLs + `report_email` en `wa_hotels` para que entre en el
   mail diario de las 10:00.

### Ojo: `wa_config.HOTEL_ID` esta desactualizado

`wa_config.config->>'HOTEL_ID'` sigue diciendo `piscinas-velasquez`, que
fue el **hotel de prueba** dado de baja hace tiempo — no tiene nada que
ver con el cliente real "Piscinas Vazquez". No afecta nada: el `owner`
del circuit breaker es `wa_runtime.owner = 'zaikoagencia'`
(= `KOMMO_SUBDOMAIN`), no `HOTEL_ID`. La seccion 8 de este README decia
que `owner` tenia que coincidir con `HOTEL_ID`: es incorrecto, quedo de
una version anterior.

## 11. Multi-hotel en un mismo Kommo (varios números de WhatsApp)

Desde esta versión, un único Kommo puede auditar varios números de
WhatsApp al mismo tiempo, siempre que **cada número conectado tenga su
propio embudo (pipeline) dedicado** en Kommo. Esto se confirmó
probándolo en vivo: un mensaje enviado desde un número de prueba cayó
inmediatamente en el embudo que se le había asignado a ese número.

### 11.1 Tabla `wa_hotels`

Nueva tabla (ver `04_schema_whatsapp_auditoria.sql`, última sección):

```sql
create table if not exists public.wa_hotels (
    hotel_id text primary key,
    hotel_name text,
    pipeline_id bigint not null unique,
    production_enabled boolean not null default false,
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now()
);
```

- `pipeline_id`: el embudo de Kommo dedicado a ese número de WhatsApp.
- `production_enabled`: interruptor por hotel. Mientras esté en
  `false`, las conversaciones de ese hotel se detectan y quedan
  guardadas en `wa_talks` pero **Workflow B no las audita** (no llama a
  Kommo ni a la IA para ellas). Se activa cuando el hotel esté listo
  para pasar a producción.

### 11.2 Cómo se resuelve el hotel de cada conversación

1. **Workflow A** (webhook, tiempo real) ya no asume un `hotel_id`
   fijo: al recibir actividad de un talk nuevo, lo guarda en
   `wa_talks` con `hotel_id = null`.
2. **Workflow B** (sync cada 12h), para cada talk con `hotel_id` nulo:
   1. Llama a `GET /api/v4/leads/{entity_id}` (vía el sub-workflow "GET
      Controlado", que ya respeta el circuit breaker / rate limit).
   2. Lee el `pipeline_id` del lead.
   3. Busca en `wa_hotels` qué hotel tiene ese `pipeline_id`.
   4. Si encuentra un hotel: guarda `hotel_id` en `wa_talks` y sigue
      auditando normalmente (si `production_enabled = true`).
   5. Si **no** encuentra ningún hotel con ese pipeline: marca el talk
      como `status = 'dormant'` con `last_error =
      'NO_MATCHING_HOTEL_PIPELINE'` y lo ignora — tal como pidió el
      cliente ("las conversaciones que caigan en pipelines que no son
      de ningún hotel, ignorarlas"). No se vuelve a intentar resolver
      ese talk salvo que se lo reactive a mano.
   - Una vez que un talk ya tiene `hotel_id` resuelto, no se repite la
     llamada a `GET /leads` en las corridas siguientes.

### 11.3 Cómo agregar un hotel/número nuevo

1. En Kommo, crear (o verificar) el embudo dedicado al número de
   WhatsApp de ese hotel y anotar su `pipeline_id`.
2. Insertar el hotel (con `production_enabled = false` para probarlo
   primero):
   ```sql
   insert into wa_hotels (hotel_id, hotel_name, pipeline_id, production_enabled)
   values ('nombre-hotel', 'Nombre del Hotel', 12345678, false);
   ```
3. Conectar el número de WhatsApp de ese hotel en Kommo (QR) y
   verificar que sus conversaciones caen en el embudo correcto.
4. Esperar a que corra Workflow B (o ejecutarlo a mano) y revisar en
   `wa_talks` que los talks de ese número quedaron con el `hotel_id`
   correcto y `status = 'monitoring'` (no `dormant`).
5. Cuando esté todo validado, activar la auditoría real:
   ```sql
   update wa_hotels set production_enabled = true where hotel_id = 'nombre-hotel';
   ```

### 11.4 Estado actual (Piscinas Vazquez)

Cuenta Kommo `zaikoagencia` (account id `31476147`) con dos embudos
registrados en `wa_hotels`:

- `PISCINAS VAZQUEZ` (número real del hotel).
- `Numero de pruebas` (número de prueba, usado para validar el
  mecanismo de resolución por pipeline).

Ambos quedaron con `production_enabled = false` — falta activarlos a
mano (paso 11.3.5) cuando se confirme que la resolución por pipeline
está funcionando bien en las corridas reales y se quiera empezar a
auditar en serio.

### 11.5 Pendiente / no incluido en este cambio

- `03_Kuro_WhatsApp_Bootstrap_Talks.json` (carga inicial de
  conversaciones ya existentes) **no** fue actualizado todavía para
  multi-hotel — sigue asumiendo un único hotel (escribe
  `wa_talks.hotel_id = wa_config.HOTEL_ID` directo, sin resolver por
  pipeline). Si se necesita bootstrapear conversaciones históricas de
  un hotel nuevo, hay que adaptarlo primero (mismo patrón de
  resolución por pipeline que Workflow B). **Sí** se corrigió el bug
  de `owner` que esto traía: mandaba `owner = HOTEL_ID` mientras que
  el circuit breaker (`wa_runtime`) ya es por cuenta de Kommo
  (`KOMMO_SUBDOMAIN`, ver sección 8) — con el owner viejo, cualquier
  corrida de Bootstrap fallaba siempre con `BATCH_OWNER_MISMATCH`.
  Ya manda `owner = KOMMO_SUBDOMAIN` como el resto.
- Puede haber filas viejas en `wa_talks` (de antes de este cambio) con
  un `hotel_id` asignado incorrectamente por la lógica anterior.
  Conviene revisarlas antes de activar `production_enabled` en algún
  hotel: `select * from wa_talks where hotel_id is not null and
  updated_at < '<fecha de este cambio>';`
