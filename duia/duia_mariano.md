# Declaración de Uso de IA (DUIA) — Parte A

**Integrante:** Mariano Chirino
**Rol / Asignación:** Parte A — Plan de indexado asistido por IA
**Materia:** Base de Datos II (UTN) — Unidad 3, Semana 1
**Proyecto Integrador:** Food Store

---

## Registro de Interacciones y Decisiones con IA

| Herramienta | Para qué se usó | Spec / Prompt (resumen) | Se aceptó / se descartó — por qué |
|---|---|---|---|
| Kiro | Revisar spec y validar la conclusión de descarte para Consulta 1 (Top 5 productos más vendidos) | Se le pasó el spec completo de `specs/parteA_consulta1.md` (objetivo, consulta, columnas candidatas, selectividad estimada, criterio de aceptación). | **Confirmó el descarte**, agregando un argumento propio: *"leer el índice + ir a la heap termina siendo más costoso que un Seq Scan directo con HashAggregate"* — sin ningún predicado selectivo, el índice no puede evitar el scan completo, y el optimizador hace bien en ignorarlo. Coincide con la medición real ya hecha (plan idéntico, sin adopción del índice). |
| Kiro | Revisar spec y validar consistencia del descarte para Consulta 2 (Pedidos con total > promedio) | Se le pasó el spec completo de `specs/parteA_consulta2.md`. | **Confirmó consistencia entre los 3 archivos del repo**: el spec documenta la decisión con la selectividad medida (50.05%) y los tiempos exactos (373 ms → 300 ms), `indices.sql` correctamente no incluye este índice (fue descartado), y `informe_mediciones.md` tiene el detalle completo. Validación cruzada sin inconsistencias. |
| OpenCode | Confirmar el índice propuesto para Consulta 2 (Pedidos con total > promedio) | Se le pasó el spec de `specs/parteA_consulta2.md`. | Propuso `CREATE INDEX idx_pedido_total ON pedido (total) WHERE eliminado = FALSE;`, aclarando que un B-Tree estándar sobre `NUMERIC` soporta perfectamente operadores de comparación en cualquier locale — a diferencia de la Consulta 3, acá no hay problema de operator class. Anticipó correctamente que el descarte, de darse, sería por selectividad (~50%) y no por un problema técnico del índice — coincide exactamente con lo medido en `informe_mediciones.md`. |
| OpenCode | Proponer índice para Consulta 3 (Búsqueda de productos por nombre) | Se le pasó el spec de `specs/parteA_consulta3.md`, con la pregunta puntual de si el índice soportaría `LIKE` en cualquier locale. | Propuso directamente `CREATE INDEX idx_producto_nombre_lower_vigente ON producto (lower(nombre) text_pattern_ops) WHERE eliminado = FALSE;`, explicando el mecanismo exacto: en locales no-C (como `es_AR.UTF-8`, habitual en Argentina) un B-Tree con operator class por defecto no puede resolver `LIKE 'pizza%'` porque el operador `~~` no está vinculado a ese operator class; `text_pattern_ops` habilita comparación byte a byte, permitiendo que el planificador reescriba el `LIKE` como el rango `lower(nombre) ~>=~ 'pizza' AND lower(nombre) ~<~ 'pizzb'`, resoluble con Index Scan. **Se aceptó tal cual**, coincide con el índice ya creado y medido. |

---

## Nota sobre el mecanismo técnico de la Consulta 3

A diferencia de lo que ocurrió en la primera exploración manual (donde se
probó un índice sin `text_pattern_ops` y falló), al consultarle a OpenCode
directamente con la spec completa, propuso la versión correcta desde el
primer intento, incluyendo la explicación del mecanismo interno (locale del
servidor, reescritura de `LIKE` como rango con `~>=~`/`~<~`). Esto muestra el
valor de una spec bien detallada: entregarle a la IA el contexto completo
(consulta exacta, criterio de aceptación) permitió que anticipara un problema
que en la exploración manual recién se descubrió después de medir.

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
