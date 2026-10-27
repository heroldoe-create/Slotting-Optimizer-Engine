# Brief final de construcción

## Slotting Optimizer Engine

**Proyecto:** `slotting-optimizer-engine`
**Versión del brief:** 1.2 — FINAL / READY TO USE / READY TO BUILD / READY TO FREEZE CONTRACTS
**Estado documental:** especificación final lista para construir y congelar contratos. Las correcciones están incorporadas documentalmente; su verificación operativa exige ejecutar las pruebas. No afirma que el microservicio o las mejoras ya existan.
**Objetivo:** microservicio analítico que lea movimientos históricos de PostgreSQL/Supabase, clasifique SKUs mediante ABC/XYZ y proponga reubicaciones factibles hacia Zona Dorada para reducir desplazamiento de picking y mejorar LPPH con evidencia.
**Alcance de referencia:** producto completo en Fases 1–3; MVP publicable mediante G-portafolio. La Fase 4 queda como roadmap y no forma parte de ninguno de los dos alcances.

---

## 1. Resumen ejecutivo

Construir un microservicio analítico que identifique qué SKUs conviene reubicar, desde qué ubicación y hacia qué pick-face, considerando rotación, variabilidad, distancia, compatibilidad física, capacidad y costo de cambio.

El sistema tendrá tres salidas coordinadas:

1. **Motor headless:** análisis reproducible por CLI, con dataset, configuración y distancias versionadas.
2. **Microservicio:** API y worker para ejecución bajo demanda o programada contra PostgreSQL/Supabase.
3. **Reporte de decisiones:** propuestas priorizadas, beneficio estimado, restricciones, aprobación y medición posterior.

La pregunta principal no será “¿qué SKU es clase A?”, sino:

> “¿Qué movimiento SKU–ubicación tiene beneficio neto positivo, cabe físicamente, no rompe restricciones y ofrece la mejor evidencia de ahorro dentro de los supuestos declarados?”

**Principio rector:**

> El motor propone; una persona autorizada decide. El sistema nunca modifica inventario ni ubicaciones transaccionales por sí mismo.

### 1.1 Autoridad normativa y entradas de agentes

La precedencia es:

1. Este brief y sus decisiones normativas son la fuente canónica de negocio, fórmulas y aceptación.
2. Los contratos ejecutables derivan del brief y lo hacen validable; no pueden relajar sus reglas.
3. La configuración selecciona valores permitidos; no redefine semántica ni prevalece sobre contratos.
4. Las pruebas verifican la especificación; no introducen reglas distintas.

Ante divergencia se detiene la publicación de resultados afectados, se aplica transitoriamente la restricción más conservadora y se registra un cambio versionado del brief. Ningún archivo secundario se corrige en silencio para “resolver” el conflicto.

`AGENTS.md`, `CLAUDE.md` y `docs/INTAKE.md` son entradas obligatorias del repositorio para Codex/Claude. Contienen enlaces y un resumen de navegación hacia `docs/specification.md`, contratos y pruebas, sin copias independientes de fórmulas. `T-INSTALL-001` verifica su existencia, enlaces y paridad de referencias; la carga efectiva en cada entorno se registra como evidencia, no se presume por la existencia de los archivos.

---

## 2. Resultado de negocio esperado

La herramienta deberá convertir una recomendación genérica como “acerque los productos que más salen” en una decisión trazable.

El siguiente texto es únicamente un **ejemplo hipotético de presentación**, no un resultado anticipado:

> “El análisis identificó doce reubicaciones físicamente compatibles con beneficio neto estimado positivo. Las tres primeras concentran la mayor parte del ahorro de viaje calculado. Se recomienda ejecutarlas en una ventana controlada y medir LPPH, distancia y adopción antes de ampliar el cambio.”

Toda conclusión deberá citar `analysis_run_id`, ventana, corte de datos, configuración, versiones de maestro/layout/matriz, fórmulas y recomendaciones aplicadas. El beneficio antes de implementar es una **estimación de modelo**; el beneficio realizado solo se reporta después de medir la operación y separar, cuando sea posible, demanda, dotación y otros cambios concurrentes.

---

## 3. Usuarios y casos de uso

### Usuarios objetivo

- Jefes y supervisores de CEDIS.
- Analistas de mejora continua.
- Equipos de inventarios y slotting.
- Consultores logísticos freelance.
- Implementadores WMS/Supabase.

### Flujo principal

1. Conectar una fuente de solo lectura.
2. Validar movimientos, maestros, inventario y distancias.
3. Configurar ventana, reglas ABC/XYZ y restricciones.
4. Ejecutar análisis y generar candidatos.
5. Revisar propuestas y motivos de exclusión.
6. Aprobar, rechazar o posponer movimientos.
7. Exportar una orden de trabajo para ejecución externa.
8. Registrar aplicación externa y verificar resultados.

La interfaz y los reportes estarán en **español**. Código, estados, identificadores y contratos técnicos estarán en **inglés**.

---

## 4. Alcance del producto completo y extensiones

Esta sección describe el alcance de referencia. La salida mínima prioritaria es G-portafolio, definida en la sección 25.

### Incluido en el producto completo

- Conector PostgreSQL/Supabase de solo lectura.
- Importación incremental y corte consistente de datos.
- Clasificación ABC/XYZ reproducible.
- Validación de compatibilidad SKU–slot.
- Matriz de distancias o grafo navegable versionado.
- Optimizador determinista de asignación `greedy_v0`.
- Beneficio bruto, costo de cambio y beneficio neto.
- Flujo de aprobación y exportación de trabajo.
- Backtesting temporal sin fuga de datos.
- API, worker, reporte HTML y dashboard ligero.
- Pruebas, benchmark, evidencias y demo.

### Excluido

- Escritura automática en el WMS/ERP o en tablas transaccionales.
- Movimiento físico de producto o confirmación automática de ejecución.
- Inventario por lote, serie, caducidad o FEFO/FIFO.
- Asignación de múltiples pick-faces por SKU en una corrida.
- Optimización matemática global exacta.
- Slotting multisede o consolidación de pedidos.
- Ruteo completo de tours de picking.
- Gemelo digital 3D.
- Predicción de demanda o calibración causal automática.
- Recomendaciones para SKUs con datos físicos o capacidad desconocidos.

Estas exclusiones no autorizan aproximaciones silenciosas. Cuando falte una entrada necesaria, el SKU o la ubicación quedará fuera del universo optimizable con causa.

---

## 5. Decisiones cerradas del modelo operativo

### 5.1 Unidad de análisis y movimiento

- La unidad de clasificación es el **SKU** dentro de un `warehouse_id`.
- La unidad de recomendación es `(sku_id, from_location_id, to_location_id)`.
- `from_location_id` proviene exclusivamente de `sku_primary_pick_face`; nunca se infiere por mayor stock, último movimiento o ubicación más cercana.
- Cada SKU tiene como máximo un pick-face primario vigente. Cero primarios produce `NO_PRIMARY_PICK_FACE`; múltiples vigentes o solapados produce `MULTIPLE_PRIMARY_PICK_FACES` y calidad crítica.
- En una corrida, un SKU puede aparecer en una sola recomendación `PROPOSED` y un slot puede asignarse como máximo una vez.
- El MVP no consolida reserva, no divide stock y no optimiza pick-faces secundarios. La reubicación afecta solo el stock y los hits del pick-face primario actual.
- ABC/XYZ usa la demanda total del SKU; el beneficio usa únicamente la demanda atribuible al origen reubicado. Ningún hit de otra ubicación se traslada implícitamente al destino.
- La recomendación se realiza únicamente mediante ejecución externa evidenciada; generarla no cambia el layout operativo.

### 5.2 Zona Dorada y elegibilidad de slots

La Zona Dorada es un conjunto explícito de ubicaciones con `is_golden=true`, no una etiqueta inferida por cercanía.

Un slot destino debe satisfacer simultáneamente:

- `is_active=true`, `is_blocked=false` y `slot_type=PICK_FACE`.
- `is_golden=true` cuando la política lo exija.
- Temperatura, peligrosidad y categoría compatibles.
- Dimensiones unitarias, peso, volumen y `authorized_capacity_units` suficientes bajo una regla de capacidad versionada.
- Ausencia de reserva, dedicación o bloqueo.
- Distancias de ida y vuelta válidas hacia/desde el anchor y entre origen/destino para el traslado.
- Acceso conocido en origen y destino.
- Evidencia de desocupación según §5.5. Estar ocupado produce `OCCUPIED_SLOT` u `OCCUPIED_BY_SAME_SKU`; evidencia insuficiente produce `MISSING_DESTINATION_OCCUPANCY_EVIDENCE`.
- Ser distinto del origen; una coincidencia produce `SOURCE_EQUALS_DESTINATION`.

No se crean pick-faces secundarios ni consolidaciones encubiertas. La cercanía no demuestra ergonomía ni ahorro.

### 5.3 Demanda, hits, líneas y reversos

Solo `PICK_CONFIRMED` genera actividad original de picking. `order_line_id` pertenece a un único `(warehouse_id, sku_id)`; una violación produce `ORDER_LINE_SKU_MISMATCH`.

```text
hit_id = (warehouse_id, sku_id, pick_task_id, location_id)

net_movement_units = original_quantity
                     - Σ reversal_quantity disponible al knowledge_cutoff

net_hit_units = Σ net_movement_units del mismo hit_id
net_line_units = Σ net_movement_units del mismo order_line_id
```

- La cantidad de reverso se normaliza como magnitud positiva enlazada por `reversal_of_id`. `net_movement_units<0` produce `REVERSAL_OVERDRAW`; no se toman valores absolutos para fabricar demanda.
- Un hit existe si `net_hit_units>0` y conserva al menos un movimiento original válido. Revertir totalmente un movimiento no elimina un hit compartido si otro movimiento del mismo hit mantiene neto positivo.
- ABC cuenta hits distintos; XYZ suma `net_movement_units` por semana del evento original. LPPH cuenta líneas completadas, no hits ni eventos.
- `order_line_completions` es la evidencia autoritativa de finalización. Una línea completada exige estado `COMPLETED`, `completed_at` en la cohorte/periodo, `net_line_units>0` y al menos un hit vigente. Actividad parcial sin finalización se reporta como `active_pick_lines`, nunca como `completed_order_lines`.
- Un reverso total de todas las actividades retira la línea completada; un reverso parcial la conserva si mantiene las condiciones anteriores.
- Los eventos originales se asignan a ventana por `moved_at`. Los reversos enlazados se aplican con política `AS_OF_RESTATEMENT` si `recorded_at<=knowledge_cutoff`, incluso cuando cruzan el límite de ventana; un reverso todavía no disponible no puede alterar el análisis.
- Recepciones, transferencias, ajustes, conteos y cancelaciones no generan demanda de picking.

### 5.4 Corte temporal, semanas ISO y conocimiento disponible

- La fuente almacena timestamps UTC. El escenario declara `analysis_timezone` IANA; default `UTC`.
- `data_cutoff_at` es límite exclusivo de datos y `knowledge_cutoff_at` límite de disponibilidad; en análisis actual coinciden salvo declaración explícita.
- Todo evento usa `moved_at` para tiempo efectivo y `recorded_at` para conocimiento. Maestros, parámetros y asignaciones usan vigencia más `recorded_at`/versión cuando el backtest lo exija.

```text
local_cutoff = data_cutoff_at convertido a analysis_timezone
window_end_local = lunes 00:00 de la semana ISO que contiene local_cutoff
window_start_local = window_end_local - 12 semanas de calendario local
window_start_utc / window_end_utc = límites locales convertidos con el offset vigente en cada límite
```

Si el corte es exactamente lunes 00:00 local, ese instante es el fin exclusivo; en cualquier otro caso se excluye la semana parcial. Los eventos aceptados cumplen `window_start_utc <= moved_at < window_end_utc` y `recorded_at <= knowledge_cutoff_at`.

- ABC usa los mismos días completos. Eventos tardíos se incorporan por lookback y nueva versión, sin editar corridas anteriores.
- Entrenamiento y prueba se separan por tiempo efectivo y conocimiento antes de clasificar.

### 5.5 Inventario, cobertura y capacidad requerida

`current_on_hand_units` es el stock del SKU en su pick-face primario, no el total de reserva/almacén. `target_stock_units` es el objetivo del pick-face.

La corrida selecciona **un único `snapshot_set_id` por almacén**, el más reciente que cumpla `snapshot_at<=data_cutoff_at`, antigüedad máxima y cobertura requerida; en empate, `snapshot_set_id` ascendente. No se combinan filas de capturas distintas.

- `snapshot_sets` declara `coverage_mode=COMPLETE|PARTIAL`, ubicaciones cubiertas, `absence_semantics` y completitud.
- Para el origen, dentro de una captura completa, ausencia significa cero solo si `absence_semantics=ZERO_WITHIN_COMPLETE_SET`; de lo contrario se exige fila cero.
- Para demostrar destino vacío se exige captura `COMPLETE` que cubra ese slot. Cualquier stock positivo lo ocupa. Si todas las filas son cero, o no hay filas y la ausencia dentro de cobertura completa significa cero, está vacío.
- Una captura parcial puede demostrar ocupación por stock positivo, pero nunca desocupación. Captura ausente, antigua o sin cobertura produce `MISSING_DESTINATION_OCCUPANCY_EVIDENCE`.

```text
required_capacity_units = max(coalesce(current_on_hand_units, 0),
                              coalesce(target_stock_units, 0))
```

- Si no hay base vigente ni target: `MISSING_CAPACITY_BASIS`. Si el resultado es cero: `ZERO_CAPACITY_BASIS`; no se propone mover aire.
- `capacity_basis_mode=TARGET_ONLY` no afirma stock real cero y exige snapshot de origen antes de aprobar trabajo.
- La factibilidad exige `required_capacity_units<=authorized_capacity_units`, dimensiones unitarias compatibles tras una orientación permitida y límites agregados de peso/volumen. El volumen total suficiente no compensa una dimensión incompatible.
- El optimizador no divide stock entre slots ni mueve reserva para compensar capacidad insuficiente.

