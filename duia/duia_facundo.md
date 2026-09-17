# Bitácora DUIA — Facundo (Parte B)

## Tareas realizadas
- Verificación del estado inicial de la base de datos `copia_trabajo` y validación del índice de la Parte A (`idx_producto_nombre_lower_vigente`).
- Redacción de especificaciones Kiro en la carpeta `specs/` para las 4 vistas del sistema:
  - `parteB_v_productos_vigentes.md`
  - `parteB_v_pedidos_resumen.md`
  - `parteB_v_pedido_detalle.md`
  - `parteB_v_usuario_seguro.md`
- Creación e implementación de la nueva vista de seguridad `v_usuario_seguro` en `views.sql`, filtrando la columna `contrasena`.
- Ejecución de pruebas de equivalencia entre las 4 vistas y sus consultas manuales directas, registrando la evidencia en `informe_mediciones.md`.

## Herramientas y Prompting
- **Herramienta:** Asistente IA (Kiro / Gemini).
- **Uso:** Generación de especificaciones de vistas, estructuración de script `views.sql` y formato del reporte de equivalencias.
- **Criterio de Aceptación:** Se aceptaron las estructuras ajustadas a la definición real de las tablas obtenida mediante `\d usuario`, asegurando la compatibilidad con el esquema existente.