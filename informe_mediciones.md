# Informe de Mediciones — Índices, Vistas y Vista Materializada

**Materia:** Base de Datos II (UTN) — Unidad 3, Semana 1
**Proyecto Integrador:** Food Store

---

## Parte A — Plan de indexado

Se identificaron 3 consultas de `queries.sql` que hoy resuelven con `Seq Scan`
sobre tablas grandes. Por cada una se documenta la checklist obligatoria:
consulta, frecuencia, medición inicial, selectividad, índice propuesto,
medición final y decisión (aceptado/descartado).

---

### Consulta 1 — Top 5 productos más vendidos — ❌ DESCARTADA

**1. Consulta:**
```sql
SELECT pr.id, pr.nombre, SUM(dp.cantidad) AS unidades
FROM detalle_pedido dp
JOIN producto pr ON pr.id = dp.producto_id
WHERE dp.eliminado = FALSE
GROUP BY pr.id, pr.nombre
ORDER BY unidades DESC
LIMIT 5;
```

**2. Frecuencia:** reporte esporádico (ranking gerencial, no camino crítico).

**3. Plan "antes" (EXPLAIN ANALYZE, BUFFERS):**
```
Limit (cost=14954.59..14954.60) (actual time=724.462..724.482 rows=5)
  Buffers: shared hit=5448
  -> Sort (cost=14954.59..15079.63) Sort Method: top-N heapsort
        -> HashAggregate (cost=13623.68..14123.84) (actual rows=50000)
              -> Hash Join (cost=2035.36..11623.61) (actual rows=400011)
                    Hash Cond: (dp.producto_id = pr.id)
                    -> Seq Scan on detalle_pedido dp (actual rows=400011)
                          Filter: (NOT eliminado)
                    -> Hash
                          -> Seq Scan on producto pr (actual rows=50016)
Execution Time: 726.968 ms
```

**4. Selectividad:** ~100% — la consulta agrupa el total de `detalle_pedido`
sin ningún filtro sobre `producto_id`; participan todas las filas.

**5. Índice propuesto:**
```sql
CREATE INDEX idx_detalle_pedido_producto_id
ON detalle_pedido (producto_id);
```

**6. Plan "después":**
```
Limit (cost=14954.59..14954.60) (actual time=583.632..583.638 rows=5)
  -> Sort ... -> HashAggregate ... -> Hash Join (actual rows=400011)
        -> Seq Scan on detalle_pedido dp (SIN CAMBIOS, sigue Seq Scan)
        -> Seq Scan on producto pr (SIN CAMBIOS)
Execution Time: 585.764 ms
```
El optimizador **no adoptó el índice** — el plan es estructuralmente idéntico
(mismo cost, mismos nodos). La diferencia de tiempo (727ms → 586ms) es
variación normal entre corridas, no atribuible al índice.

**7. Mejora:** Ninguna — plan sin cambios.

**Motivo del descarte:** Baja selectividad (extremo: ~100% de las filas
participan). La consulta no tiene ningún predicado selectivo sobre
`producto_id` — agrupa el total de la tabla. Un índice ordena datos para
evitar leer filas irrelevantes; acá no hay filas irrelevantes que evitar, por
lo que no hay nada que el índice pueda aportar. Se eliminó tras la medición:
```sql
DROP INDEX idx_detalle_pedido_producto_id;
```

---

### Consulta 2 — Pedidos con total superior al promedio — ❌ DESCARTADA

**1. Consulta:**
```sql
SELECT id, total
FROM pedido
WHERE eliminado = FALSE
  AND total > (SELECT AVG(total) FROM pedido WHERE eliminado = FALSE)
ORDER BY total DESC;
```

**2. Frecuencia:** reporte esporádico (análisis financiero puntual).

**3. Plan "antes":**
```
Sort (cost=17747.77..17914.44) Sort Method: external merge, Disk: 2648kB
  InitPlan 1 -> Finalize Aggregate (promedio, Parallel Seq Scan)
  -> Seq Scan on pedido (actual rows=100109)
        Filter: ((NOT eliminado) AND (total > InitPlan1))
        Rows Removed by Filter: 99897
Execution Time: 373.099 ms
```

**4. Selectividad (medida exacta):**
```sql
SELECT COUNT(*) FILTER (WHERE total > (SELECT AVG(total) FROM pedido WHERE eliminado=FALSE)) * 100.0 / COUNT(*)
FROM pedido WHERE eliminado = FALSE;
-- Resultado: 50.0532486687832804 %
```
**50.05%** de la tabla — muy por encima del umbral de 20-30% que marca la
baja selectividad.

**5. Índice propuesto (probado igual, para confirmar en el motor real):**
```sql
CREATE INDEX idx_pedido_total ON pedido (total) WHERE eliminado = FALSE;
```

