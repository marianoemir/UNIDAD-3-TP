# spec: idx_producto_nombre_lower_vigente

**Objetivo:** acelerar la búsqueda de productos por nombre, sin distinguir
mayúsculas/minúsculas, para autocompletado de catálogo.

**Consulta afectada:**
```sql
SELECT id, nombre, precio
FROM producto
WHERE lower(nombre) LIKE 'pizza%'
  AND eliminado = FALSE;
```

**Frecuencia:** camino crítico (búsqueda de catálogo / autocompletado, uso
frecuente por parte de los usuarios finales).

**Columnas candidatas:**
- `lower(nombre)` — se necesita un índice de expresión, ya que la consulta
  no filtra sobre la columna cruda `nombre`, sino sobre `lower(nombre)`. El
  índice parcial existente `idx_producto_nombre_vigente` (sobre `nombre` sin
  `lower()`) no cubre este caso.

**Selectividad estimada:** muy alta — se espera que muy pocos productos
coincidan con un prefijo de nombre específico.

**Criterio de aceptación:** el plan pasa de Seq Scan a Index/Bitmap Scan y el
tiempo baja al menos un orden de magnitud.

**Resultado (ver `informe_mediciones.md`):** ACEPTADO, con corrección.
Selectividad medida: ~0.002% (1 de 50.016 filas). El primer intento
(`lower(nombre)` con operator class por defecto) fue ignorado por el
optimizador — un B-Tree estándar no soporta `LIKE` para búsquedas de prefijo
salvo en locale `C`. Se corrigió con `text_pattern_ops`:
```sql
CREATE INDEX idx_producto_nombre_lower_vigente
ON producto (lower(nombre) text_pattern_ops)
WHERE eliminado = FALSE;
```
Con la corrección, el plan pasó a `Bitmap Heap Scan` usando el índice,
bajando de 50.191 ms a 0.222 ms (~226x más rápido).
