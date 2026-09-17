# Food Store — Instrucciones para Agentes

## Stack Tecnológico

| Componente | Tecnología / Versión | Propósito |
|---|---|---|
| Motor BD | PostgreSQL 16+ | Base de datos relacional |
| Lenguaje | SQL / PL/pgSQL | Definición de esquema, vistas, funciones y procedimientos |

## Estructura de carpetas

Los archivos `.sql` heredados de entregas anteriores (`schema.sql`, `objects.sql`,
`data.sql`, `queries.sql`, `carga_masiva.sql`, `indices_semana3.sql`) están
dentro de la carpeta `Archivos necesarios para la BD/`, siguiendo la misma
convención de entregas previas.

Los archivos **nuevos** de esta entrega (`indices.sql`, `views.sql`,
`informe_mediciones.md`) van sueltos en la **raíz** del repositorio, junto a
`AGENTS.md`, `protocolo_seguridad.md` y `README.md` — así coinciden
exactamente con la estructura de entrega que pide el enunciado de la cátedra.

Las carpetas `specs/` y `duia/` también están en la raíz, con un archivo por
integrante dentro de cada una.

## Orden de ejecución

```
Archivos necesarios para la BD/schema.sql
→ Archivos necesarios para la BD/objects.sql
→ Archivos necesarios para la BD/data.sql
→ Archivos necesarios para la BD/carga_masiva.sql
→ Archivos necesarios para la BD/indices_semana3.sql
→ indices.sql (Parte A de este TP, en la raíz)
→ views.sql (Partes B y C de este TP, en la raíz)
```

Cada archivo depende del anterior. `carga_masiva.sql` e `indices_semana3.sql`
solo se aplican sobre `copia_trabajo`, nunca sobre `plantilla_food_store`.

## Base de Conocimiento y Steering

- `schema.sql`: Tipos ENUM, tablas, constraints e índices originales.
- `objects.sql`: Vistas base, función de cálculo, triggers y `sp_crear_pedido`.
- `data.sql`: Datos de prueba chicos.
- `queries.sql`: Historias de usuario resueltas + consultas analíticas (banco
  de consultas candidatas para la Parte A de este TP).
- `carga_masiva.sql`: Población masiva (heredado, sin cambios).
- `indices_semana3.sql`: 2 índices adicionales ya aceptados en una entrega
  previa (ver sección de índices existentes, abajo).
- `indices.sql` **(nuevo — Parte A)**: los `CREATE INDEX` finalmente aceptados
  en esta entrega, comentados con su justificación.
- `views.sql` **(nuevo — Partes B y C)**: las 4 vistas simples + la vista
  materializada de esta entrega.
- `specs/` **(nuevo)**: especificaciones de Kiro, una por cada índice, vista o
  vista materializada, redactadas ANTES de generar el SQL con OpenCode.
- `informe_mediciones.md` **(nuevo)**: EXPLAIN ANALYZE antes/después de cada
  consulta indexada, tiempo de escritura antes/después, mediciones de la
  vista materializada, justificación de índices descartados.
- `duia.md` **(nuevo)**: bitácora de uso de IA de esta entrega.
- `.kiro/steering/project-overview.md`: visión general y orden de ejecución.
- `.kiro/steering/conventions.md`: convenciones + índices y vistas existentes.
- `.kiro/steering/objects-and-patterns.md`: triggers, `sp_crear_pedido`, patrones.

## Índices ya existentes (NO recrear, ni duplicar en la Parte A)

### Del schema original
- `idx_producto_categoria_id` — `producto(categoria_id)`
- `idx_pedido_usuario_id` — `pedido(usuario_id)`
- `idx_producto_nombre_vigente` — `producto(nombre) WHERE eliminado = FALSE` (parcial)

### De una entrega previa (`indices_semana3.sql`)
- `idx_pedido_fecha_estado_vigente` — `pedido(fecha, estado) WHERE eliminado = FALSE` (parcial)
- `idx_pedido_usuario_total_estado` — `pedido(usuario_id, total) WHERE eliminado = FALSE AND estado IN ('CONFIRMADO', 'TERMINADO')` (parcial)