### 5.6 Gobierno del cambio

```text
PROPOSED
APPROVED
REJECTED
DEFERRED
EXECUTION_STARTED
APPLIED_EXTERNALLY
VERIFIED
EXPIRED
CANCELLED
```

| Origen | Destino | Actor y condición |
| --- | --- | --- |
| `PROPOSED` | `APPROVED`, `REJECTED`, `DEFERRED` | Aprobador; propuesta vigente, snapshot revalidado y reserva transaccional obtenida para aprobar. |
| `PROPOSED` | `EXPIRED` | Sistema, por cambio material de §9. |
| `PROPOSED`, `APPROVED`, `DEFERRED` | `CANCELLED` | Aprobador/administrador antes del claim de ejecución; libera reserva. |
| `APPROVED` | `DEFERRED` | Aprobador si `execution_claim` no existe; libera reserva. |
| `APPROVED`, `DEFERRED` | `EXPIRED` | Sistema por cambio material o `valid_until`; libera reserva. |
| `DEFERRED` | `PROPOSED` | Solo si `valid_until` no venció, fingerprint coincide con el último run vigente, snapshot sigue válido y restricciones/umbrales permanecen `PASS`; si no, `EXPIRED`. |
| `APPROVED` | `EXECUTION_STARTED` | Ejecutor registra `execution_claim` con token, revisión y `expected_version`. |
| `EXECUTION_STARTED` | `APPLIED_EXTERNALLY` | Ejecutor evidencia finalización completa. Ejecución parcial permanece en `EXECUTION_STARTED`. |
| `EXECUTION_STARTED` | Histórico no reabrible (`is_current=false`) | Sistema, tras reconciliar resultados por SKU y crear la versión sucesora; deja de contar como activo para unicidad. |
| `APPLIED_EXTERNALLY` | `VERIFIED` | Verificador aporta ventana post y medición. |

`REJECTED`, `CANCELLED`, `EXPIRED` y `VERIFIED` son terminales para esa versión. `EXECUTION_STARTED` no puede cancelarse ni expirarse retroactivamente: una ejecución parcial/fallida exige reconciliación y nueva versión. Ningún estado salta de `PROPOSED` a `VERIFIED`.

Un cambio no material sobre `PROPOSED`, `APPROVED` o `DEFERRED` incrementa `recommendation_version` **conservando el estado** y enlazando el historial. Toda transición conserva actor, motivo, `command_id`, versión y timestamp.

### 5.7 Reservas y protocolo de ejecución externa

El schema propio mantiene `slot_reservations` con clave única activa por `(warehouse_id, to_location_id)` y otra por `(warehouse_id, sku_id)` para aprobaciones vigentes.

- Aprobar es una transacción: bloquear la recomendación, revalidar versión/factibilidad, comprobar ausencia de reserva activa, insertar reserva y cambiar estado. Un conflicto produce `SLOT_RESERVATION_CONFLICT`; solo una asignación compatible puede aprobarse.
- `DEFERRED`, `CANCELLED` y `EXPIRED` liberan la reserva antes de ejecución. `APPLIED_EXTERNALLY` la convierte en `OCCUPIED_PENDING_SNAPSHOT` y sigue bloqueando el slot hasta que una captura confirme ocupación; entonces se cierra. Así se evita reasignarlo con evidencia de vacío anterior. Nada escribe en WMS.
- Toda orden exportada lleva `work_order_token`, `work_order_revision`, `analysis_fingerprint` y `valid_until`. Cualquier cambio de estado o revisión invalida las exportaciones anteriores.
- `execution_claim` registra inicio externo, actor, token, revisión y timestamp. Claim y cancelación usan la misma transacción de versión: uno gana y el otro recibe `STATE_CONFLICT`.
- El progreso externo usa `NOT_STARTED|IN_PROGRESS|PARTIAL|COMPLETED|FAILED`. Solo `COMPLETED` permite `APPLIED_EXTERNALLY`; parcial o fallido queda visible y nunca se presenta como aplicación completa.
- La reconciliación parcial/fallida registra el resultado por SKU contra una captura posterior. Para cada recomendación no movida, la versión anterior queda en `EXECUTION_STARTED` con `is_current=false`, enlazada a la `(recommendation_id, recommendation_version)` sucesora y con su reserva liberada; la nueva versión conserva `recommendation_key` y `recommendation_id`, incrementa `recommendation_version` y nace en `PROPOSED`, sujeta a revalidación. Las recomendaciones movidas siguen la transición y reserva de `APPLIED_EXTERNALLY`; si una nueva propuesta las incluye, su nuevo `from_location_id` genera otra clave e ID. La versión histórica no se reabre ni cuenta como activa para unicidad.

---

## 6. Datos de entrada y contratos

### 6.1 Fuentes mínimas

| Fuente | Campos mínimos | Regla principal |
| --- | --- | --- |
| `inventory_movements` | `movement_id`, `warehouse_id`, `sku_id`, `location_id`, `movement_type`, `quantity`, `moved_at`, `recorded_at`, `pick_task_id`, `order_line_id`, `cohort_id`, `reversal_of_id` | Tiempo efectivo y de disponibilidad; reversos enlazados. |
| `order_line_completions` | `order_line_id`, `warehouse_id`, `sku_id`, `status`, `completed_at`, `recorded_at` | Autoridad de finalización; sin ella no se denomina línea completada. |
| `skus` | `sku_id`, `warehouse_id`, dimensiones, peso, `category`, `temperature_zone`, `hazard_class`, `target_stock_units`, vigencia y `recorded_at` | Un registro por versión y unidades declaradas. |
| `sku_primary_pick_face` | `warehouse_id`, `sku_id`, `location_id`, `valid_from`, `valid_to`, `assignment_reason`, `recorded_at` | Una sola asignación vigente por SKU al corte. |
| `locations` | `location_id`, `warehouse_id`, `slot_type`, `is_golden`, estado, dimensiones útiles, límites de peso/volumen, `compatibility`, `graph_node_id`, `layout_version_id`, `valid_from`, `valid_to`, `recorded_at` | Un registro por versión; solapamientos son error crítico. |
| `location_access_times` | `location_id`, `layout_version_id`, `access_time_s`, `access_time_source`, `valid_from`, `valid_to`, `recorded_at` | Fuente versionada de acceso; desconocido no se convierte en cero. |
| `location_capacity_rules` | `location_id`, `capacity_rule_version`, `authorized_capacity_units`, dimensiones máximas unitarias, orientaciones permitidas, límites agregados, vigencia | Modelo acotado de factibilidad, sin optimización 3D. |
| `snapshot_sets` | `snapshot_set_id`, `warehouse_id`, `snapshot_at`, `recorded_at`, `coverage_mode`, `covered_locations`, `absence_semantics`, `is_complete` | Captura atómica; no mezclar conjuntos. |
| `inventory_snapshots` | `snapshot_set_id`, `sku_id`, `location_id`, `on_hand_units` | Filas asociadas a una única captura y cobertura declarada. |
| `distance_matrix` o `layout_graph` | origen/destino o `edge_id`, distancia/peso en metros, `distance_mode`, `layout_version_id`, `valid_from`, `valid_to`, `recorded_at`, `is_directed`, `directionality_justification` | Distancia caminable versionada; dirigida cuando se declare. |
| `picker_time_summaries` | `warehouse_id`, `cohort_id`, `period_start`, `period_end`, `completed_order_lines`, `picker_busy_hours`, `travel_hours`, `recorded_at` | Autoridad de horas; conteo de líneas para reconciliación. |
| `classification_history` | `warehouse_id`, `classification_cutoff`, `sku_id`, clase ABC/XYZ, `config_hash`, `classification_run_id`, `recorded_at` | Observaciones temporales de estabilidad, no repeticiones de una corrida. |
| `slotting_state_snapshot` | `state_snapshot_id`, `as_of_at`, aplicaciones, cooldowns, recomendaciones/reservas activas y hashes | Estado operativo versionado usado por elegibilidad y reproducción. |

Los nombres pueden mapearse a vistas, pero el contrato canónico no cambia. Todo mapping se declara y versiona. Para análisis actual se usa la versión válida y disponible al corte. Backtesting completo exige reconstruir vigencia **y conocimiento** `as_of`; una fuente sin historia recibe la limitación correspondiente.

### 6.2 Contratos cerrados de compatibilidad, acceso y capacidad

```yaml
compatibility:
  allowed_temperature_zones: [AMBIENT]
  allowed_hazard_classes: [NONE]
  allowed_categories: [GENERAL]
```

Un SKU es compatible cuando sus tres atributos pertenecen a las listas. Lista ausente significa `MISSING_COMPATIBILITY`, no “acepta todo”.

`access_time_s>=0` y `access_time_source ∈ {MEASURED, CONFIGURED, SYNTHETIC, UNKNOWN}`. `access_time_measured=true` se deriva solo para `MEASURED` o `SYNTHETIC` golden. Con `require_measured_access_time=true`, cualquier `UNKNOWN`, nulo o `CONFIGURED` no medido produce `MISSING_ACCESS_TIME`. Cero exige fuente válida.

El modelo físico acotado comprueba, sin generar empaquetamiento 3D:

1. Existe una orientación permitida que hace caber cada dimensión unitaria en las dimensiones útiles y máximas declaradas.
2. Peso unitario, peso agregado y volumen agregado no exceden límites.
3. `required_capacity_units<=authorized_capacity_units` para la versión de regla vigente.

Si falta cualquiera de esas evidencias, no se afirma factibilidad. `authorized_capacity_units` es autoritativo solo con `capacity_rule_version` y método documentado.

### 6.3 Distancias, cohortes y autoridad de mapping

Para matriz simétrica, ida y vuelta usan la misma distancia. Para matriz/grafo dirigido se requieren ambas direcciones y `directionality_justification`. Si una dirección necesaria falta o es inalcanzable: `MISSING_DISTANCE`.

En `layout_graph`, la distancia es shortest path con Dijkstra o equivalente declarado, pesos `>0` y versión fijada. Empates de caminos se resuelven por secuencia lexicográfica de `edge_id`; el resultado incluye camino y hash. `synthetic_euclidean` exige `is_directed=false` y simetría.

`picker_time_summaries.period_end` es exclusivo. `source_mapping.yaml` es la **única autoridad** de `period_granularity: DAY|WEEK`; repetirlo en configuración con otro valor produce `CONFIG_CONFLICT`. Repetir `stability_lag_weeks` en `classification` y `backtest` con valores distintos produce `CONFIG_CONFLICT`; si coinciden, se usa ese valor único.

Las cohortes de un mismo almacén/periodo no solapan población. Varias cohortes se agregan sumando líneas y horas, nunca promediando LPPH. Cada hit lleva `cohort_id`; la proyección exige que las cohortes del plan y del baseline correspondan.

`order_line_completions` autoriza finalización y `picker_time_summaries` autoriza horas. `derived_completed_order_lines` se calcula desde completions/reversos y se reconcilia contra el conteo del summary. En golden sintético la diferencia es cero. En fuente real se exportan diferencia absoluta/porcentual y tolerancia; superarla produce `LPPH_SOURCE_UNRECONCILED`, advertencia y proyección `null`.

### 6.4 Ingesta incremental y consistencia

- Cuenta read-only, `statement_timeout` y `REPEATABLE READ READ ONLY` por corte.
- Extracción por `(moved_at, movement_id)` y `(recorded_at, movement_id)` con keyset; no `OFFSET`.
- Watermark con lookback; reprocesar es idempotente por clave y hash.
- Cada corrida registra cortes, watermark, conteos, snapshot set, estado operativo y hashes de mapping/configuración/layout/distancias/historia.
- Las fuentes se leen en vista coherente. Sin consistencia: `INVALID_SNAPSHOT`, sin recomendaciones.
- Escritura limitada al schema propio `slotting`.

### 6.5 Validación

Como mínimo:

- Esquema, tipos, unidades, enums, claves y configuración económica.
- Duplicados, vigencias solapadas y asignaciones primarias ausentes/múltiples.
- Timestamps efectivos/disponibles, cortes ISO, DST y eventos tardíos.
- Reversos huérfanos, múltiples, cíclicos, excedidos y compartidos por hit.
- Líneas multi-SKU, finalización ausente y reconciliación de summaries.
- Referencias inexistentes y cantidades/dimensiones negativas.
- Cobertura/completitud de snapshot sets y evidencia de ocupación.
- Acceso desconocido, capacidad/orientación no demostrable y compatibilidad ausente.
- Distancias ausentes, negativas, asimétricas sin justificación o caminos inalcanzables.
- Periodos/cohortes de picker solapados y población incompatible.
- Movimientos de un mismo `hit_id` con `cohort_id` distinto producen `COHORT_INCONSISTENCY_IN_HIT`; el hit se rechaza y no entra a clasificación ni proyección.
- Historial de clasificación/estado ausente o cortes de estabilidad repetidos.

Salidas: `accepted_records`, `rejected_records`, `warnings`, errores por campo y `data_quality_status=OK|WARNING|CRITICAL`. Un error crítico impide recomendaciones; una advertencia nunca se transforma en dato inventado.

### 6.6 Dataset sintético

Incluye generador reproducible con 100,000 movimientos para demo y 1,000,000 para benchmark, perfiles ABC/XYZ, clases vacías, semanas cero, Zona Dorada ocupada, compatibilidad/capacidad adversa, tareas multiubicación, varios movimientos por hit, líneas parciales/completas/revertidas, capturas completas/parciales/antiguas, accesos desconocidos, grafos dirigidos, historia de estabilidad y cohortes.

Casos temporales: domingo `23:59:59`, lunes `00:00:01`, DST y evento anterior recibido después del corte. La variante `America/Mexico_City` usa fixtures y configuración propios; `T-DATA-002` ejecuta ambos.

Archivos mínimos:

