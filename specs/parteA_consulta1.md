# spec: idx_detalle_pedido_producto_id

**Objetivo:** acelerar el ranking de los 5 productos más vendidos, agregando
por cantidad total pedida.

**Consulta afectada:**
```sql
SELECT pr.id, pr.nombre, SUM(dp.cantidad) AS unidades
FROM detalle_pedido dp
JOIN producto pr ON pr.id = dp.producto_id
WHERE dp.eliminado = FALSE
GROUP BY pr.id, pr.nombre
ORDER BY unidades DESC
LIMIT 5;
```

**Frecuencia:** reporte esporádico (ranking gerencial, no camino crítico de
la aplicación).

**Columnas candidatas:**
- `detalle_pedido.producto_id` (usada en el JOIN y en el GROUP BY vía `pr.id`)

**Selectividad estimada:** muy baja / prácticamente nula — la consulta no
tiene ningún filtro sobre `producto_id`, participan el 100% de las filas de
`detalle_pedido` en la agregación.

**Criterio de aceptación:** el plan pasa de Seq Scan a Index/Bitmap Scan y el
tiempo baja al menos un orden de magnitud.

**Resultado (ver `informe_mediciones.md`):** DESCARTADO. El optimizador no
adoptó el índice — el plan quedó idéntico. Sin ningún filtro selectivo, el
índice no tiene forma de evitar leer filas irrelevantes, porque no las hay.
