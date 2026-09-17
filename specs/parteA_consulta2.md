# spec: idx_pedido_total

**Objetivo:** acelerar la consulta de pedidos cuyo total supera el promedio
general de todos los pedidos vigentes.

**Consulta afectada:**
```sql
SELECT id, total
FROM pedido
WHERE eliminado = FALSE
  AND total > (SELECT AVG(total) FROM pedido WHERE eliminado = FALSE)
ORDER BY total DESC;
```

**Frecuencia:** reporte esporádico (análisis financiero puntual, no camino
crítico).

**Columnas candidatas:**
- `pedido.total` (usada en el filtro `WHERE total > ...` y en el `ORDER BY`)

**Selectividad estimada:** a confirmar con medición exacta, pero se anticipa
alta (cercana al 50%), dado que "mayor al promedio" típicamente devuelve
aproximadamente la mitad de una distribución.

**Criterio de aceptación:** el plan pasa de Seq Scan a Index/Bitmap Scan y el
tiempo baja al menos un orden de magnitud.

**Resultado (ver `informe_mediciones.md`):** DESCARTADO. Selectividad medida:
50.05% de la tabla, por encima del umbral de baja selectividad (20-30%) de
la guía académica. El optimizador sí adoptó el índice (Bitmap Heap Scan +
Index Only Scan para el promedio), pero la mejora fue de solo ~19%
(373.099 ms → 300.029 ms), muy por debajo del orden de magnitud exigido. Se
descarta por sobreindexación: el beneficio de lectura no compensa el costo
de mantenimiento sobre una tabla de escritura frecuente.