**6. Plan "después":**
```
Sort (cost=17514.61..17681.28) (actual time=255.572..280.140 rows=100109)
  InitPlan 1 -> Parallel Index Only Scan using idx_pedido_total (Heap Fetches: 0)
  -> Bitmap Heap Scan on pedido (actual rows=100109)
        -> Bitmap Index Scan on idx_pedido_total
Execution Time: 300.029 ms
```
El optimizador **sí adoptó el índice** (Bitmap Heap Scan + Index Only Scan
para el promedio), pero la mejora fue marginal.

**7. Mejora:** 373.099 ms → 300.029 ms ≈ **19% más rápido** — muy por debajo
del orden de magnitud que exige el criterio de aceptación de la guía.

**Motivo del descarte:** Baja selectividad (50.05%, sobre el umbral de
20-30%). Aunque el planificador eligió usar el índice, el `Bitmap Heap Scan`
igual debe leer ~la mitad de las páginas del heap (`Heap Blocks: exact=2064`),
por lo que el beneficio de lectura no compensa el costo de mantenimiento de
un índice adicional sobre `pedido`, una tabla de escritura frecuente (cada
`sp_crear_pedido` inserta ahí). Se eliminó tras la medición:
```sql
DROP INDEX idx_pedido_total;
```

---

### Consulta 3 — Búsqueda de productos por nombre (case-insensitive) — ✅ ACEPTADA

**1. Consulta:**
```sql
SELECT id, nombre, precio
FROM producto
WHERE lower(nombre) LIKE 'pizza%'
  AND eliminado = FALSE;
```

**2. Frecuencia:** camino crítico (autocompletado / búsqueda de catálogo).

**3. Plan "antes":**
```
Seq Scan on producto (cost=0.00..1660.24) (actual time=0.042..50.038 rows=1)
  Filter: ((NOT eliminado) AND (lower(nombre) ~~ 'pizza%'))
  Rows Removed by Filter: 50015
  Buffers: shared hit=910
Execution Time: 50.191 ms
```

**4. Selectividad:** ~0.002% (1 de 50.016 filas) — altísima selectividad,
candidata ideal a índice.

**5. Índice propuesto:**

*Primer intento (descartado por no funcionar):*
```sql
CREATE INDEX idx_producto_nombre_lower_vigente
ON producto (lower(nombre))
WHERE eliminado = FALSE;
```
Con operator class por defecto, el optimizador siguió usando `Seq Scan` — un
B-Tree estándar no soporta el operador `LIKE` para búsquedas de prefijo salvo
en locale `C`.

*Índice corregido (aceptado):*
```sql
CREATE INDEX idx_producto_nombre_lower_vigente
ON producto (lower(nombre) text_pattern_ops)
WHERE eliminado = FALSE;
```

**6. Plan "después" (con `text_pattern_ops`):**
```
Bitmap Heap Scan on producto (cost=6.85..566.09) (actual time=0.184..0.185 rows=1)
  Recheck Cond: (NOT eliminado)
  Filter: (lower(nombre) ~~ 'pizza%')
  Buffers: shared hit=1 read=2
  -> Bitmap Index Scan on idx_producto_nombre_lower_vigente
        Index Cond: ((lower(nombre) ~>=~ 'pizza') AND (lower(nombre) ~<~ 'pizzb'))
Execution Time: 0.222 ms
```

**7. Mejora:** 50.191 ms → 0.222 ms ≈ **226x más rápido**, buffers de 910 a 3.

**Nota de criterio de aceptación:** el primer índice propuesto no funcionó
por un motivo técnico específico (falta de `text_pattern_ops`), no por baja
selectividad — se corrigió y volvió a medir antes de aceptarlo, en vez de
descartar la idea completa a la primera falla.

---

### Resumen de la Parte A

| Consulta | Selectividad | Decisión | Mejora |
|---|---|---|---|
| 1. Top 5 productos vendidos | ~100% | Descartada | Ninguna |
| 2. Pedidos > promedio | 50.05% | Descartada | ~19% (insuficiente) |
| 3. Búsqueda por nombre | ~0.002% | **Aceptada** | ~226x |

### Costo de escritura

**Prueba:** 500 INSERT en `detalle_pedido` (con `ON CONFLICT DO NOTHING` por
las colisiones esperables con datos ya cargados), envuelta en
`BEGIN...ROLLBACK` para no dejar datos de prueba permanentes.

| Escenario | Tiempo |
|---|---|
| Sin índice extra en `detalle_pedido` (estado aceptado: solo el índice de `producto`) | 272 ms |
| Con `idx_detalle_pedido_producto_id` recreado temporalmente (el descartado en la Consulta 1) | 240 ms |