```text
movements_demo.csv
movements_demo_tz.csv
order_line_completions_demo.csv
skus_demo.csv
sku_primary_pick_face_demo.csv
locations_demo.csv
location_access_times_demo.csv
location_capacity_rules_demo.csv
snapshot_sets_demo.csv
inventory_demo.csv
distance_matrix_demo.csv
picker_time_summaries_demo.csv
classification_history_demo.csv
slotting_state_snapshot_demo.json
source_mapping.demo.yaml
analysis_config.demo.yaml
analysis_config.tz_demo.yaml
dataset_manifest.json
```

La cohorte golden contiene 12,000 líneas, 100 horas ocupadas y 30 de viaje agregadas sobre las mismas 12 semanas del beneficio. `T-INSTALL-001` y `T-EXPORT-001` verifican existencia, schema y reapertura.

---

## 7. Clasificación ABC/XYZ

### 7.1 ABC por frecuencia de salida

Para cada SKU `s` con al menos un hit en la ventana:

```text
hits_s = count_distinct((pick_task_id, location_id) válidos de s)
share_s = hits_s / H
H = Σ hits_s de todos los SKUs con demanda
```

Orden determinista: `hits_s` descendente, `units_picked_s` descendente y `sku_id` ascendente. Para cada posición se calcula `cum_before_s`, la participación acumulada de los SKUs anteriores.

```text
A si cum_before_s < abc_a_threshold
B si abc_a_threshold <= cum_before_s < abc_b_threshold
C en otro caso
```

Defaults: `abc_a_threshold=0.80` y `abc_b_threshold=0.95`. El SKU que cruza un umbral queda en la clase anterior, porque su `cum_before` todavía está por debajo del límite. Esta regla evita divergencias por redondeo.

Los SKUs con cero hits se marcan `NO_DEMAND`; no se fuerzan dentro de C ni reciben recomendación de alta rotación. Si `H=0`, toda la corrida queda `NO_DEMAND_WINDOW` y no hay ranking ABC. Si un solo SKU concentra toda la demanda, `cum_before=0` y ese SKU es A; B y C pueden quedar vacías y la matriz 3×3 muestra cero sin considerarlo un error.

### 7.2 XYZ por variabilidad semanal

Para cada SKU se construyen las 12 semanas ISO completas, **incluidas las semanas con cero unidades**:

```text
mean_s = promedio(units_picked semanales)
cv_s = sample_stddev(units_picked semanales) / mean_s
```

La desviación es muestral (`n-1`). Se requieren al menos dos semanas y `mean_s > 0`; con la ventana default se exigen las 12 para clasificación automática.

```text
X si cv_s <= xyz_x_threshold
Y si xyz_x_threshold < cv_s <= xyz_y_threshold
Z si cv_s > xyz_y_threshold
```

Defaults: `xyz_x_threshold=0.50` y `xyz_y_threshold=1.00`. `cv_s=0` es X. Si faltan semanas o la historia es insuficiente, el resultado es `INSUFFICIENT_HISTORY`, no una clase estimada. Un SKU sin demanda usa `NO_DEMAND`.

### 7.3 Matriz combinada, estabilidad y gobierno

La salida combina `AX`, `AY`, `AZ`, `BX`, `BY`, `BZ`, `CX`, `CY` y `CZ`, más estados no clasificables. La matriz informa política y estabilidad, pero no sustituye el beneficio.

- `classification_run_id` referencia ventana, umbrales, dataset y versión de fórmula.
- Se exportan hits, unidades, shares, `cum_before`, medias semanales, desviación, CV y empates.
- `min_class_runs=2` exige dos observaciones con `classification_cutoff` distintos, separados al menos por `stability_lag_weeks=4`, mismo almacén/SKU, configuración y versión de fórmula. Las ventanas pueden solaparse; repetir el mismo corte o dataset no crea estabilidad temporal.
- Las dos clases ABC y XYZ deben persistir entre esos cortes. Si difieren, el SKU queda `CLASS_NOT_STABLE`; si faltan observaciones, `INSUFFICIENT_CLASS_RUNS`.
- La primera corrida sin historia termina `SUCCEEDED_WITH_WARNINGS` si no hay errores críticos, con propuestas bloqueadas para los SKU afectados. Cero propuestas no es fallo.
- `min_class_runs<2` solo se admite con `data_mode=synthetic`; en otra fuente produce `INVALID_CLASS_STABILITY_CONFIG`. Backtesting estándar fija exactamente `2`.
- `slotting_state_snapshot` aporta aplicaciones, cooldown, dwell y recomendaciones/reservas activas. Ese estado y `classification_history` entran en `analysis_fingerprint` cuando afectan el resultado.
- Los SKUs `INSUFFICIENT_HISTORY` o `NO_DEMAND` no entran en optimización automática.

---

## 8. Modelo de desplazamiento y beneficio

### 8.1 Tiempo incremental por hit

El MVP calcula un proxy pick-face↔anchor, no un tour. Para cada ubicación:

```text
outbound_m(l) = path_distance_m(node(l), pick_anchor)
inbound_m(l)  = path_distance_m(pick_anchor, node(l))  si dirigido
              = outbound_m(l)                          si simétrico

pick_travel_s(l) = (outbound_m(l) + inbound_m(l)) / picker_speed_m_s
                   + location_access_s(l)
```

No existe un `round_trip_factor` multiplicador: la ida y la vuelta se suman explícitamente. Si una dirección necesaria falta, el par se excluye con `MISSING_DISTANCE`.

- `picker_speed_m_s` se configura por operación, no por persona.
- `location_access_s(l)` proviene de `location_access_times`; desconocido produce `MISSING_ACCESS_TIME`.
- Cada recomendación incluye `access_time_measured`, caminos, distancias y versiones.
- Euclídea solo se admite como `synthetic_euclidean`, simétrica y declarada; nunca se presenta como caminable real.
- Manipulación que no cambia entre ubicaciones se excluye para no contarla dos veces.

### 8.2 Demanda afectada y beneficio bruto

ABC/XYZ clasifica con todos los hits del SKU, pero el movimiento solo cambia el pick-face origen:

```text
affected_hits_s_from = count_distinct(hit_id válidos de s en from_location_id)

projected_affected_hits_s = affected_hits_s_from
                            × benefit_horizon_days / analysis_window_days

gross_benefit_s(s, from, to) = projected_affected_hits_s
                               × (pick_travel_s(from) - pick_travel_s(to))
```

Los hits de otras ubicaciones no se reasignan ni generan ahorro. El plan exporta las cohortes de esos hits afectados para la validación de LPPH. Un modelo de reasignación de demanda entre ubicaciones queda fuera de alcance.

La proyección supone tasa constante, no forecast. Beneficio negativo se rechaza aun para `AX`.

### 8.3 Unidades, traslado y costo de cambio

```text
units_to_move = current_on_hand_units  con snapshot vigente del origen
              = target_stock_units     solo en modo TARGET_ONLY
```

`relocation.units_to_move_source=SNAPSHOT_ELSE_TARGET`; el segundo caso marca `units_to_move_estimated=true` y exige snapshot antes de aprobar.

```text
si relocation_travel_mode=DISTANCE_BASED:
    relocation_travel_s = [path_distance_m(from,to)
                           + path_distance_m(to,from)] / relocation_speed_m_s

si relocation_travel_mode=CONFIGURED|SYNTHETIC:
    relocation_travel_s = relocation_travel_s_configured

relocation_cost_s = fixed_move_cost_s
                    + units_to_move × handling_cost_s_per_unit
                    + relocation_travel_s

net_benefit_s = gross_benefit_s - relocation_cost_s - disruption_cost_s
```

En matriz simétrica los dos trayectos de traslado usan la misma distancia; en dirigida se exigen ambas. `relocation_speed_m_s>0`, la fuente se declara `MEASURED|CONFIGURED|SYNTHETIC` y el valor entra en fingerprint/evidencia. Falta de distancia, velocidad o valor configurado produce `MISSING_RELOCATION_TRAVEL`; nunca se sustituye por cero silencioso.

`disruption_cost_s` es marginal por recomendación. El costo fijo de ventana se calcula aparte:

```text
plan_net_benefit_s = Σ net_benefit_s del plan - plan_fixed_disruption_cost_s
```

El plan agregado debe ser positivo; este total no altera retroactivamente el ranking ni se prorratea.

- Parámetros exportados con unidades y fuente (`MEASURED`, `CONFIGURED` o `SYNTHETIC`).
- `benefit_ratio=net_benefit_s/relocation_cost_s`; si costo es cero, ratio=`null` y `ratio_check=PASS` solo si el neto supera el mínimo.
- Defaults: `min_net_benefit_s=3600`, `min_benefit_ratio=1.0`, horizonte 84 días y máximo 20 movimientos.
- Bruto, reubicación, disrupción y neto nunca se combinan para ocultar costos.

### 8.4 Validación de configuración económica

Calidad crítica si se incumple cualquier regla:

- `picker_speed_m_s>0`, `relocation_speed_m_s>0`, horizonte y ventana `>0`.
- Costos, accesos, `min_net_benefit_s` y `min_benefit_ratio` `>=0`.
- `0<abc_a_threshold<abc_b_threshold<1`; `0<=xyz_x_threshold<xyz_y_threshold`.
- `attribution_factor` y cada sensibilidad en `[0,1]`; lista de sensibilidad no vacía y sin duplicados.
- `max_candidates_per_sku>=1`, `max_moves_per_run>=1`, `canonical_float_decimals` entero entre 0 y 9.
- `stability_lag_weeks>=1`; backtesting estándar exige `min_class_runs=2`.
- `synthetic_euclidean` no puede ser dirigido; modos/fuentes desconocidos se rechazan.

---

## 9. Optimizador de asignación `greedy_v0`

Heurística determinista; no afirma óptimo global.

### Universo elegible

Un SKU entra solo si:

1. Tiene primario único, clasificación válida y estable; faltantes producen `NO_PRIMARY_PICK_FACE`, `INSUFFICIENT_CLASS_RUNS` o `CLASS_NOT_STABLE`.
2. Tiene inventario/capacidad, acceso, datos físicos y ocupación de destino demostrables.
3. No está en cooldown, dwell, bloqueo o lista de exclusión.
4. Tiene hits afectados en el origen y al menos un destino factible que supera beneficio/costo.

El destino debe estar vacío según §5.5. No se modelan swaps, cadenas, consolidaciones ni pick-faces secundarios.

### Algoritmo

1. Validar universo y registrar todas las causas de exclusión.
2. Generar destinos factibles; conservar máximo `max_candidates_per_sku=10` por `(net_benefit_s DESC, gross_benefit_s DESC, to_location_id ASC)`.
3. Ordenar pares por `(net_benefit_s DESC, gross_benefit_s DESC, sku_id ASC, to_location_id ASC)`.
4. Aceptar un par si SKU y slot no fueron usados; reservar lógicamente el slot solo dentro del plan.
5. Detener al agotar candidatos o alcanzar `max_moves_per_run`.
6. `rank` es la secuencia de aceptación, comenzando en 1; `work_sequence=rank` salvo resecuenciación externa fuera del motor.
7. Calcular `plan_net_benefit_s`. Si hay seleccionados y no es positivo: `PLAN_NET_BENEFIT_NOT_POSITIVE`, `proposed_count=0`, sin prorrateo. Las recomendaciones seleccionadas descartadas por este rechazo se emiten en `exclusions.csv` con `primary_reason=PLAN_NET_BENEFIT_NOT_POSITIVE`; no se prorratea el costo fijo entre ellas.
8. Emitir plan viable y candidatos descartados por competencia, umbral, límite o rechazo del plan.

ABC/XYZ controla elegibilidad; el ranking usa beneficio neto y desempates, nunca una puntuación oculta de clase.

### Estabilidad, dwell, cooldown y vigencia

- Cooldown de 30 días por `(sku_id,to_location_id)` desde `APPLIED_EXTERNALLY`.
- Dwell de 30 días por SKU desde `last_slotting_applied_at`; sin historial de slotting no bloquea, pero las demás reglas siguen aplicando.
- `recommendation_key` identifica `(warehouse_id,sku_id,from_location_id,to_location_id)`; no puede haber dos activos con la misma clave.
- Cambios no materiales incrementan versión y conservan estado `PROPOSED`, `APPROVED` o `DEFERRED`.
- Expira si cambia signo del neto, cae bajo mínimos, el plan deja de ser positivo, una restricción pasa a `FAIL`, origen/destino dejan de ser vigentes o vence `valid_until`.
- `valid_until = min(snapshot_expires_at, assignment_valid_to, location_valid_to, distance_valid_to, data_cutoff_at + recommendation_validity_hours)`, ignorando valores nulos. Debe ser posterior al corte; aprobación y claim lo revalidan.

### Prioridad canónica de exclusiones

`primary_reason` es la primera causa presente en este registro; las secundarias conservan el mismo orden. Un código nuevo debe registrarse antes de usarse:

```text
NO_PRIMARY_PICK_FACE
INSUFFICIENT_CLASS_RUNS
CLASS_NOT_STABLE
INSUFFICIENT_HISTORY
NO_DEMAND
MISSING_CAPACITY_BASIS
ZERO_CAPACITY_BASIS
MISSING_DESTINATION_OCCUPANCY_EVIDENCE
MISSING_ACCESS_TIME
MISSING_DISTANCE
MISSING_RELOCATION_TRAVEL
OCCUPIED_SLOT
OCCUPIED_BY_SAME_SKU
SOURCE_EQUALS_DESTINATION
INCOMPATIBLE_TEMPERATURE
INCOMPATIBLE_HAZARD
INCOMPATIBLE_CATEGORY
INCOMPATIBLE_DIMENSIONS
EXCESS_WEIGHT
INSUFFICIENT_CAPACITY
RELOCATION_COOLDOWN
MIN_DWELL_NOT_MET
NEGATIVE_OR_BELOW_MIN_NET_BENEFIT
PLAN_NET_BENEFIT_NOT_POSITIVE
```

