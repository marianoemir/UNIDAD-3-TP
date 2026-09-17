# Spec: mv_facturacion_categoria_mes

**Objetivo:** Materializar el reporte analítico de facturación total agrupado por categoría de producto y por mes/año para evitar procesar los miles de registros de pedidos y detalles en cada consulta.

**Consulta analítica original:**
SELECT 
    c.id AS categoria_id,
    c.nombre AS categoria,
    DATE_TRUNC('month', p.fecha)::DATE AS mes,
    COUNT(DISTINCT p.id) AS total_pedidos,
    SUM(dp.cantidad * dp.precio_unitario) AS total_facturado
FROM categoria c
JOIN producto pr ON pr.categoria_id = c.id
JOIN detalle_pedido dp ON dp.producto_id = pr.id
JOIN pedido p ON p.id = dp.pedido_id
WHERE p.estado = 'CONFIRMADO' AND p.eliminado = FALSE
GROUP BY c.id, c.nombre, DATE_TRUNC('month', p.fecha)
ORDER BY mes DESC, total_facturado DESC;

**Criterios de aceptación:**
1. Crear una vista materializada llamada `mv_facturacion_categoria_mes` `WITH DATA`.
2. Crear un índice único sobre la combinación de `(categoria_id, mes)` para permitir actualizaciones sin bloqueo mediante `REFRESH MATERIALIZED VIEW CONCURRENTLY`.
3. La consulta directa sobre la vista debe reducir el tiempo de respuesta significativamente comparada con la ejecución sobre las tablas base.