# Spec — Vista: v_pedido_detalle

## Estado
Reutilizada de `objects.sql` (Verificada para Parte B).

## Objetivo
Desglosar las líneas de detalle asociadas a cada pedido, vinculando la información de los productos comprados con sus cantidades y precios unitarios.

## Requerimientos
- **Tablas base:** `detalle_pedido` (o `linea_pedido`), `producto`
- **Joins:** Relación por `id_producto`
- **Columnas expuestas:**
  - `id_pedido`
  - `id_producto`
  - `nombre_producto`
  - `cantidad`
  - `precio_unitario`
  - `subtotal` (si aplica)

## Criterio de Aceptación
La cantidad total de registros retornados por `SELECT COUNT(*) FROM v_pedido_detalle` debe ser idéntica al conteo directo sobre la tabla de detalle de pedidos.