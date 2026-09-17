# Declaración de Uso de IA (DUIA) — Parte A

**Integrante:** Mariano Chirino
**Rol / Asignación:** Parte A — Plan de indexado asistido por IA
**Materia:** Base de Datos II (UTN) — Unidad 3, Semana 1
**Proyecto Integrador:** Food Store

---

## Registro de Interacciones y Decisiones con IA

| Herramienta | Para qué se usó | Spec / Prompt (resumen) | Se aceptó / se descartó — por qué |
|---|---|---|---|
| OpenCode | Proponer índice para Consulta 1 (Top 5 productos más vendidos) | Spec en `specs/parteA_consulta1.md`: consulta exacta, columnas candidatas (`producto_id` en `detalle_pedido`), criterio de aceptación (Seq Scan → Index Scan, mejora de un orden de magnitud). | **Se descartó.** Se creó `idx_detalle_pedido_producto_id`, pero el plan quedó idéntico (mismo `Seq Scan` en ambas tablas, mismo cost). La consulta no tiene ningún filtro selectivo sobre `producto_id` — agrupa el 100% de la tabla — por lo que no hay filas que el índice pueda evitar leer. Se eliminó tras la medición. |
| OpenCode | Proponer índice para Consulta 2 (Pedidos con total > promedio) | Spec en `specs/parteA_consulta2.md`: consulta exacta, columna candidata (`total` en `pedido`), selectividad estimada. | **Se descartó.** Se creó `idx_pedido_total` y, a diferencia de la Consulta 1, el optimizador sí lo adoptó (`Bitmap Heap Scan` + `Index Only Scan` para el promedio). Sin embargo la mejora fue de solo ~19% (373.099 ms → 300.029 ms), muy por debajo del orden de magnitud exigido. Selectividad medida: 50.05% de la tabla — por encima del umbral de baja selectividad (20-30%) de la guía académica. Se eliminó tras la medición. |
| OpenCode | Proponer índice para Consulta 3 (Búsqueda de productos por nombre, case-insensitive) | Spec en `specs/parteA_consulta3.md`: consulta con `lower(nombre) LIKE 'pizza%'`, columna candidata como índice de expresión. | **Se aceptó, con corrección.** El primer índice propuesto (`lower(nombre)` con operator class por defecto) fue ignorado por el optimizador — un B-Tree estándar no soporta `LIKE` para búsquedas de prefijo salvo en locale `C`. Se corrigió agregando `text_pattern_ops` a la definición del índice. Con la corrección, el plan pasó de `Seq Scan` (50.191 ms) a `Bitmap Heap Scan` usando el índice (0.222 ms) — mejora de ~226x. |

---

## Nota sobre el primer intento fallido de la Consulta 3

Este caso se documenta explícitamente porque ilustra el criterio de
aceptación de la cátedra: la propuesta inicial de la IA (un índice de
expresión simple sobre `lower(nombre)`) era razonable en principio, pero al
medir en el motor real se comprobó que el optimizador no lo usaba. En vez de
descartar la idea completa, se investigó el motivo técnico (falta de
`text_pattern_ops` para soportar `LIKE` con prefijo) y se corrigió la
propuesta antes de volver a medir. Esto es un ejemplo de "leer línea por
línea y verificar", no de aplicar la primera sugerencia de la IA sin
cuestionarla.

## Medición de costo de escritura

Se midió el tiempo de 500 `INSERT` de prueba en `detalle_pedido` (envueltos
en `BEGIN...ROLLBACK`), comparando el estado con y sin el índice descartado
de la Consulta 1 recreado temporalmente. El resultado (272 ms sin el índice
extra vs. 240 ms con él) no mostró una penalización clara — con este volumen
de prueba, la variación normal del sistema pesa más que el costo real de
mantenimiento de un índice adicional. Se documenta este resultado tal cual
salió, sin forzar una conclusión que los datos no sustentan.

## Resumen de decisiones

| Consulta | Selectividad | Decisión | Mejora |
|---|---|---|---|
| 1. Top 5 productos vendidos | ~100% | Descartada | Ninguna |
| 2. Pedidos > promedio | 50.05% | Descartada | ~19% (insuficiente) |
| 3. Búsqueda por nombre | ~0.002% | **Aceptada** | ~226x |

Detalle completo de cada consulta (planes reales antes/después) en
`informe_mediciones.md`.
