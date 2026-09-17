# Protocolo de Seguridad — Food Store

Este documento define los pasos obligatorios que se aplican **siempre**, sin excepción,
antes de que cualquier script (propio o generado por IA) toque la base de datos del proyecto.

## Motor y entorno

- Motor: PostgreSQL
- Base de trabajo local: `food_store_dev`
- Base "plantilla" con el esquema y el seed chico ya aplicados: `plantilla_food_store`
- Base de trabajo descartable: `copia_trabajo`

## Paso 1 — Copia

Nunca se trabaja directamente sobre la base que contiene datos que importan.
Antes de cualquier cambio se crea una copia descartable:

```bash
createdb -T plantilla_food_store copia_trabajo
```

Si la plantilla `plantilla_food_store` todavía no existe, se crea una vez a partir del esquema:

```bash
createdb plantilla_food_store
psql -d plantilla_food_store -f "schema.sql"
psql -d plantilla_food_store -f "objects.sql"
psql -d plantilla_food_store -f "data.sql"
```

Todo el trabajo de esta entrega (carga masiva, índices nuevos, vistas, vista
materializada) se ejecuta sobre `copia_trabajo`, nunca sobre
`plantilla_food_store` ni sobre ninguna base con datos reales de producción.

## Paso 2 — Transacción

Todo script que escriba datos (INSERT, UPDATE, DELETE, o que agregue
restricciones) se ejecuta primero dentro de una transacción abierta:

```sql
BEGIN;

-- acá va el script generado por la IA o el cambio propio

-- se revisa: cuántas filas afectó, qué mensajes tiró, si el resultado
-- es el esperado

ROLLBACK; -- primero SIEMPRE se revierte para confirmar que se entendió el efecto
```

Recién cuando el efecto fue inspeccionado y es el esperado, se repite la
operación terminando en `COMMIT` en lugar de `ROLLBACK`.

**Nota para la Parte A de esta entrega:** al medir el costo de escritura
(varios cientos de `INSERT` en `detalle_pedido`, antes y después de crear los
índices nuevos), conviene envolver esa carga de prueba en su propia
transacción con `ROLLBACK` al final, para no dejar datos de prueba
permanentes en `copia_trabajo` que después ensucien otras mediciones.

## Paso 3 — Respaldo

Antes de cualquier cambio estructural se saca un respaldo de la copia de
trabajo, independiente del `ROLLBACK`:

```bash
pg_dump copia_trabajo > respaldos/copia_trabajo_YYYYMMDD_HHMM.sql
```

**Se considera cambio estructural, y requiere respaldo previo, cualquiera de:**
- `ALTER`, `DROP`
- `CREATE TRIGGER`, `CREATE FUNCTION`
- `CREATE INDEX` (Parte A de esta entrega)
- `CREATE VIEW` (Parte B de esta entrega)
- `CREATE MATERIALIZED VIEW` (Parte C de esta entrega)
- Cualquier migración

Los respaldos se guardan en la carpeta `respaldos/` del repo (o fuera del
repo si el archivo es pesado), con fecha y hora en el nombre.

**Recomendación para esta entrega:** sacar un respaldo separado justo antes de
empezar cada Parte (A, B, C), ya que cada una agrega objetos distintos
(índices, vistas, vista materializada) y conviene poder volver atrás a un
punto intermedio sin perder el trabajo de las otras partes.

## Paso 4 — Estadísticas actualizadas antes de medir

Después de crear un índice, correr:

```sql
ANALYZE <tabla>;
```

Sin este paso, el optimizador puede seguir usando estadísticas viejas y el
plan de `EXPLAIN ANALYZE` no refleja el índice recién creado — esto invalida
la comparación "antes/después" de la Parte A.

## Paso 5 — Verificación de equivalencia (vistas)

Para cada vista de la Parte B, antes de darla por válida:

1. Ejecutar la vista (`SELECT * FROM v_nombre;` o con las columnas que exponga).
2. Ejecutar la consulta manual equivalente, escrita por el propio estudiante.
3. Comparar que ambos resultados coincidan exactamente (conteo de filas y,
   si hace falta, `EXCEPT` en ambos sentidos).

Este paso es nuevo respecto a entregas anteriores — las vistas ya existentes
(`v_productos_vigentes`, `v_pedidos_resumen`, `v_pedido_detalle`) no habían
pasado por esta verificación formal todavía.

## Regla de fondo

Ningún script generado por OpenCode o Kiro se ejecuta directamente sobre la base. El flujo
siempre es:

1. Especificar primero en Kiro (spec guardada en `specs/`).
2. Generar con OpenCode a partir de esa spec.
3. Leer línea por línea antes de aplicar.
4. Probar dentro de `BEGIN...ROLLBACK` sobre `copia_trabajo`.
5. Si el cambio es estructural (índice, vista o vista materializada), sacar
   respaldo antes del `COMMIT` final.
6. Si el cambio afecta índices, correr `ANALYZE` antes de medir.
7. Si el cambio es una vista, verificar equivalencia (Paso 5) antes de darla
   por válida.
8. Recién ahí `COMMIT` y commit en Git — separado y descriptivo por cada
   pieza (índice, vista o vista materializada).

Ningún paso de esta lista se salta, incluso cuando el cambio parece trivial.