### Visibilidad de Zona Dorada bloqueada

`golden_zone_misallocation.csv` lista slots dorados ocupados por clase C o `NO_DEMAND`, hits, ocupación y layout. Es visibilidad, no una propuesta de swap/expulsión, y no altera `greedy_v0`.

---

## 10. KPIs y medición de impacto

Toda métrica declara cohorte, ventana, fuente y denominador. Denominador cero produce `null`.

### Operación y cobertura

- `completed_order_lines` deriva de `order_line_completions` y reversos; actividad no completada se reporta aparte como `active_pick_lines`.
- `LPPH = Σ derived_completed_order_lines / Σ picker_busy_hours` para cohortes no solapadas. Nunca se promedian ratios de cohortes.
- `golden_hit_coverage = hits de SKUs en Zona Dorada / hits totales`.
- `travel_seconds_per_hit = Σ pick_travel_s(location_id del hit) / count(hit_id)`, con la misma fórmula y versión de distancias del beneficio.
- `candidate_count`, `proposed_count`, aprobación, adopción y verificación.
- `blocked_golden_slots`, movimientos, churn y expiración.
- Beneficio bruto, costos, disrupción fija/marginal y neto, siempre separados.
- Diferencia de líneas derivadas vs. summary y estado `LPPH_SOURCE_UNRECONCILED`.

### LPPH proyectado

Antes de calcular se exige correspondencia exacta de:

1. **Periodo:** el baseline agregado cubre `[benefit_period_start,benefit_period_end)` y `benefit_horizon_days` es exactamente su duración; no se resta un beneficio de 84 días a horas de 7.
2. **Población:** `plan_cohort_ids` de los hits afectados coincide con `baseline_cohort_ids`; cohortes sin solapamiento y mismo almacén.
3. **Fuentes:** horas de viaje medidas, atribución justificada y reconciliación de líneas dentro de tolerancia.

Si no puede alinearse periodo, cohorte o actividad, `projected_LPPH=null` con `LPPH_PERIOD_MISMATCH`, `LPPH_COHORT_MISMATCH` o `LPPH_SOURCE_UNRECONCILED`. No hay normalización temporal implícita.

```text
net_plan_benefit_s = max(0, plan_net_benefit_s del plan evaluado)

attributed_savings_h = min(net_plan_benefit_s / 3600
                           × attribution_factor,
                           measured_travel_hours_baseline)

projected_busy_hours = picker_busy_hours - attributed_savings_h

si projected_busy_hours > 0:
    projected_LPPH = completed_order_lines / projected_busy_hours
si projected_busy_hours <= 0:
    projected_LPPH = null
```

- El manifiesto declara `plan_scope`, periodo, cohortes y estados incluidos; evaluar `PROPOSED` no implica aplicación.
- `0<=measured_travel_hours_baseline<=picker_busy_hours`; violación es error de datos.
- Horas no positivas producen `null` y `INVALID_PROJECTED_BUSY_HOURS`.
- Líneas y cohorte permanecen fijas; no se presupone demanda ni capacidad adicional.
- Se reporta sensibilidad, no compromiso.

### Impacto realizado

Después de `APPLIED_EXTERNALLY` se comparan ventanas pre/post equivalentes:

```text
realized_delta_LPPH_pct = 100 × (LPPH_post - LPPH_pre) / LPPH_pre
```

El reporte adjunta demanda, mix, dotación, horas, ausencias y cambios concurrentes. Sin grupo control o diseño adicional es **asociación pre/post**, no efecto causal.

---

## 11. Backtesting temporal

El backtesting evalúa la regla sin reescribir historia ni relajar XYZ/estabilidad.

### Protocolo cerrado

Default: **20 semanas ISO completas**, 16 de historia y 4 de prueba; `min_class_runs=2` y `stability_lag_weeks=4` son fijos para este protocolo.

1. Fijar `train_cutoff` al inicio exclusivo de las últimas 4 semanas antes de clasificar.
2. Clasificación de estabilidad con las 12 semanas terminadas en `train_cutoff-4 semanas`.
3. Clasificación de entrenamiento con las 12 semanas terminadas en `train_cutoff`. Los cortes son distintos y las ventanas pueden solaparse; la clase debe persistir o el SKU queda `CLASS_NOT_STABLE`.
4. Usar solo eventos con `recorded_at<=train_cutoff` y versiones de maestros/parámetros/layout/inventario/distancias/estado disponibles y vigentes en ese corte.
5. Generar plan con demanda afectada al origen. Ningún dato de prueba influye en clases, umbrales, velocidades, costos, accesos o selección.
6. Aplicar el plan hipotético a los picks de prueba y comparar contra baseline sin movimientos con idéntica demanda.
7. Exportar ahorro contrafactual, cobertura, clases, estabilidad, cohortes y fingerprints por etapa.

Si faltan 20 semanas: job `SUCCEEDED_WITH_WARNINGS`, `data_quality_status=WARNING`, `backtest_status=INSUFFICIENT_BACKTEST_HISTORY` y `AC-12=NOT_MET`; no invalida AC-02 del análisis principal ni otros AC independientes.

Si falta historia de vigencia se marca `LIMITED_CURRENT_LAYOUT`. Si falta `recorded_at` o historia de conocimiento/maestros/parámetros se marca `LIMITED_KNOWLEDGE_RECONSTRUCTION`. Ambas son sensibilidades, no réplicas históricas; el fixture de aceptación AC-12 debe incluir un caso bitemporal completo en `PASS` y ambas limitaciones correctamente etiquetadas.

`T-BACKTEST-001` incluye un evento anterior recibido después del corte y un maestro modificado posteriormente: ninguno puede filtrarse al entrenamiento. La proyección de LPPH sigue §10 y el beneficio realizado exige ejecución/medición posterior.

---

## 12. Recomendaciones y evidencia

### Artefacto canónico

`recommendations.csv` contiene `recommendation_id`, `recommendation_key`, `recommendation_version` y `analysis_fingerprint`, pero **no contiene `analysis_run_id`, `job_id` ni timestamps de ejecución**. Así puede ser byte-reproducible entre corridas independientes.

Cada recomendación incluye:

- Almacén, SKU, primario autoritativo, origen, destino, snapshot set y alcance de stock.
- Clase, historia de estabilidad y estado operativo usado.
- Hits totales para clasificación y `affected_hits_s_from` para beneficio, con cohortes.
- Unidades a mover y marca de estimación.
- Distancias dirigidas, caminos, acceso, capacidad y versiones.
- Bruto, traslado reconstructible, costos, disrupción, neto y ratio.
- Restricciones, `rank`, `valid_until`, estado y causas de exclusión.
- Hashes de configuración, fuentes, layout, distancias, historia y contenido.

### Evidencia operativa no canónica

`GET /recommendations/{id}/evidence`, historial y `run_manifest.json` añaden `analysis_run_id`, `job_id`, actores, `command_id`, estados, timestamps, tokens y revisiones. Esos datos no forman parte del diff canónico.

Aprobación y rechazo exigen comentario. Los overrides no modifican cálculos originales.

La orden de trabajo usa `work_sequence=rank`, token, revisión, vigencia y referencias legibles. Su ejecución es externa y sigue el protocolo de §5.7; exportar no equivale a iniciar ni completar trabajo.

---

## 13. Arquitectura del sistema

```text
PostgreSQL / Supabase (read-only)
              │
              ▼
      Ingesta + validación ──► snapshots de entrada
              │
              ▼
      Motor ABC/XYZ + asignación ──► schema slotting
              │                         │
              ├── CLI / reportes        ├── API FastAPI
              └── backtesting           └── worker de jobs
```

### Procesos

1. **API:** consulta corridas, solicita análisis, gestiona aprobaciones y exporta.
2. **Worker:** ejecuta jobs idempotentes y programados.
3. **CLI:** usa exactamente la misma librería de dominio sin levantar servicios.

La cola usa tablas del schema propio y `FOR UPDATE SKIP LOCKED`; no requiere Redis para el MVP. Un solo scheduler está activo por despliegue. Los workers paralelos no comparten RNG ni estado mutable fuera de PostgreSQL.

### Stack

- Python 3.12.
- FastAPI y Pydantic.
- SQLAlchemy y psycopg.
- PostgreSQL/Supabase.
- Pandas, NumPy y Pandera.
- Alembic para el schema propio.
- Pytest y Playwright para dashboard.
- HTML/Plotly para reportes y dashboard ligero.
- `pyproject.toml` y lockfile.

---

## 14. API, jobs, identidad e idempotencia

### Endpoints mínimos

| Operación | Comportamiento |
| --- | --- |
| `POST /analysis-runs` | Crea job; `202` con `job_id`. |
| `GET /analysis-runs/{id}` | Estado, calidad, conteos y errores. |
| `GET /recommendations` | Filtros por run, clase, estado y beneficio. |
| `POST /recommendations/{id}/transitions` | Transición con `command_id` y versión. |
| `POST /recommendations/{id}/execution-claim` | Toma de trabajo con token/revisión y versión. |
| `GET /recommendations/{id}/evidence` | Evidencia operativa completa. |
| `GET /exports/{id}` | Artefactos publicados, nunca staging. |
| `GET /health` | Salud del proceso, no una corrida. |

### Identidad, claves e idempotencia

- `job_id` y `analysis_run_id`: UUID v4 únicos; no se reutilizan por `UPSERT`.
- `analysis_fingerprint`: SHA-256 de entradas, configuración, mapping, layout, distancias, clasificación/estado histórico y versión del motor que afecten al resultado.
- `result_content_hash`: hash sin IDs de ejecución ni timestamps. Dos runs independientes tienen IDs distintos y pueden compartir fingerprint/hash.
- `recommendation_key`: SHA-256 de `warehouse_id`, `sku_id`, `from_location_id`, `to_location_id`, en ese orden, Unicode NFC, sin espacios periféricos, sensibles a mayúsculas, codificados en UTF-8 y unidos por el byte Unit Separator `0x1F`; el símbolo `␟` solo es representación visual.
- `recommendation_id=UUIDv5(0e6f0f52-6a6d-5b1f-9f4f-3f3d6bb0f001, recommendation_key)`; namespace constante y versionado en manifiesto.
- La clave HTTP de idempotencia es `(actor, operation, Idempotency-Key)`. El hash del payload se **almacena para comparar**, no forma parte de la identidad. Misma clave/payload devuelve el mismo job; mismo key con payload distinto produce `IDEMPOTENCY_KEY_REUSED`.
- Solicitudes simultáneas se resuelven con restricción única e inserción atómica; solo una crea el job.
- Transiciones y claims usan `command_id` UUID v4 y `expected_version`; repetir comando devuelve el mismo resultado y conflicto produce `STATE_CONFLICT`.

### Estados de job

```text
QUEUED
RUNNING
SUCCEEDED
SUCCEEDED_WITH_WARNINGS
FAILED
CANCELLED
```

- Primera corrida sin estabilidad: `SUCCEEDED_WITH_WARNINGS`, `INSUFFICIENT_CLASS_RUNS`, sin forzar propuestas.
- Historia backtest insuficiente: `SUCCEEDED_WITH_WARNINGS`, calidad `WARNING`, `backtest_status=INSUFFICIENT_BACKTEST_HISTORY`, AC-12=`NOT_MET`.
- `functional_status=PASS` admite éxito/advertencias sin calidad crítica y artefactos válidos; `proposed_count>=1` no es requisito.
- `FAILED`, `CANCELLED`, `INVALID_SNAPSHOT` o calidad crítica impiden publicar resultados exitosos.

### Recuperación y publicación consistente

Cada job registra `attempt`, `max_attempts`, `lease_owner`, `lease_expires_at` y `heartbeat_at`.

- Si el worker cae tras tomar el job, expira el lease y otro worker lo retoma con `attempt+1`, misma clave de idempotencia y entradas inmutables. Superar intentos produce `FAILED` con historial.
- Los artefactos se escriben en staging con checksums. Solo después de completarlos se crea atómicamente `publication_marker` y se actualiza el job; los consumidores ven únicamente conjuntos publicados.
- Caída antes del marker: staging se descarta/reutiliza y se reconstruye; nunca hay éxito parcial. Caída después del marker y antes de respuesta: retry devuelve el resultado publicado.
- Reintentos de fuente usan backoff solo en operaciones idempotentes. Dashboard no cancela jobs.

### Contratos, orden y reproducción

`analysis_config`, `source_mapping` y resultados llevan `schema_version`; cambios incompatibles incrementan mayor.

CSV canónicos usan UTF-8, LF, decimales `ROUND_HALF_UP`, columnas declaradas y orden explícito:

- Clasificación: `sku_id ASC`.
- Recomendaciones: `rank ASC, sku_id ASC, to_location_id ASC`.
- Exclusiones: `sku_id ASC, location_id ASC, primary_reason ASC`.
- Misallocation: `location_id ASC, sku_id ASC`.
- Trabajo: `work_sequence ASC`.

`rank` es secuencia de aceptación y `work_sequence=rank`. `recommendations.csv` no contiene run/job IDs. Los manifiestos conservan esos valores fuera del contenido canónico; el diff compara artefactos canónicos directamente y manifiestos tras normalizar únicamente campos operativos declarados.

---

## 15. Dashboard y reportes

### Pantallas del producto completo

1. Estado de fuentes y calidad.
2. Configuración y ejecución.
3. Matriz ABC/XYZ.
4. Recomendaciones y comparación actual/destino.
5. Exclusiones por restricción.
6. Backtesting.
7. Aprobación, aplicación y verificación.

### Visualizaciones mínimas

- Matriz 3×3 con conteos e hits.
- Pareto de frecuencia con umbrales.
- Tabla de beneficio bruto/costo/neto.
- Mapa esquemático actual/destino sin afirmar escala CAD.
- Embudo propuesto→aprobado→aplicado→verificado.
- LPPH observado pre/post con cohortes y advertencias.

### Exportaciones

