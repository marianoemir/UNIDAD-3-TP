# Bitácora DUIA — Facundo (Parte B)

**Materia:** Base de Datos II (UTN) — Unidad 3, Semana 1
**Proyecto Integrador:** Food Store

---

## Registro de Interacciones y Decisiones con IA

### Interacción 1: Especificación en Kiro de las 3 vistas reutilizadas

* **Herramienta:** Kiro
* **Propósito:** Redactar las specs de las vistas de reporte que ya existían
  en `objects.sql` (`v_productos_vigentes`, `v_pedidos_resumen`,
  `v_pedido_detalle`), para poder verificarlas formalmente en este TP.
* **Prompt entregado (resumen):** "Documentá en formato spec las 3 vistas de
  reporte ya creadas en `objects.sql`: objetivo, tablas base, columnas
  expuestas y criterio de aceptación (equivalencia contra consulta manual)."
* **Propuesta recibida:** Kiro generó una primera versión de las specs con
  nombres de columna genéricos (`id_producto`, `id_pedido`, `vigente`, etc.)
  en lugar de los nombres reales del esquema (`id`, `eliminado`, etc.).
* **Evaluación y ajuste:** Al releer `objects.sql` línea por línea contra
  las specs generadas, se detectó que los nombres de columna no coincidían
  con la definición real de las vistas. Se corrigieron las 3 specs
  (`specs/parteB_v_productos_vigentes.md`, `specs/parteB_v_pedidos_resumen.md`,
  `specs/parteB_v_pedido_detalle.md`) para que reflejen exactamente las
  columnas, joins y filtros que usa cada `CREATE VIEW` real.
* **Resultado aceptado:** Specs corregidas y verificadas contra el SQL real
  de `objects.sql`, no solo contra la descripción textual de la consigna.

### Interacción 2: Especificación y generación de `v_usuario_seguro`

* **Herramienta:** Kiro (spec) / OpenCode (generación del SQL)
* **Spec utilizado:** `specs/parteB_v_usuario_seguro.md`
* **Propósito:** Definir una vista que exponga los datos de perfil de
  `usuario` sin la columna `contrasena`, para poder otorgar `SELECT` sobre
  ella sin dar acceso a la tabla base completa.
* **Prompt entregado (resumen):** "A partir de la spec de
  `v_usuario_seguro`, generá el `CREATE VIEW` que expone todas las columnas
  de `usuario` excepto `contrasena`, agregando un comentario que explique el
  motivo de seguridad."
* **Propuesta recibida:** OpenCode generó el `CREATE OR REPLACE VIEW`
  listando explícitamente las columnas permitidas (en vez de `SELECT *`
  seguido de una exclusión), con un `COMMENT ON VIEW` describiendo el
  propósito de seguridad.
* **Evaluación y ajuste:** Se verificó columna por columna contra
  `\d usuario` que la vista no incluyera `contrasena` y que sí incluyera el
  resto de las columnas relevantes para reportes (`nombre`, `apellido`,
  `mail`, `celular`, `rol`, `eliminado`, `created_at`). No hizo falta
  modificar la propuesta.
* **Resultado aceptado:** `v_usuario_seguro` tal como quedó en `views.sql`.

### Interacción 3: Verificación de equivalencia de las 4 vistas

* **Herramienta:** Asistente IA (Kiro / Gemini) para estructurar el formato
  del reporte de equivalencias.
* **Propósito:** Comparar, para cada vista, el resultado contra su consulta
  manual equivalente y documentar la evidencia en `informe_mediciones.md`.
* **Uso:** Se le pidió a la IA que propusiera el formato de tabla para
  registrar la verificación (vista, consulta manual, conteo de filas,
  coincidencia). Las consultas de verificación en sí y los conteos de filas
  se ejecutaron a mano sobre `copia_trabajo`, no los generó la IA.
* **Criterio de aceptación:** se aceptaron las estructuras de vista ajustadas
  a la definición real de las tablas, obtenida mediante `\d usuario` y la
  lectura directa de `objects.sql` — nunca se dio por válida una propuesta
  de la IA sin chequearla contra el esquema real.
* **Resultado:** las 4 vistas (`v_productos_vigentes`, `v_pedidos_resumen`,
  `v_pedido_detalle`, `v_usuario_seguro`) coinciden exactamente (o con la
  diferencia esperada por el filtro de vigencia) contra su consulta manual —
  ver tabla completa en `informe_mediciones.md`, sección Parte B.

---

## Resumen de decisiones

| Vista | Origen | Decisión sobre la spec de la IA |
|---|---|---|
| `v_productos_vigentes` | Reutilizada | Spec corregida: nombres de columna no coincidían con `objects.sql` |
| `v_pedidos_resumen` | Reutilizada | Spec corregida: nombres de columna no coincidían con `objects.sql` |
| `v_pedido_detalle` | Reutilizada | Spec corregida: nombres de columna no coincidían con `objects.sql` |
| `v_usuario_seguro` | Nueva | Propuesta de OpenCode aceptada sin cambios, verificada contra `\d usuario` |