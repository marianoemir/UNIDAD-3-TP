# Food Store — TP: Índices, vistas y vistas materializadas (Base de Datos II)

Proyecto integrador de un sistema de venta de comida, implementado en PostgreSQL.
Este repositorio contiene el Trabajo Práctico de la **Unidad 3, Semana 1** —
*Índices, vistas y vistas materializadas en Food Store* — resuelto en grupo
de 3 integrantes con OpenCode y Kiro como herramientas de IA.

## Grupo

Grupo 10

**Integrantes:**
- Mariano Chirino
- Andrés Fabre
- Facundo Quiroga

## Repositorio

**Link:** https://github.com/marianoemir/UNIDAD-3-TP

## Requisitos previos (heredados de semanas anteriores)

Este TP no vuelve a crear el modelo de datos ni a poblar la base desde cero:
parte de la `copia_trabajo` tal como quedó al cierre de la Semana 3/4, y solo
agrega índices y vistas nuevas sobre esa base, sin alterar tablas ni
restricciones.

| Archivo | Contenido |
|---|---|
| `archivos necesarios para la BD/schema.sql` | Tipos ENUM, tablas, constraints e índices base (`idx_producto_categoria_id`, `idx_pedido_usuario_id`, `idx_producto_nombre_vigente`) |
| `archivos necesarios para la BD/objects.sql` | Vistas base (`v_categorias_vigentes`, `v_productos_vigentes`, `v_pedidos_resumen`, `v_pedido_detalle`), función de cálculo, triggers y procedimiento `sp_crear_pedido` |
| `archivos necesarios para la BD/data.sql` | Datos de prueba chicos (categorías, productos, usuarios, pedidos) |
| `archivos necesarios para la BD/carga_masiva.sql` | Población masiva: 20.000 usuarios, 50.000 productos, 200.000 pedidos y ~400.000 detalles, necesaria para que las diferencias de plan sean observables |
| `archivos necesarios para la BD/indices_semana3.sql` | Índices creados y medidos en la Semana 3 (`idx_pedido_fecha_estado_vigente`, `idx_pedido_usuario_total_estado`), dados por existentes en este TP |
| `archivos necesarios para la BD/transacciones.sql` | Transacciones de referencia de entregas anteriores |
| `archivos necesarios para la BD/queries.sql` | Historias de usuario resueltas y consultas analíticas — banco de consultas candidatas para la Parte A de este TP |

**Orden de ejecución (heredado, no forma parte de esta entrega):**
`schema.sql → objects.sql → data.sql → carga_masiva.sql → indices_semana3.sql`

> Nota: el índice de producto/categoría propuesto en la Semana 3 no se
> recreó — el optimizador lo ignoró y no produjo ninguna mejora medible (ver
> la entrega de esa semana).

## Entregables de este TP

| Archivo | Qué contiene |
|---|---|
| `indices.sql` | Parte A — el índice finalmente aceptado (`idx_producto_nombre_lower_vigente`, con `text_pattern_ops`), comentado con su justificación. Los dos descartados por sobreindexación no se incluyen acá (quedan documentados en `informe_mediciones.md`) |
| `views.sql` | Parte B — la vista nueva de seguridad `v_usuario_seguro` (usuario sin `contrasena`). Las otras 3 vistas de reporte (`v_productos_vigentes`, `v_pedidos_resumen`, `v_pedido_detalle`) ya existían en `objects.sql` de la entrega anterior; en este TP se verificaron formalmente contra sus specs (`specs/parteB_*.md`) y contra su consulta manual equivalente, en lugar de recrearlas, para no duplicar objetos ya vigentes en la base |
| `materializadas.sql` | Parte C — vista materializada `mv_facturacion_categoria_mes` (`WITH DATA`) más su índice único `idx_mv_facturacion_cat_mes_pk` sobre `(categoria_id, mes)`, pensado para permitir a futuro `REFRESH CONCURRENTLY` |
| `specs/` | Especificaciones de Kiro, una por cada índice, vista o vista materializada, redactadas antes de generar el SQL con OpenCode |
| `informe_mediciones.md` | `EXPLAIN ANALYZE` antes/después de cada consulta candidata de la Parte A, medición del costo de escritura, verificación de equivalencia de las vistas de la Parte B, y comparación de tiempos y estrategia de refresco de la Parte C |
| `duia/` | Declaración de Uso de IA, un archivo por integrante, con el detalle de cada interacción relevante con Kiro/OpenCode: qué se pidió, qué propuso la IA, qué se aceptó, modificó o descartó, y por qué |
| `protocolo_seguridad.md` | Protocolo obligatorio de copia de trabajo, transacción reversible, respaldo previo a cambios estructurales, `ANALYZE` antes de medir y verificación de equivalencia de vistas |