- `sku_classification.csv`.
- `recommendations.csv` y `recommendations.json`.
- `exclusions.csv` con causa primaria y una fila por causa secundaria.
- `golden_zone_misallocation.csv` como visibilidad, sin acciones automáticas.
- `work_order.csv` solo para aprobadas y con snapshot revalidado.
- `analysis_report.html` autocontenido.
- `run_manifest.json` con hashes, fingerprint, IDs de ejecución y evidencias.

---

## 16. Seguridad y operación

- Credenciales mediante gestor de secretos o variables de entorno; nunca en SQL, logs, reportes o repositorio.
- Usuario de base con permisos mínimos de lectura sobre vistas aprobadas y sin acceso a tablas no necesarias.
- Schema `slotting` separado, con rol de escritura exclusivo del servicio.
- TLS para conexiones administradas; `sslmode` y verificación documentados.
- Consultas parametrizadas y mapping declarativo; ningún identificador de tabla se concatena desde entrada de usuario.
- Límites de `statement_timeout`, `lock_timeout`, memoria, filas y duración de job.
- Logs sin datos personales de pickers; se usan IDs agregados o pseudonimizados cuando la fuente los contenga.
- API del producto completo con autenticación y autorización por rol: lectura, aprobación y administración. G-portafolio CLI no expone red.
- Retención configurable de corridas y evidencias; no se borra la fuente ni el historial de aprobación.
- Métricas de jobs, latencia de fuente, filas procesadas, errores y tamaño de artefactos.

---

## 17. Requisitos no funcionales y benchmark

- Resultados deterministas y hashes canónicos.
- Lectura incremental sin bloquear la operación de la fuente.
- Escritura limitada al schema propio.
- Procesamiento por lotes con memoria acotada.
- Instalación y demo documentadas.
- Sin credenciales, datos reales ni secretos en el repositorio.
- Toda estimación distinguible de una medición realizada.

### 17.1 Meta y protocolo

**Meta a verificar, no resultado obtenido:** procesar 1,000,000 de movimientos, 10,000 SKUs y 2,000 ubicaciones con una mediana menor de 60 segundos en el perfil de referencia.

Perfil: Intel Core i5-8250U, 16 GB RAM, SSD, Linux x86_64, Python 3.12, single-process y sin UI. Otro hardware produce `NOT_COMPARABLE`, salvo que se defina otra referencia antes de medir.

`make bench` no levanta API, worker ni dashboard. Archiva:

1. Dataset pregenerado, semilla, configuración y hashes.
2. Región medida: lectura local, validación, ABC/XYZ, candidatos, asignación y exportación. La latencia de red Supabase se mide en un benchmark separado.
3. Una corrida de calentamiento y cinco medidas en procesos nuevos.
4. Duraciones individuales, mediana, máximo, RSS y conteos; la aceptación usa la mediana. RSS se reporta, pero no tiene umbral de aceptación en este brief.
5. Igualdad de hashes canónicos en todas las corridas. Las duraciones se guardan aparte.
6. Manifiesto de hardware, sistema, dependencias, comando y commit.

Un benchmark con resultados distintos o invariantes rotos es inválido aunque sea rápido.

### 17.2 Presupuesto de esfuerzo del producto completo

- **60%:** Fase 1 — datos, motor, optimización y pruebas.
- **20%:** Fase 2 — API, worker, seguridad y flujo de aprobación.
- **20%:** Fase 3 — dashboard, backtesting ampliado y demo.

El porcentaje no equivale a semanas. G-portafolio permite publicar el núcleo verificado antes de completar G2/G3.

### 17.3 Decisiones normativas de implementación

| ID | Decisión cerrada |
| --- | --- |
| D-01 | ABC usa hits por `(pick_task_id, location_id)`; LPPH usa líneas; XYZ usa unidades semanales. |
| D-02 | Semanas con cero entran en XYZ; historia insuficiente no se clasifica automáticamente. |
| D-03 | Los empates ABC se resuelven por unidades y `sku_id`; el cruce de umbral usa `cum_before`. |
| D-04 | Sin snapshot/capacidad o datos físicos no hay recomendación factible. |
| D-05 | Zona Dorada es atributo explícito; cercanía no basta. |
| D-06 | `greedy_v0` no ejecuta swaps, cadenas ni múltiples pick-faces por SKU. |
| D-07 | El ranking usa beneficio neto; la clase controla política/elegibilidad. |
| D-08 | Distancia euclídea solo se permite en modo sintético declarado. |
| D-09 | LPPH proyectado exige periodo/cohortes alineados, líneas reconciliadas, viaje medido y atribución; si falta, es `null`. |
| D-10 | Backtesting usa 16 semanas de historia + 4 de prueba, dos cortes XYZ válidos y datos `as_of`; la limitación de layout actual se declara. |
| D-11 | Reintentos y aprobaciones son idempotentes; estados usan versión esperada. |
| D-12 | La fuente es read-only; la aplicación física siempre es externa. |
| D-13 | Inventario y unidades a mover se basan en el pick-face primario; reserva no se consolida. |
| D-14 | `analysis_run_id` es UUID v4; el fingerprint de contenido es determinista y separado. |
| D-15 | Acceso desconocido excluye; slots destino ocupados se excluyen incluso si contienen el mismo SKU. |
| D-16 | `disruption_cost_s` es marginal por movimiento; el costo fijo del plan se calcula aparte. |
| D-17 | ABC usa demanda total; beneficio usa solo hits afectados del pick-face origen. |
| D-18 | Viajes y traslados suman direcciones; no existe factor de ida/vuelta implícito. |
| D-19 | Primario, acceso, capacidad y ocupación tienen fuentes versionadas; ausencia no inventa cero. |
| D-20 | Estabilidad exige cortes distintos y clase persistente; estado histórico entra en fingerprint. |
| D-21 | Backtesting reconstruye vigencia y conocimiento; limitaciones no se venden como réplica. |
| D-22 | Reservas y claims transaccionales impiden aprobaciones/ejecuciones incompatibles. |
| D-23 | Artefactos canónicos omiten IDs de ejecución; publicación es atómica y recuperable. |
| D-24 | Brief, contratos, configuración y pruebas tienen autoridad explícita y entradas comunes para agentes. |

---

## 18. Estructura propuesta

```text
slotting-optimizer-engine/
├── AGENTS.md
├── CLAUDE.md
├── api/
│   ├── main.py
│   ├── runs.py
│   └── recommendations.py
├── worker/
│   ├── runner.py
│   └── scheduler.py
├── slotting/
│   ├── classify.py
│   ├── eligibility.py
│   ├── benefit.py
│   ├── optimizer.py
│   ├── backtest.py
│   └── metrics.py
├── connectors/
│   ├── postgres.py
│   ├── supabase.py
│   └── mapping.py
├── data/
│   ├── contracts.py
│   ├── validation.py
│   └── synthetic.py
├── reporting/
│   ├── export.py
│   └── html.py
├── dashboard/
├── contracts/
│   ├── analysis_config.yaml
│   ├── analysis_config.demo.yaml
│   ├── analysis_config.tz_demo.yaml
│   ├── source_mapping.yaml
│   └── source_mapping.demo.yaml
├── samples/
│   ├── movements_demo.csv
│   ├── movements_demo_tz.csv
│   ├── order_line_completions_demo.csv
│   ├── skus_demo.csv
│   ├── sku_primary_pick_face_demo.csv
│   ├── locations_demo.csv
│   ├── location_access_times_demo.csv
│   ├── location_capacity_rules_demo.csv
│   ├── snapshot_sets_demo.csv
│   ├── inventory_demo.csv
│   ├── distance_matrix_demo.csv
│   ├── picker_time_summaries_demo.csv
│   ├── classification_history_demo.csv
│   ├── slotting_state_snapshot_demo.json
│   └── dataset_manifest.json
├── tests/
├── scripts/
├── docs/
│   ├── specification.md
│   ├── INTAKE.md
│   └── audits/
├── artifacts/acceptance/
├── migrations/
├── Makefile
├── pyproject.toml
├── README.md
└── .gitignore
```

---

## 19. Catálogo canónico de pruebas

Esta tabla es la única fuente normativa de IDs. §20, §21 y §25 solo pueden referenciar pruebas aquí definidas. **MVP** significa obligatoria en G-portafolio y heredada por el producto completo; **FULL** significa obligatoria únicamente para el producto completo.

| Test ID | Propósito y casos mínimos | Evidencia | AC | Nivel |
| --- | --- | --- | --- | --- |
| `T-DATA-001` | Contratos, mapping, corte bitemporal, primario, acceso, capacidad, snapshots, conflictos de configuración y read-only local/DB. | Golden dataset, errores, hash pre/post de `samples/` en MVP y grants sin escritura en FULL. | AC-01 | MVP |
| `T-DATA-002` | Hits con varios movimientos, reversos, líneas completas/parciales, ISO/DST/timezones, cohortes, cohorte inconsistente por hit y granularidad. | Conteos por movimiento/hit/línea, finalización, fixtures UTC/Mexico y reconciliación. | AC-05 | MVP |
| `T-ABC-001` | Shares, acumulados, umbrales, empates, un solo SKU A y clases vacías. | Valores esperados, clases y evidencia por SKU. | AC-03 | MVP |
| `T-XYZ-001` | Semanas cero, media, desviación muestral, CV, X/Y/Z e historia insuficiente. | Series semanales, parámetros y resultados esperados. | AC-04 | MVP |
| `T-CONSTRAINT-001` | Primario, peso, dimensiones/orientación, capacidad autorizada, compatibilidad, ocupación y acceso. | Casos primario/secundario/reserva, captura completa/parcial/antigua y volumen suficiente con dimensión incompatible. | AC-07 | MVP |
| `T-OPT-001` | Un SKU/slot por corrida, rank, empates, límites, cooldown, dwell y ausencia de swaps. | Plan, reservas lógicas, causas y golden cases. | AC-08 | MVP |
| `T-BENEFIT-001` | Hits afectados multiubicación, viajes dirigidos, traslado reconstructible, unidades, costos y configuración. | Caso 20/80 hits sin atribución cruzada, dirección inaccesible, fórmulas y golden cases. | AC-09 | MVP |
| `T-ELIGIBILITY-001` | Balance del universo, prioridad de causas, vigencia, expiración, rechazo de plan y misallocation sin acción. | Exclusiones completas —incluidas las seleccionadas descartadas por `PLAN_NET_BENEFIT_NOT_POSITIVE`—, `valid_until`, estados y reportes reconciliados. | AC-10 | FULL |
| `T-EXPORT-001` | Schemas, orden, redondeo, ausencia de run IDs en canónicos, distancias, tiempos y misallocation. | CSV/JSON/HTML reabiertos, conteos, hashes y contratos; control MVP de exclusiones. | AC-11 | MVP |
| `T-BACKTEST-001` | 16+4, estabilidad persistente, conocimiento/vigencia `as_of`, baseline y estados de historia insuficiente. | Evento tardío, maestro posterior, caso bitemporal PASS y limitaciones etiquetadas. | AC-12 | FULL |
| `T-KPI-001` | Finalización de líneas, reconciliación, cohortes, periodo de 84 días, techos y guardas `null`. | Mismatch 84/7 y de cohortes, diferencias summary, agregados y golden KPI. | AC-13 | MVP |
| `T-E2E-CLI-001` | CLI, estados terminales, primera corrida, cero propuestas y exclusiones mínimas completas. | Logs, exit codes, calidad, causas y artefactos. | AC-02 | MVP |
| `T-REPRO-001` | Repetición/paralelismo, historia en fingerprint, run IDs distintos y CSV canónicos idénticos. | Hashes, diff vacío, manifests normalizados y resultados por proceso. | AC-06 | MVP |
| `T-JOB-001` | Idempotencia actor+operación+clave, payloads simultáneos, leases, caídas y publicación atómica. | Mismo job, conflicto por payload, recuperación antes/después del marker y ausencia de éxito parcial. | AC-14 | FULL |
| `T-WORKFLOW-001` | Reservas transaccionales, aprobaciones concurrentes, claims, cancelación, parcial, reconciliación con nueva versión y ejecución externa. | Un solo ganador, tokens invalidados, `STATE_CONFLICT`, claves/versiones tras parcial, evidencia y diff de fuente. | AC-15 | FULL |
| `T-UI-001` | Matriz, beneficio, exclusiones, misallocation y seguimiento. | Playwright, capturas y valores contra resultados canónicos. | AC-16 | FULL |
| `T-SEC-001` | Permisos, TLS, secretos, datos sensibles, logs y exports. | Escaneos, roles y configuración sin secretos. | AC-17 | FULL |
| `T-INSTALL-001` | Entorno limpio, fixtures completos, UTC/TZ, agentes/Intake, ausencia MVP de secretos y límites de alcance. | Comandos, enlaces AGENTS/CLAUDE, escaneo básico, schemas, README y checklist. | AC-19 | MVP |
| `T-PERF-EVIDENCE-001` | Protocolo de benchmark y manifiesto, sin imponer el umbral de 60 s. | Warmup, cinco medidas, RSS, hardware, hashes y comparabilidad. | AC-02, AC-19 | MVP |
| `T-PERF-001` | Mediana menor de 60 s en perfil de referencia con resultados válidos. | Cinco duraciones, mediana, máximo, RSS y commit. | AC-02 | FULL |
| `T-SCOPE-001` | Ausencia de autoescritura, multi-slot, swaps, FEFO, 3D y otra Fase 4. | Inventario de alcance/dependencias y checklist; control MVP básico en instalación. | AC-20 | FULL |

### Suites compuestas cerradas

`T-CRITICAL-PORTFOLIO` contiene exactamente estos **13 IDs atómicos**:

```text
T-DATA-001
T-DATA-002
T-ABC-001
T-XYZ-001
T-CONSTRAINT-001
T-OPT-001
T-BENEFIT-001
T-EXPORT-001
T-KPI-001
T-E2E-CLI-001
T-REPRO-001
T-INSTALL-001
T-PERF-EVIDENCE-001
```

