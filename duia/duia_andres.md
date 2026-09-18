# Declaración de Uso de IA (DUIA) — Andrés

## Parte C: Vista Materializada

* **Herramientas utilizadas:** Kiro / OpenCode / Nemotron / Gemini.
* **Propósito:** Especificación de requerimientos, generación de la vista materializada con índice único y análisis de tiempos de ejecución.

### Interacción 1: Especificación en Kiro
* **Spec utilizado:** `specs/parteC_facturacion_categoria_mes.md`.
* **Propósito:** Definir el reporte de facturación por categoría y mes, la consulta base de agregación y los criterios de aceptación para la vista materializada y su índice único.

### Interacción 2: Generación del SQL con LLM / OpenCode
* **Prompt entregado:** "Generá el script SQL para la Parte C basándote en la especificación en `specs/parteC_facturacion_categoria_mes.md`. Debe incluir la creación de la vista materializada `mv_facturacion_categoria_mes` con la cláusula WITH DATA y un índice único llamado `idx_mv_facturacion_cat_mes_pk` sobre las columnas `(categoria_id, mes)`."
* **Propuesta recibida:** Se obtuvo la propuesta del `CREATE MATERIALIZED VIEW` agregando `WITH DATA` al inicio.
* **Evaluación y ajuste:** Se detectó un error de sintaxis en PostgreSQL (la cláusula `WITH DATA` debe ubicarse al final de la definición de la vista materializada, posterior al `SELECT`). Se corrigió el orden de las cláusulas antes de ejecutarlo en la base de datos.
* **Resultado aceptado:** Creación exitosa de `mv_facturacion_categoria_mes` e índice único `idx_mv_facturacion_cat_mes_pk`.