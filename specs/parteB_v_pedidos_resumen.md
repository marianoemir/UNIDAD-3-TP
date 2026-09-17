# Spec — Vista: v_pedidos_resumen

## Estado
Reutilizada de `objects.sql` (Verificada para Parte B).

## Objetivo
Consolidar la información general de los pedidos de la tienda, incluyendo datos clave del usuario comprador y el total acumulado.

## Requerimientos
- **Tablas base:** `pedido`, `usuario`
- **Joins:** `pedido.id_usuario = usuario.id_usuario`
- **Columnas expuestas:**
  - `id_pedido`
  - `fecha`
  - `id_usuario`
  - `nombre_usuario` (o combinación de nombre/apellido)
  - `estado`
  - `monto_total` (o total derivado)

## Criterio de Aceptación
La consulta a la vista debe arrojar el mismo número de filas y registros exactos que la consulta manual directa uniendo las tablas `pedido` y `usuario`.