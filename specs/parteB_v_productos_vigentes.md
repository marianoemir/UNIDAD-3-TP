# Spec — Vista: v_productos_vigentes

## Estado
Reutilizada de `objects.sql` (Verificada para Parte B).

## Objetivo
Proporcionar un acceso simplificado a la lista de productos activos/vigentes en la tienda, con el nombre de su categoría, omitiendo aquellos que han sido dados de baja lógicamente (producto o categoría).

## Requerimientos
- **Tablas base:** `producto`, `categoria`
- **Join:** `producto.categoria_id = categoria.id`
- **Filtro de vigencia:** `producto.eliminado = false` y `categoria.eliminado = false`
- **Columnas expuestas:**
  - `id` (de producto)
  - `nombre` (de producto)
  - `precio`
  - `stock`
  - `categoria` (nombre de la categoría)

## Criterio de Aceptación
La ejecución de `SELECT * FROM v_productos_vigentes` debe retornar la misma cantidad de filas y los mismos valores que:
```sql
SELECT p.id, p.nombre, p.precio, p.stock, c.nombre AS categoria
FROM producto p
JOIN categoria c ON c.id = p.categoria_id
WHERE p.eliminado = FALSE AND c.eliminado = FALSE;
```