# Spec — Vista: v_pedido_detalle

## Estado
Reutilizada de `objects.sql` (Verificada para Parte B).

## Objetivo
Desglosar las líneas de detalle asociadas a cada pedido, mostrando el nombre del producto comprado, la cantidad y los montos.

## Requerimientos
- **Tablas base:** `detalle_pedido`, `producto`
- **Join:** `detalle_pedido.producto_id = producto.id`
- **Filtro de vigencia:** `detalle_pedido.eliminado = false`
- **Columnas expuestas:**
  - `pedido_id`
  - `producto` (nombre del producto)
  - `cantidad`
  - `precio_unitario`
  - `subtotal`

## Criterio de Aceptación
La ejecución de `SELECT * FROM v_pedido_detalle` debe retornar la misma cantidad de filas y los mismos valores que:
```sql
SELECT dp.pedido_id,
       pr.nombre AS producto,
       dp.cantidad, dp.precio_unitario, dp.subtotal
FROM detalle_pedido dp
JOIN producto pr ON pr.id = dp.producto_id
WHERE dp.eliminado = FALSE;
```