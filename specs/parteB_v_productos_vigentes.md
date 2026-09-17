# Spec — Vista: v_productos_vigentes

## Estado
Reutilizada de `objects.sql` (Verificada para Parte B).

## Objetivo
Proporcionar un acceso simplificado a la lista de productos activos/vigentes en la tienda, omitiendo aquellos que han sido dados de baja lógicamente.

## Requerimientos
- **Tabla base:** `producto`
- **Filtro de vigencia:** `vigente = true`
- **Columnas expuestas:**
  - `id_producto`
  - `nombre`
  - `descripcion`
  - `precio`
  - `id_categoria`
  - `vigente`

## Criterio de Aceptación
La ejecución de `SELECT * FROM v_productos_vigentes` debe retornar exactamente el mismo conjunto de filas que:
`SELECT * FROM producto WHERE vigente = true;`