### Quién hizo qué

| Parte | Integrante | Contenido |
|---|---|---|
| Parte A — Plan de indexado asistido por IA | Mariano Chirino | 3 consultas candidatas evaluadas con spec, `EXPLAIN ANALYZE` antes/después y medición de escritura; 2 descartadas por sobreindexación (sin filtro selectivo, y selectividad ~50% con mejora insuficiente) y 1 aceptada (búsqueda de productos por nombre, ~226x más rápida con `text_pattern_ops`) |
| Parte B — Vistas para los reportes del sistema | Facundo Quiroga | Specs de las 4 vistas de reporte (corrigiendo nombres de columna generados por la IA que no coincidían con el esquema real), vista nueva de seguridad `v_usuario_seguro` (oculta `contrasena`), y verificación de equivalencia de las 4 vistas contra su consulta manual directa |
| Parte C — Vista materializada | Andrés Fabre | Vista materializada `mv_facturacion_categoria_mes` con índice único, comparación de tiempos contra la consulta original sobre tablas base (~3700x más rápida), estrategia de refresco e implicancias de la desactualización para los usuarios |

El detalle completo de cada parte (spec usado, qué generó la IA, qué se
aceptó o descartó y por qué, y la verificación sobre el motor real) está en
`informe_mediciones.md` y en el `duia/` de cada integrante.

## Cómo reproducir las pruebas de este trabajo

```bash
# 1. Crear la plantilla con el esquema y el seed chico (si no existe ya)
createdb plantilla_food_store
psql -d plantilla_food_store -f "archivos necesarios para la BD/schema.sql"
psql -d plantilla_food_store -f "archivos necesarios para la BD/objects.sql"
psql -d plantilla_food_store -f "archivos necesarios para la BD/data.sql"

# 2. Crear una copia de trabajo descartable a partir de la plantilla
createdb -T plantilla_food_store copia_trabajo

# 3. Aplicar los prerrequisitos heredados (carga masiva e índices de la Semana 3)
psql -d copia_trabajo -f "archivos necesarios para la BD/carga_masiva.sql"
psql -d copia_trabajo -f "archivos necesarios para la BD/indices_semana3.sql"

# 4. Aplicar los objetos nuevos de este TP
psql -d copia_trabajo -f "indices.sql"
psql -d copia_trabajo -f "views.sql"
psql -d copia_trabajo -f "materializadas.sql"
```

Para repetir las mediciones de `informe_mediciones.md`:

```sql
-- Parte A: antes/después de cada índice (ver specs/parteA_*.md por la consulta exacta)
EXPLAIN (ANALYZE, BUFFERS) <consulta candidata>;

-- Después de crear cada índice, siempre actualizar estadísticas antes de volver a medir
ANALYZE <tabla>;

-- Parte C: comparar la consulta original contra la vista materializada
EXPLAIN ANALYZE SELECT * FROM mv_facturacion_categoria_mes;
```

Antes de aplicar cualquier cambio sobre la base se sigue el flujo de
`protocolo_seguridad.md`: nunca se trabaja sobre `plantilla_food_store`
directamente, siempre sobre una copia descartable (`copia_trabajo`); se prueba
primero dentro de `BEGIN...ROLLBACK`; se saca respaldo (`pg_dump`) antes de
cualquier cambio estructural (`CREATE INDEX`, `CREATE VIEW`,
`CREATE MATERIALIZED VIEW`); se corre `ANALYZE` después de cualquier índice
nuevo, antes de medir con `EXPLAIN ANALYZE`; y cada vista se valida contra su
consulta manual equivalente antes de darla por válida.

## Criterio de aceptación

Ninguna propuesta de la IA se aplicó "porque lo dijo la IA": cada índice se
aceptó solo después de medir el plan antes y después con `EXPLAIN ANALYZE` y
de poder explicar por qué mejoraba (o no). Se documentaron también los casos
en los que el optimizador ignoró el índice propuesto o la mejora fue
insuficiente (Parte A), en vez de descartarlos en silencio, y cada vista se
verificó fila por fila contra su consulta manual equivalente antes de darla
por válida (Parte B).