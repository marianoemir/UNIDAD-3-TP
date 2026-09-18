# Spec — Vista: v_pedidos_resumen

## Estado
Reutilizada de `objects.sql` (Verificada para Parte B).

## Objetivo
Consolidar la información general de los pedidos de la tienda, incluyendo el nombre completo del usuario comprador.

## Requerimientos
- **Tablas base:** `pedido`, `usuario`
- **Join:** `pedido.usuario_id = usuario.id`
- **Filtro de vigencia:** `pedido.eliminado = false`
- **Columnas expuestas:**
  - `id` (de pedido)
  - `usuario` (nombre y apellido concatenados)
  - `fecha`
  - `estado`
  - `forma_pago`
  - `total`

## Criterio de Aceptación
La ejecución de `SELECT * FROM v_pedidos_resumen` debe retornar la misma cantidad de filas y los mismos valores que:
```sql
SELECT ped.id,
       u.nombre || ' ' || u.apellido AS usuario,
       ped.fecha, ped.estado, ped.forma_pago, ped.total
FROM pedido ped
JOIN usuario u ON u.id = ped.usuario_id
WHERE ped.eliminado = FALSE;
```