`T-CRITICAL-001` contiene exactamente los **21 IDs atómicos** de la tabla, incluidos `T-PERF-EVIDENCE-001` y `T-PERF-001`. No incluye suites anidadas. AC-18 verifica su ejecución completa.

Ninguna suite puede quedar verde con fallos, omisiones o skips. `T-PERF-001` conserva por separado la aserción de rendimiento del producto completo. Cobertura sugerida: **75% en `slotting/`, `connectors/` y `data/`**, solo informativa.

---

## 20. Criterios de aceptación y evidencia

El **producto completo** termina cuando los 20 criterios tienen `PASS`. G-portafolio usa el subconjunto de la sección 25 y no reclama completitud. Las evidencias se guardan en `artifacts/acceptance/<run_id>/` con comandos, commit, entorno, resultados y SHA-256.

| ID | Criterio verificable | Prueba y evidencia mínima |
| --- | --- | --- |
| AC-01 | Lee fuente/dataset y valida contratos, duplicados, reversos, cortes y cobertura. | `T-DATA-001`: golden cases, conteos, hash local pre/post en MVP y grants read-only en FULL. |
| AC-02 | Ejecuta análisis CLI hasta `SUCCEEDED` o `SUCCEEDED_WITH_WARNINGS` sin calidad crítica; `proposed_count=0` puede ser válido. Producto completo además exige benchmark `PASS`. | `T-E2E-CLI-001` y `T-PERF-001`: estados funcional/rendimiento separados. Si rendimiento falla, AC-02=`NOT_MET`; G-portafolio puede publicar con esa limitación visible. |
| AC-03 | Calcula ABC con frecuencia, acumulados y empates exactos. | `T-ABC-001`: valores esperados, clases y evidencia por SKU. |
| AC-04 | Calcula XYZ con semanas cero, CV e historia insuficiente. | `T-XYZ-001`: series semanales, media, desviación, CV y clases. |
| AC-05 | Reconcilia movimientos, hits compartidos, líneas completadas, unidades, reversos y cortes temporales. | `T-DATA-002`: agregación por movimiento/hit/línea, finalización, ISO/timezones y cohortes. |
| AC-06 | Reproduce clases, plan, rankings y hashes con entradas e historial idénticos. | `T-REPRO-001`: run IDs distintos, fingerprints iguales y diff canónico vacío sin normalizar CSV. |
| AC-07 | Admite solo movimientos con primario, capacidad, compatibilidad, acceso y ocupación demostrados. | `T-CONSTRAINT-001`: matriz física/operativa, capturas y exclusiones con causas. |
| AC-08 | Asigna sin conflictos SKU/slot y respeta límites, cooldown y desempates. | `T-OPT-001`: plan, reservas lógicas y golden cases. |
| AC-09 | Calcula beneficio solo con demanda afectada y costos reconstructibles. | `T-BENEFIT-001`: hits multiubicación, direcciones, traslado, costos, neto y ratio. |
| AC-10 | Explica cada exclusión sin descartar silenciosamente SKUs o slots. | `T-ELIGIBILITY-001`: `exclusions.csv`, causa primaria/secundarias y balance del universo. |
| AC-11 | Exporta clasificación, recomendaciones, exclusiones y evidencias canónicas reutilizables. | `T-EXPORT-001`: archivos reabiertos, orden/redondeo, hashes y ausencia de run IDs en CSV canónico. |
| AC-12 | Backtesting reconstruye vigencia y conocimiento, evita fuga y compara igual demanda. | `T-BACKTEST-001`: protocolo 16+4, estabilidad, eventos tardíos, maestros y fixture bitemporal `PASS`. |
| AC-13 | Calcula LPPH con líneas completadas y proyecta solo con periodo/cohortes/fuentes alineados. | `T-KPI-001`: reconciliación, agregados, mismatch 84/7, viaje, atribución y advertencias pre/post. |
| AC-14 | API/worker ejecutan jobs idempotentes, recuperables y con publicación atómica. | `T-JOB-001`: concurrencia, reintentos, caídas, leases, markers y ausencia de resultados parciales. |
| AC-15 | Aprobación, reserva, claim y ejecución externa evitan conflictos y preservan historial. | `T-WORKFLOW-001`: aprobaciones/claims concurrentes, tokens, parcial, evidencia y fuente sin cambios. |
| AC-16 | Dashboard/reporte muestran matriz, beneficio, exclusiones y seguimiento. | `T-UI-001`: Playwright, capturas y valores contrastados contra resultados canónicos. |
| AC-17 | Credenciales y datos sensibles no aparecen en logs, artefactos o repositorio. | `T-SEC-001`: escaneo de secretos, permisos, logs y exports. |
| AC-18 | Todas las pruebas críticas del alcance están verdes. | `T-CRITICAL-001`: JUnit sin fallos/omisiones y cobertura solo informativa. |
| AC-19 | Instalación, límites, entradas de agentes y demo son repetibles en el alcance declarado. | `T-INSTALL-001` y `T-PERF-EVIDENCE-001`: entorno limpio, fixtures, AGENTS/CLAUDE, README y benchmark documentado. |
| AC-20 | No incorpora autoescritura WMS, multi-slot, FEFO, 3D u otra Fase 4. | `T-SCOPE-001`: inventario de dependencias/alcance y checklist de exclusiones. |

Una desviación se marca `NOT_MET` con causa. Ningún porcentaje de cobertura, captura o ejemplo hipotético sustituye estas evidencias.

---

## 21. Roadmap y puertas de entrega

Las fases ordenan la construcción; no constituyen una promesa de calendario. Una fase solo se cierra con sus pruebas, artefactos y limitaciones registrados. Ninguna fase posterior puede ocultar un fallo del núcleo.

### 21.1 Fases y puertas

| Fase | Contenido | Puerta de salida |
| --- | --- | --- |
| **Fase 1 — Núcleo verificable** | Dataset sintético, contratos, lectura local, ABC/XYZ, elegibilidad, beneficio, `greedy_v0`, CLI, exportaciones y evidencias. | **G1:** `T-CRITICAL-PORTFOLIO` completo y en `PASS`; demo e instalación limpia repetibles. |
| **Fase 2 — Servicio operativo** | Conector incremental PostgreSQL/Supabase, schema `slotting`, API, worker, roles, jobs, transiciones y aplicación externa evidenciada. | **G2:** AC-14, AC-15 y AC-17 en `PASS`; reintentos idempotentes, fuente sin escritura, secretos ausentes y contratos de API versionados. |
| **Fase 3 — Validación completa** | Dashboard, backtesting, medición pre/post, benchmark, endurecimiento y expediente completo. | **G3:** los 20 AC en `PASS`, `T-CRITICAL-001` verde y `T-PERF-001` con resultado `PASS` en el perfil de referencia. |
| **Fase 4 — Futuro, fuera de alcance** | Optimización global, swaps/cadenas, multi-slot, lotes/FEFO, ruteo de tours, pronóstico o 3D. | No habilita G1/G2/G3 ni G-portafolio. Requiere un brief y criterios nuevos. |

### 21.2 Secuencia obligatoria

1. Congelar contratos, golden datasets y fórmulas antes de optimizar velocidad.
2. Construir primero el motor headless; CLI, API y dashboard consumen la misma librería.
3. Verificar factibilidad y beneficio antes de desarrollar visualizaciones.
4. Incorporar la fuente administrada solo después de probar el corte y la lectura sin escritura.
5. Medir rendimiento únicamente sobre resultados correctos y reproducibles.
6. Publicar G-portafolio cuando cumpla la sección 25, aunque G2/G3 sigan pendientes.

Si una dependencia obligatoria no está disponible, se registra el bloqueo y se reduce el universo elegible; no se sustituye por un supuesto silencioso.

---

## 22. Guion de demo reproducible

La demo usa exclusivamente datos sintéticos. Su objetivo es demostrar trazabilidad y límites, no una mejora real de almacén.

### 22.1 Escenarios incluidos

- `SKU-AX-001`: candidato compatible con 100 hits afectados en origen y beneficio neto superior al mínimo.
- `SKU-AX-006`: 100 hits totales, solo 20 en origen; el beneficio usa 20 y nunca los 80 restantes.
- Un grafo con direcciones asimétricas y otro con retorno inaccesible para validar viaje y exclusión.
- Capturas completa, parcial y antigua: solo la completa puede demostrar destino vacío.
- `SKU-AZ-002`: alta rotación, pero sin base de capacidad vigente; debe quedar excluido.
- `SKU-BX-003`: destino compatible cuyo beneficio neto no alcanza el umbral.
- `SKU-AY-004` y `SKU-AX-005`: compiten por el mismo slot; el desempate y la reserva lógica deben ser visibles.
- Movimientos duplicados, un `pick_task_id` en dos ubicaciones, dos movimientos bajo un mismo hit y reversos parciales/totales para reconciliar hits, líneas completadas y unidades.
- Una semana sin unidades y eventos en el límite domingo/lunes con offset.
- Un slot Zona Dorada ocupado por clase C para el reporte de misallocation, sin generar swaps.

La clasificación del escenario usa `contracts/analysis_config.demo.yaml` con `min_class_runs=1`, relajación permitida **solo con `data_mode=synthetic`**. El archivo operativo `analysis_config.yaml` conserva `2`; copiar el valor demo a una fuente real produce `INVALID_CLASS_STABILITY_CONFIG`.

### 22.2 Secuencia

```bash
make install
make test-portfolio
make demo
make demo-reproduce
make bench-portfolio
```

1. `make install` crea el entorno desde el lockfile y verifica versiones sin descargar datos reales.
2. `make test-portfolio` ejecuta exactamente `T-CRITICAL-PORTFOLIO` y genera JUnit, logs y hashes.
3. `make demo` carga los CSV de movimientos, maestros, inventario, distancias y tiempos; ejecuta la CLI y produce clasificación, exclusiones, misallocation, recomendaciones, reporte HTML y manifiesto.
4. `make demo-reproduce` repite el análisis en un proceso nuevo y exige diff vacío directo de artefactos canónicos, incluidos CSV sin run IDs. `run_manifest.json` se compara tras normalizar únicamente campos operativos declarados; `result_content_hash` debe ser idéntico.
5. `make bench-portfolio` ejecuta el protocolo y archiva duración, RSS, hardware y hashes; no convierte la meta de 60 s en requisito de portafolio.

### 22.3 Recorrido de presentación

1. Mostrar ventana, corte, semilla, mapping y hashes antes de los resultados.
2. Abrir el Pareto y la matriz ABC/XYZ; comprobar un cruce de umbral y una semana cero.
3. Explicar `SKU-AZ-002`: clase alta no equivale a recomendación factible.
4. Comparar para `SKU-AX-001` y `SKU-AX-006` demanda total vs. afectada, viaje actual/destino, traslado reconstructible, costos y beneficio neto.
5. Mostrar la competencia por slot y demostrar que ningún SKU ni destino se asigna dos veces.
6. Reabrir CSV/JSON y cotejar sus valores con el reporte; no validar cifras solo mediante capturas.
7. Mostrar LPPH observado y, si faltan horas de viaje o atribución, comprobar `projected_LPPH=null`.
8. Mostrar el slot dorado ocupado por clase C y comprobar que el reporte no propone un swap.
9. Repetir la corrida y verificar igualdad de hashes con run IDs independientes.
10. Cerrar con las limitaciones y el estado del benchmark.

**Demo exitosa:** comandos reproducibles, escenarios previstos resueltos según contrato, artefactos reabiertos, hashes iguales y ninguna afirmación de impacto realizado. Una UI parcial no bloquea esta demo headless; un error de cálculo sí la invalida.

---

## 23. Entregables por alcance

Las etiquetas son normativas: **[MVP]** pertenece a G-portafolio; **[FULL]** se añade para el producto completo. Un entregable FULL no puede presentarse como disponible por aparecer en el árbol del repositorio.

| Etiqueta | Entregable | Contenido mínimo verificable |
| --- | --- | --- |
| **[MVP]** | Motor de dominio | ABC/XYZ, contratos, elegibilidad, distancias, beneficio, `greedy_v0` y KPIs. |
| **[MVP]** | CLI, contratos y agentes | Comando único, YAML versionados, `AGENTS.md`, `CLAUDE.md`, `docs/INTAKE.md` y códigos honestos. |
| **[MVP]** | Dataset sintético | Demo, benchmark, generador, seed, golden cases y hashes. |
| **[MVP]** | Exportaciones | Clasificación, recomendaciones, exclusiones, `golden_zone_misallocation`, reporte HTML y manifiesto reutilizables. |
| **[MVP]** | Pruebas de portafolio | `T-CRITICAL-PORTFOLIO`, JUnit, logs, evidencia funcional y protocolo de rendimiento. |
| **[MVP]** | Instalación y demo | README, lockfile, Makefile, comandos, troubleshooting y limitaciones visibles. |
| **[MVP]** | Evidencia de publicación | Estado de los 12 criterios, commit, entorno, hashes y resultado de benchmark separado. |
| **[FULL]** | Conector administrado | PostgreSQL/Supabase read-only, incrementalidad, watermark, lookback y corte consistente. |
| **[FULL]** | Persistencia propia | Migraciones del schema `slotting`, retención y reconciliación de corridas. |
| **[FULL]** | API y worker | Jobs, idempotencia, autenticación, roles, límites y observabilidad. |
| **[FULL]** | Gobierno del cambio | Aprobación, rechazo, orden de trabajo, ejecución externa e historial. |
| **[FULL]** | Backtesting | Historia 16+4, dos cortes XYZ, baseline contrafactual, versiones `as_of` y advertencias de layout actual. |
| **[FULL]** | Dashboard | Matriz, Pareto, beneficios, exclusiones, seguimiento y pruebas Playwright. |
| **[FULL]** | Seguridad y operación | Escaneo de secretos, permisos, TLS, logs, métricas y recuperación de jobs. |
| **[FULL]** | Benchmark completo | `T-PERF-001` en perfil de referencia, cinco medidas y evidencia comparable. |
| **[FULL]** | Expediente final | Los 20 AC, `T-CRITICAL-001`, cobertura informativa y registro de desviaciones. |