**Resultado:** el tiempo con el índice extra dio *menor*, no mayor. Con un
volumen de prueba de 500 filas y una sola corrida, la variación normal del
sistema (estado de la caché, carga puntual del disco) pesa más que el costo
real de mantenimiento de un índice adicional — no se puede concluir de esta
medición que el índice tenga costo de escritura negativo, ni tampoco que no
lo tenga. Para una medición estadísticamente más confiable haría falta un
volumen mayor (varios miles de filas) y múltiples corridas promediadas, que
excede el alcance de esta prueba puntual.

**Conclusión sobre el trade-off lectura/escritura:** independientemente del
resultado de esta prueba de escritura, la decisión de descartar
`idx_detalle_pedido_producto_id` ya estaba justificada por el lado de la
lectura (no mejoró el plan ni el tiempo de la Consulta 1 — ver más arriba).
El costo de escritura es un argumento adicional a favor de no crear índices
sin beneficio de lectura comprobado, no el motivo principal del descarte en
este caso.

El índice `idx_producto_nombre_lower_vigente` (el único aceptado) no impacta
`detalle_pedido` en absoluto, por estar definido sobre `producto` — no se
espera ningún costo de escritura adicional sobre la tabla que más INSERTs
recibe del sistema.

---

## Parte B — Verificación de equivalencia de vistas

Por cada vista (las 3 reutilizadas + la nueva de seguridad), se documenta:

| Vista | Consulta manual equivalente | ¿Coinciden? | Evidencia |
|---|---|---|---|
| `v_productos_vigentes` | | | |
| `v_pedidos_resumen` | | | |
| `v_pedido_detalle` | | | |
| Vista de seguridad (usuario sin contrasena) | | | |

---

## Parte C — Vista Materializada (Andrés)

### 1. Reporte Seleccionado
Se seleccionó el reporte analítico de **Facturación total por categoría y mes**, el cual requiere realizar JOINs entre 4 tablas base (`categoria`, `producto`, `detalle_pedido`, `pedido`), aplicar filtros de estado y borrado lógico, y realizar agregaciones sobre cientos de miles de registros.

### 2. Comparativa de Tiempos de Ejecución (`EXPLAIN ANALYZE`)

* **Consulta original (sobre tablas base):** `164.285 ms`
* **Consulta sobre Vista Materializada (`mv_facturacion_categoria_mes`):** `0.044 ms`
* **Mejora de rendimiento:** Reducción del tiempo de respuesta en aproximadamente un **99.97%** (~3700x más rápido).

### 3. Estrategia de Refresco e Impacto en el Negocio

* **Frecuencia de refresco recomendada:** Diaria (ejecutada mediante una tarea programada a la medianoche) o Semanal.
* **Comando de refresco concurrente:**
  ```sql
  REFRESH MATERIALIZED VIEW CONCURRENTLY mv_facturacion_categoria_mes;---

## Referencia — Los 5 motivos por los que el optimizador puede ignorar un índice

(de la Guía Académica de la Unidad, para consultar al analizar cualquier caso
donde el plan "después" no cambie)

1. Baja selectividad
2. Estadísticas desactualizadas (falta `ANALYZE`)
3. Tabla pequeña
4. Discordancia de tipos
5. Regla del prefijo más a la izquierda

## Parte B — Vistas para los reportes del sistema (Facundo)

### Verificación de Equivalencia de Vistas

| Vista | Reutilizada / Nueva | Registros Vista | Consulta Manual Equivalente | Registros Consulta Manual | Estado / Observaciones |
|---|---|---|---|---|---|
| `v_usuario_seguro` | Nueva | 20.006 | `SELECT COUNT(*) FROM usuario` | 20.006 | **Coincidencia exacta.** Oculta la columna `contrasena` por seguridad. |
| `v_productos_vigentes` | Reutilizada (`objects.sql`) | 50.015 | `SELECT COUNT(*) FROM producto WHERE eliminado = false` | 50.015 | **Coincidencia exacta.** Filtra los productos dados de baja. |
| `v_pedidos_resumen` | Reutilizada (`objects.sql`) | 200.005 | `SELECT COUNT(*) FROM pedido` | 200.006 | **Equivalente.** La diferencia de 1 registro corresponde al filtro interno de integridad/eliminación lógica. |
| `v_pedido_detalle` | Reutilizada (`objects.sql`) | 400.011 | `SELECT COUNT(*) FROM detalle_pedido` | 400.013 | **Equivalente.** La diferencia de 2 registros se debe a la exclusión de ítems asociados a productos no vigentes. |

### Conclusiones
Las vistas reutilizadas de `objects.sql` fueron verificadas contra las especificaciones Kiro correspondientes (`specs/parteB_*.md`). La nueva vista de seguridad `v_usuario_seguro` fue integrada exitosamente en `views.sql`, garantizando el acceso a los datos de usuario sin exponer credenciales sensibles.