**Antes de proponer un índice nuevo en la Parte A, verificar contra estos 5.**
Un índice que duplica o es redundante con alguno de estos es candidato directo
al descarte por sobreindexación que pide la consigna.

## Vistas ya existentes (reutilizables para la Parte B)

- `v_categorias_vigentes` — categorías con `eliminado = FALSE`
- `v_productos_vigentes` — productos + categoría, ambos vigentes
- `v_pedidos_resumen` — pedidos vigentes + nombre completo del usuario
- `v_pedido_detalle` — líneas de detalle vigentes + nombre del producto

**Estas 4 ya cubren 3 de las 4 vistas que pide la Parte B** de este TP
("productos vigentes con categoría", "pedidos con datos del usuario", "detalle
de pedido con nombre del producto"). Para la Parte B: verificar que cumplen la
spec exacta de este TP y hacer la verificación de equivalencia contra una
consulta manual (esto no se había hecho antes). La única vista genuinamente
nueva a crear es la de seguridad: `usuario` sin la columna `contrasena`.

## Protocolo de seguridad ante la BD (obligatorio)

1. Trabajar sobre una copia descartable: `createdb -T plantilla_food_store copia_trabajo`.
2. Todo script de escritura corre primero dentro de `BEGIN...ROLLBACK` y se
   inspecciona antes de aceptarlo.
3. Cambios estructurales (`CREATE INDEX`, `CREATE VIEW`, `CREATE MATERIALIZED
   VIEW`) requieren `pg_dump` de respaldo previo a `respaldos/`.
4. Después de crear un índice, correr `ANALYZE` sobre la tabla afectada antes
   de medir con EXPLAIN ANALYZE.
5. Recién al final: `COMMIT` y luego commit en Git.

Detalles y comandos exactos en `protocolo_seguridad.md`.

## Flujo obligatorio de esta entrega (Kiro → OpenCode → Git)

Para cada índice, vista o vista materializada, en este orden:

1. **Especificar en Kiro primero**: qué se necesita, sobre qué consulta o
   reporte, con qué criterio de aceptación. El spec se guarda en `specs/`.
2. **Generar con OpenCode** a partir de esa spec — nunca pedir "creame un
   índice" sin más contexto.
3. **Leer línea por línea** antes de ejecutar. Probar sobre `copia_trabajo` o
   dentro de una transacción reversible, con respaldo previo si corresponde.
4. **Commit en Git separado y descriptivo** por cada pieza (ej: `git commit -m
   "Índice compuesto pedido(fecha, estado) — reduce Seq Scan en reporte
   mensual"`). El historial de commits es parte de la entrega.

## Reglas Duras del Proyecto

1. **Nombres de tablas:** singular y español (`categoria`, `producto`, `usuario`, `pedido`, `detalle_pedido`).
2. **Borrado lógico:** nunca `DELETE`. Usar `UPDATE <tabla> SET eliminado = TRUE WHERE id = :id AND eliminado = FALSE`.
3. **Altas de pedidos:** siempre `CALL sp_crear_pedido(...)`.
4. **Triggers automáticos:** no modificar `trg_subtotal`, `trg_total_ins`, `trg_total_upd`.
5. **Vistas vigentes:** usarlas para filtrar registros activos en vez de escribir el filtro a mano.
6. **JOINs y borrado lógico:** el filtro `eliminado = FALSE` debe aplicarse en cada tabla involucrada.
7. **Índices propuestos:** ninguno se aplica sin poder explicar en la defensa
   oral qué nodo del plan ataca y por qué se espera que mejore — ni sin
   verificar primero que no duplique uno de los 5 ya existentes.
8. **Vistas de seguridad:** la vista que expone `usuario` nunca debe incluir
   la columna `contrasena` (ni ninguna otra columna sensible que se agregue
   a futuro).