La documentación de arquitectura y este brief acompañan ambos alcances. El README debe mostrar el alcance efectivamente entregado, no el árbol aspiracional completo.

---

## 24. Reglas no negociables

1. **Solo lectura:** la fuente operativa nunca recibe escrituras del motor.
2. **Decisión humana:** una propuesta no equivale a aprobación, aplicación ni verificación.
3. **Semántica separada:** hits, líneas, unidades y eventos no se intercambian.
4. **Corte explícito:** toda cifra declara ventana, zona horaria, `data_cutoff_at` y versión de entradas.
5. **ABC no es ahorro:** sin distancia y beneficio neto positivo no hay recomendación.
6. **Factibilidad demostrable:** sin inventario/capacidad, atributos físicos, compatibilidad o acceso conocido no se afirma que el movimiento cabe ni que ahorra tiempo.
7. **Zona Dorada explícita:** no se inventa por proximidad ni por una clase ABC.
8. **Distancia honesta:** el proxy no se presenta como tour; euclídea solo en modo sintético declarado.
9. **Costos visibles:** bruto, reubicación, disrupción y neto se reportan por separado.
10. **Determinismo:** mismas entradas y versión producen los mismos resultados canónicos.
11. **Sin fuga temporal:** entrenamiento y prueba se separan antes de clasificar y seleccionar.
12. **LPPH condicionado:** sin horas de viaje y factor de atribución, la proyección es `null`.
13. **No causalidad espuria:** pre/post sin diseño adicional se denomina asociación.
14. **Sin secretos ni datos reales en el repositorio:** esta regla también aplica al MVP, aunque AC-17 sea FULL.
15. **Estados honestos:** `FAILED`, `NOT_MET`, `NOT_COMPARABLE`, `null` y las exclusiones no se convierten en éxito.
16. **Sin ampliación silenciosa:** swaps, multi-slot, FEFO, autoescritura, pronóstico o 3D requieren otro alcance.
17. **Evidencia sobre narrativa:** cobertura, capturas o ejemplos no sustituyen pruebas y artefactos reabiertos.

---

## 25. G-portafolio: publicación mínima verificable

G-portafolio es una publicación del **núcleo funcional headless**, no del microservicio completo. Puede declararse antes de G2/G3, siempre que el README y el manifiesto listen lo pendiente.

### 25.1 Criterios incluidos

| Criterio | Alcance exigido en portafolio | Evidencia de cierre |
| --- | --- | --- |
| AC-01 | Dataset local validado, deduplicado, reversos y corte consistente. | `T-DATA-001`, golden cases, conteos y hashes. |
| AC-02 | CLI termina con éxito o advertencias no críticas; funcionalidad y rendimiento se separan. | `T-E2E-CLI-001` funcional en `PASS` y `T-PERF-EVIDENCE-001`. |
| AC-03 | ABC exacto, incluidos acumulados, umbrales y empates. | `T-ABC-001`. |
| AC-04 | XYZ exacto, con ceros, CV muestral e historia insuficiente. | `T-XYZ-001`. |
| AC-05 | Hits multiubicación, líneas, unidades y reversos reconciliados. | `T-DATA-002`. |
| AC-06 | Clases, plan, rankings y hashes reproducibles con run IDs independientes. | `T-REPRO-001`. |
| AC-07 | Solo movimientos compatibles entran al plan. | `T-CONSTRAINT-001`. |
| AC-08 | Sin conflictos SKU/slot; límites, cooldown, dwell y desempates respetados. | `T-OPT-001`. |
| AC-09 | Demanda afectada, viajes dirigidos, traslado, costos y umbrales trazables. | `T-BENEFIT-001`. |
| AC-11 | Exportaciones ordenadas, dataset completo y manifiesto reutilizable. | `T-EXPORT-001`. |
| AC-13 | LPPH observado correcto y proyección `null` cuando falta evidencia. | `T-KPI-001`. |
| AC-19 | Instalación, agentes/Intake y demo limpios en modo `portfolio_mvp`. | `T-INSTALL-001`, `T-PERF-EVIDENCE-001` y guion de demo. |

Son exactamente **12 AC de portafolio**. `T-CRITICAL-PORTFOLIO` agrupa los **13 IDs atómicos** citados en §19; la diferencia entre cantidades se debe a que `T-PERF-EVIDENCE-001` aporta evidencia transversal de AC-02/AC-19. No se permiten omisiones, skips ni sustituciones por cobertura.

### 25.2 Regla funcional y de rendimiento

AC-02 se registra con dos estados independientes:

```text
functional_status = PASS | NOT_MET
performance_status = PASS | NOT_MET | NOT_COMPARABLE
```

`functional_status=PASS` admite `SUCCEEDED` o `SUCCEEDED_WITH_WARNINGS` con calidad no crítica y artefactos válidos; no exige propuestas. G-portafolio exige ese estado y evidencia de rendimiento válida. Admite `performance_status=NOT_MET` o `NOT_COMPARABLE` si la limitación aparece en README, reporte y manifiesto. Esto **no** cambia AC-02 del producto completo: para G3, un rendimiento distinto de `PASS` mantiene AC-02=`NOT_MET`.

### 25.3 Puerta de publicación

`G-PORTFOLIO=PASS` requiere simultáneamente:

1. `T-CRITICAL-PORTFOLIO` completo y verde.
2. Los doce criterios anteriores satisfechos en su alcance de portafolio.
3. Instalación limpia, demo y reproducción documentadas.
4. Exportaciones reabiertas, orden canónico y hashes verificados.
5. `run_manifest.json` con commit, entorno, configuración, comandos, resultados, fingerprint y SHA-256.
6. Benchmark con evidencia, sin ocultar umbral fallido ni hardware no comparable.
7. README con el distintivo textual constante:

```text
PORTFOLIO_BADGE_TEXT = "PORTFOLIO MVP — núcleo funcional verificado; producto completo pendiente"
```

AC-10, AC-12, AC-14, AC-15, AC-16, AC-17, AC-18 y AC-20 quedan fuera de la puerta y se marcan `PENDING_FULL`; no se afirma que sus funciones existan. Esta exclusión no relaja las reglas de seguridad, semántica o alcance de §24.

G-portafolio no autoriza textos como “microservicio completo”, “todos los AC superados”, “impacto LPPH demostrado” o “benchmark de 60 s cumplido” salvo evidencia específica y alcance correspondiente.

---

## 26. Configuración sintética inicial y ejemplos numéricos

Referencia ejecutable para construcción/pruebas. Valores sintéticos, no mediciones reales.

### 26.1 `analysis_config.demo.yaml`

```yaml
schema_version: "1.2"
project: slotting-optimizer-engine
mode: portfolio_mvp
data_mode: synthetic
seed: 20261027
analysis_timezone: UTC
data_cutoff_at: "2026-10-19T00:00:00Z"
knowledge_cutoff_at: "2026-10-19T00:00:00Z"
analysis_window_days: 84
weekly_history_required: 12
abc:
  a_threshold: 0.80
  b_threshold: 0.95
xyz:
  x_threshold: 0.50
  y_threshold: 1.00
classification:
  min_class_runs: 1  # PROHIBIDO FUERA DE data_mode=synthetic
  stability_lag_weeks: 4
travel:
  distance_mode: synthetic_euclidean
  is_directed: false
  picker_speed_m_s: 1.2
  pick_anchor: PACK-01
  require_measured_access_time: true
relocation:
  units_to_move_source: SNAPSHOT_ELSE_TARGET
  relocation_travel_mode: DISTANCE_BASED
  relocation_speed_m_s: 1.0
  relocation_travel_source: SYNTHETIC
benefit:
  horizon_days: 84
  min_net_benefit_s: 3600
  min_benefit_ratio: 1.0
  fixed_move_cost_s: 1800
  handling_cost_s_per_unit: 15
  disruption_cost_s: 1200
  plan_fixed_disruption_cost_s: 0
backtest:
  history_weeks: 16
  test_weeks: 4
  stability_lag_weeks: 4
  min_class_runs: 2
optimizer:
  algorithm: greedy_v0
  max_candidates_per_sku: 10
  max_moves_per_run: 20
  relocation_cooldown_days: 30
  min_dwell_days: 30
  recommendation_validity_hours: 24
lpph:
  attribution_factor: 0.50
  sensitivity_factors: [0.25, 0.50, 0.75]
  baseline_period_start: "2026-07-27T00:00:00Z"
  baseline_period_end: "2026-10-19T00:00:00Z"
  baseline_cohort_ids: [PICKERS-01]
  line_reconciliation_tolerance_abs: 0
  line_reconciliation_tolerance_pct: 0.0
source:
  mapping_file: contracts/source_mapping.demo.yaml
  snapshot_max_age_hours: 24
export:
  formats: [csv, json, html]
  canonical_float_decimals: 6
  canonical_rounding: ROUND_HALF_UP
```

`source_mapping.demo.yaml` es autoridad de columnas, claves, timezone y `period_granularity: WEEK`. Doce summaries semanales agregan exactamente el baseline de 84 días. `analysis_config.tz_demo.yaml` materializa la variante `America/Mexico_City`; no comparte corte ciego con UTC.

Hashes y fingerprints usan contenido canónico. Comentarios YAML, IDs y timestamps de ejecución no alteran `result_content_hash`.

### 26.2 Caso golden de beneficio

`SKU-AX-001` tiene 100 hits totales y **100 hits afectados en `A-99`**, todos de `PICKERS-01`, durante 84 días:

```text
projected_affected_hits = 100 × 84 / 84 = 100
```

Origen: 120 m en cada dirección y acceso 8 s. Destino `GD-01`: 20 m en cada dirección y acceso 4 s. Velocidad 1.2 m/s:

```text
pick_travel_s(actual) = (120 + 120) / 1.2 + 8 = 208.000000 s
pick_travel_s(destino) = (20 + 20) / 1.2 + 4 = 37.333333 s

gross_benefit_s = 100 × (208 - 37.333333...)
                = 17,066.666667 s
```

El snapshot completo vigente contiene 40 unidades del SKU en el primario y demuestra `GD-01` vacío. `target_stock_units=50`, por lo que capacidad requerida es 50; el destino tiene regla autorizada y dimensiones compatibles.

Para traslado, `A-99→GD-01=300 m` y `GD-01→A-99=300 m`, velocidad 1 m/s:

```text
relocation_travel_s = (300 + 300) / 1.0 = 600 s
relocation_cost_s = 1,800 + 40 × 15 + 600 = 3,000 s
net_benefit_s = 17,066.666667 - 3,000 - 1,200
              = 12,866.666667 s
              = 3.574074 h
benefit_ratio = 12,866.666667 / 3,000 = 4.288889
```

Supera los mínimos. El costo es reconstructible y la factibilidad no depende solo de volumen.

### 26.3 Caso golden de LPPH proyectado

Baseline `PICKERS-01`: 12,000 líneas completadas derivadas y reconciliadas, 100 horas ocupadas y 30 de viaje, agregadas sobre exactamente los mismos 84 días del beneficio. Las cohortes del plan y baseline coinciden; costo fijo del plan es cero:

```text
observed_LPPH = 12,000 / 100 = 120.000000
net_plan_benefit_s = 12,866.666667
attributed_savings_h = min(12,866.666667 / 3,600 × 0.50, 30)
                     = 1.787037 h
projected_busy_hours = 100 - 1.787037 = 98.212963 h
projected_LPPH = 12,000 / 98.212963 = 122.183464
```

Es proyección condicionada, no mejora realizada. Precisión interna completa; exportación decimal con `ROUND_HALF_UP` a seis posiciones. Si baseline fuese de siete días, cambiara de cohorte o no reconciliara líneas, `projected_LPPH=null` sin cambiar el beneficio.

### 26.4 Casos adversos obligatorios

| Caso | Resultado exigido |
| --- | --- |
| 100 hits totales, 20 en origen y 80 en otro slot | Beneficio calcula solo 20; los 80 no se reasignan. |
| Ida 100 m y retorno 160 m | Distancia de viaje 260 m; nunca 2×100. |
| Retorno inaccesible | `MISSING_DISTANCE`, sin recomendación. |
| Dos movimientos en un hit; uno totalmente revertido | Hit se conserva si el otro mantiene neto positivo. |
| Línea con actividad pero sin completion | `active_pick_lines`, no `completed_order_lines`. |
| Captura parcial sin filas del destino | `MISSING_DESTINATION_OCCUPANCY_EVIDENCE`, no “vacío”. |
| Volumen total suficiente, dimensión incompatible | Exclusión dimensional. |
| Dos cortes repetidos del mismo dataset | No satisfacen estabilidad. |
| Dos cortes distintos con clase diferente | `CLASS_NOT_STABLE`. |
| Beneficio 84 días vs. baseline 7 días | `LPPH_PERIOD_MISMATCH`, proyección `null`. |
| Dos aprobaciones concurrentes por el mismo slot | Una reserva gana; la otra recibe `SLOT_RESERVATION_CONFLICT`. |

### 26.5 Reconciliación final

El reporte muestra por separado hits totales/afectados, bruto 17,066.666667 s, reubicación 3,000 s, disrupción 1,200 s, neto 12,866.666667 s, capacidad 50, LPPH observado 120.000000 y proyectado 122.183464 bajo supuestos alineados. No publica impacto realizado antes de aplicación y medición.

Estos casos son golden tests de fórmulas, contratos y presentación; no evidencia de optimalidad, desempeño ni causalidad.

---

## 27. Changelog unificado de auditorías — v1.1

La v1.1 conserva los cálculos validados de §26 y cierra las dos auditorías sobre la versión modificada por el usuario. No se adoptaron recomendaciones que habrían introducido swaps, multi-slot o escritura WMS. Las precisiones de dataset de la segunda auditoría se identifican aquí como PD-01 a PD-03 para hacerlas verificables.

| Hallazgo | Cierre incorporado |
| --- | --- |
| B-01 | §11 fija 16 semanas de historia + 4 de prueba y dos cortes de 12 semanas para cumplir XYZ y estabilidad. |
| B-02 | §19 se convierte en catálogo canónico `test_id → propósito → evidencia → AC → nivel`, con suites cerradas. |
| B-03 | §7, §9, §14 y AC-02 definen `INSUFFICIENT_CLASS_RUNS`, éxito con advertencias y cero propuestas válido. |
| B-04 | §6.5 y §18 añaden distancias y `picker_time_summaries`; instalación/exportación los reabren. |
| A-01 | §5.2 y §9 exigen destino vacío; el mismo SKU ocupándolo también es exclusión. |
| A-02 | §8.3 define `units_to_move`, política `SNAPSHOT_ELSE_TARGET`, marca de estimación y revalidación. |
| A-03 | §5.3 y §7.1 definen hit por `(pick_task_id, location_id)` y prueba multiubicación. |
| A-04 | §5.5 selecciona el snapshot más reciente válido con desempate por `snapshot_id`. |
| A-05 | §8.1 exige acceso conocido; el desconocido excluye y no se reemplaza por cero. |
| A-06 | §6.2 cierra el schema de `compatibility` y sus enums. |
| A-07 | §14 y AC-02 aceptan `SUCCEEDED_WITH_WARNINGS` solo sin calidad crítica. |
| A-08 | §5.6 fija origen, destino, actor y condición de cada transición. |
| A-09 | §9 define umbrales y cambios físicos que causan `EXPIRED`. |
| A-10 | §8 renombra la función a `pick_travel_s(l)` en fórmula y beneficio. |
| M-01 | §10 y §26 usan `net_plan_benefit_s` como techo conservador. |
| M-02 | §10 añade guarda explícita de horas proyectadas no positivas. |
| M-03 | §22 y §26 separan configuración demo y rechazan estabilidad relajada fuera de datos sintéticos. |
| M-04 | §26 declara corte sintético UTC y `T-INSTALL-001` lo verifica. |
| M-05 | §19 lista los 21 tests atómicos de `T-CRITICAL-001`. |
| M-06 | §25 distingue 12 AC y 13 pruebas atómicas de portafolio. |
| M-07 | AC-05 y `T-DATA-002` se incorporan a G-portafolio. |
| M-08 | §6.1/§6.2 fijan cohorte, periodo exclusivo y granularidad DAY/WEEK. |
| M-09 | §9 define cooldown por `(sku_id, to_location_id)` desde aplicación externa. |
| M-10 | §9 exime dwell solo cuando no existe historial de slotting del SKU. |
| L-01 | §12 conserva el término correcto “Hashes”; no había que alterar el contenido. |
| L-02 | §5.3 equipara reversos acumulados al total a reverso total y el exceso a error crítico. |
| L-03 | §6.1/§6.2 añaden `directionality_justification` y reglas para distancias dirigidas. |
| L-04 | §7.1 y `T-ABC-001` cubren un único SKU A y clases B/C vacías. |
| L-05 | §17.1 declara RSS informativo, sin umbral de aceptación. |
| L-06 | §14 y §22 normalizan IDs/timestamps y comparan `result_content_hash`. |
| L-07 | §25 fija `PORTFOLIO_BADGE_TEXT` como constante textual. |
| HC-01 | §5.1/§5.5 limitan `current_on_hand_units` al pick-face primario y excluyen reserva. |
| HC-02 | §6.1/§11 exigen versiones `as_of`; layout actual en backtest queda expresamente limitado. |
| HC-03 | §14 separa UUID v4 de ejecución, fingerprint de contenido e idempotencia HTTP. |
| HD-01 | §9 y §15 añaden `golden_zone_misallocation.csv` sin acciones automáticas. |
| HD-02 | §8.3 separa disrupción marginal por movimiento y costo fijo del plan. |
| PD-01 | §5.4/§6.5 y `T-DATA-002` cubren límites ISO y `America/Mexico_City`. |
| PD-02 | §5.3/§10 y `T-KPI-001` fijan el tratamiento de líneas con reverso parcial/total. |
| PD-03 | §14, `T-EXPORT-001` y `T-REPRO-001` fijan orden canónico explícito por exportación. |

**Resultado documental de v1.1:** se incorporaron correcciones para 4 bloqueantes, 10 hallazgos altos, 10 medios, 7 documentales, 3 críticos de datos, 2 de diseño y 3 precisiones de dataset. “Incorporada” describió el cambio de texto, nunca la ejecución de pruebas; la verificación operativa permaneció condicionada a evidencia.

---

## 28. Integración final de auditorías y estados de cierre

### 28.1 Fuentes y deduplicación

| Fuente | SHA-256 | Uso |
| --- | --- | --- |
| `Dictamen REQUIERE.txt` | `98403fc1f9d364aea3ccdb190e26eb7b5b76b6666fded9b178f293a32035dbf2` | H-01…H-14 y gobierno D-01…D-04. |
| `Auditoría de la v1.txt` | `fd2574164d0b66e6c6ce24827f9b1ad83731ab1633a44631fe2b6a1b4149be14` | N-01…N-10. |
| `Auditoría de la v1(1).txt` | Igual al anterior | Duplicado byte a byte; no duplica hallazgos. |
| `Auditoría de la ve.txt` | `236359d7bb0e12f9a13ae9b817f0248f1dd1c3e540621f94c6ba49371286751c` | CR, AH, MH, LB y alias AD. |

Las fuentes se conservarán en `docs/audits/` durante la construcción. La tabla siguiente enlaza hallazgo, modificación y prueba. **INCORPORATED** significa corrección escrita en esta versión; **VERIFIED** solo podrá marcarse cuando la prueba indicada tenga evidencia `PASS`.

### 28.2 Matriz final hallazgo → cierre → verificación

| Hallazgo | Corrección incorporada | Verificación requerida |
| --- | --- | --- |
| H-01 | Beneficio limitado a hits afectados del origen. | `T-BENEFIT-001`, caso 20/80. |
| H-02 | Viaje suma outbound+inbound y exige retorno. | `T-BENEFIT-001`, asimetría/inaccesibilidad. |
| H-03 | Fuente única `sku_primary_pick_face`. | `T-DATA-001`, `T-CONSTRAINT-001`. |
| H-04 | Snapshot set completo, cobertura y ausencia contratados. | `T-CONSTRAINT-001`, capturas adversas. |
| H-05 | Completion autoritativa; actividad parcial tiene otro nombre. | `T-DATA-002`, `T-KPI-001`. |
| H-06 | Reconciliación movimiento→hit→línea y restatement `as_of`. | `T-DATA-002`, `T-BACKTEST-001`. |
| H-07 | Periodo, cohortes y actividad deben alinearse para LPPH. | `T-KPI-001`, mismatch 84/7. |
| H-08 | Historia operativa versionada y estabilidad temporal persistente. | `T-REPRO-001`, `T-BACKTEST-001`. |
| H-09 | Capacidad acotada y traslado reconstructible. | `T-CONSTRAINT-001`, `T-BENEFIT-001`. |
| H-10 | Reserva transaccional única de SKU/slot. | `T-WORKFLOW-001`, aprobaciones concurrentes. |
| H-11 | Token, revisión, claim, vigencia y progreso parcial. | `T-WORKFLOW-001`, claim/cancelación. |
| H-12 | `recorded_at` y reconstrucción bitemporal de conocimiento. | `T-BACKTEST-001`, evento/maestro tardío. |
| H-13 | Idempotencia actor+operación+clave; payload almacenado para comparar. | `T-JOB-001`, solicitudes simultáneas. |
| H-14 | Leases, reintentos y publicación atómica de artefactos. | `T-JOB-001`, caídas antes/después del marker. |
| D-01 | Fuentes, hashes y estados INCORPORATED/VERIFIED explícitos. | Revisión documental y `T-INSTALL-001`. |
| D-02 | Autoridad brief→contratos→configuración→pruebas. | `T-INSTALL-001`, control de divergencias. |
| D-03 | `AGENTS.md`, `CLAUDE.md` e Intake enlazan reglas canónicas. | `T-INSTALL-001`, enlaces y carga registrada. |
| D-04 | Controles MVP de exclusiones, secretos y alcance asignados. | `T-E2E-CLI-001`, `T-EXPORT-001`, `T-INSTALL-001`. |
| N-01 | CSV canónico sin run/job IDs; evidencia operativa separada. | `T-REPRO-001`, `T-EXPORT-001`. |
| N-02 | Fuentes de primario y acceso añadidas a contratos/demo. | `T-DATA-001`, `T-CONSTRAINT-001`. |
| N-03 | Cambio no material conserva estado y crea versión enlazada. | `T-WORKFLOW-001`. |
| N-04 | Historia insuficiente mapeada a warning y AC-12 `NOT_MET`. | `T-BACKTEST-001`. |
| N-05 | Vigencia de `DEFERRED→PROPOSED` hecha verificable. | `T-WORKFLOW-001`. |
| N-06 | AC-19 incluye evidencia benchmark; AC-05 incluye cortes. | `T-PERF-EVIDENCE-001`, `T-DATA-002`. |
| N-07 | Cortes separados por lag; solapamiento permitido sin repetir corte. | `T-BACKTEST-001`. |
| N-08 | Absorbido por N-01; `recommendation_id` permanece en CSV. | `T-EXPORT-001`, `T-REPRO-001`. |
| N-09 | Read-only local por hash y DB por grants. | `T-DATA-001`. |
| N-10 | Fixtures/config propios para `America/Mexico_City`. | `T-DATA-002`. |
| CR-01 | Primario autoritativo, vigente y único. | `T-CONSTRAINT-001`. |
| CR-02 | Acceso versionado con fuente y derivación de flag. | `T-CONSTRAINT-001`, `T-BENEFIT-001`. |
| CR-03 | Vacío solo por evidencia completa; ausencia no es cero implícito. | `T-CONSTRAINT-001`. |
| CR-04 | Traslado por distancia dirigida o valor configurado declarado. | `T-BENEFIT-001`. |
| AH-01 | Fórmula dirigida sin `round_trip_factor`. | `T-BENEFIT-001`. |
| AH-02 | Agregación neta por movimiento, hit y línea. | `T-DATA-002`. |
| AH-03 | Completions autoriza líneas, summaries horas y reconciliación. | `T-KPI-001`. |
| AH-04 | Dijkstra/ac equivalente, pesos positivos y desempate por edges. | `T-DATA-001`, `T-BENEFIT-001`. |
| AH-05 | Corte ISO local con conversión UTC en cada límite. | `T-DATA-002`, DST/timezones. |
| MH-01 | `rank` es secuencia de aceptación desde 1. | `T-OPT-001`, `T-EXPORT-001`. |
| MH-02 | Base de capacidad cero se excluye. | `T-CONSTRAINT-001`. |
| MH-03 | Validación cerrada de parámetros económicos. | `T-DATA-001`, `T-BENEFIT-001`. |
| MH-04 | Backtesting estándar fija `min_class_runs=2`. | `T-BACKTEST-001`. |
| MH-05 | Cohortes no solapadas y agregación por sumas. | `T-KPI-001`. |
| MH-06 | `source_mapping` es autoridad única de granularidad. | `T-DATA-001`, `CONFIG_CONFLICT`. |
| MH-07 | Misallocation incluido explícitamente en entregable MVP. | `T-EXPORT-001`. |
| MH-08 | Registro ordenado de `primary_reason`. | `T-ELIGIBILITY-001`; control básico `T-E2E-CLI-001`. |
| LB-01 | Namespace UUIDv5 constante y versionado. | `T-REPRO-001`, manifiesto. |
| LB-02 | SHA-256, orden, NFC, UTF-8 y separador `0x1F`. | `T-EXPORT-001`, `T-REPRO-001`. |
| LB-03 | Una línea pertenece a un único SKU/almacén. | `T-DATA-002`. |
| LB-04 | `travel_seconds_per_hit` reutiliza `pick_travel_s(l)`. | `T-KPI-001`. |
| LB-05 | Euclídea sintética siempre simétrica/no dirigida. | `T-BENEFIT-001`. |
| LB-06 | Decimal canónico con `ROUND_HALF_UP`. | `T-EXPORT-001`. |
| LB-07 | `valid_until` como mínimo de vigencias y TTL. | `T-ELIGIBILITY-001`, `T-WORKFLOW-001`. |
| LB-08 | `work_sequence=rank`. | `T-OPT-001`, `T-EXPORT-001`. |
| AD-01 | Alias de CR-01. | Misma verificación CR-01. |
| AD-02 | Alias de CR-02. | Misma verificación CR-02. |
| AD-03 | Alias de CR-03. | Misma verificación CR-03. |
| AD-04 | Alias de CR-04. | Misma verificación CR-04. |
| AD-05 | Alias de AH-01. | Misma verificación AH-01. |
| AD-06 | Alias de AH-02. | Misma verificación AH-02. |
| AD-07 | Alias de AH-05. | Misma verificación AH-05. |
| AD-08 | Alias conjunto de MH-01 y MH-08. | Verificaciones de ambos. |

### 28.3 Declaración final

Las 61 referencias anteriores quedan **INCORPORATED** en la especificación 1.2, con alias deduplicados y sin ampliar Fase 4. El número 61 cuenta filas/referencias trazadas, no hallazgos únicos: existe cobertura redundante entre auditorías, además de los alias AD explícitos. La versión final puede congelar contratos para construcción. Ninguna fila se marca VERIFIED por esta edición documental: ese estado exige los resultados, comandos, hashes y artefactos de las pruebas correspondientes.

---

**Cierre normativo:** especificación final lista para construir y congelar contratos. El software se declara entregado únicamente cuando el alcance elegido aporta pruebas `PASS`, artefactos publicados, hashes y estados exigidos.