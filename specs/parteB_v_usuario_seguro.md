# Spec — Vista: v_usuario_seguro

## Estado
Nueva vista a crear para la Parte B.

## Objetivo
Exponer los datos del perfil de los usuarios para reportes generales o paneles administrativos sin comprometer la seguridad del sistema.

## Requerimientos
- **Tabla base:** `usuario`
- **Restricción de seguridad:** Se oculta explícitamente la columna `contrasena`.
- **Columnas expuestas:**
  - `id`
  - `nombre`
  - `apellido`
  - `mail`
  - `celular`
  - `rol`
  - `eliminado`
  - `created_at`

## Criterio de Aceptación
1. La vista no debe exponer en ningún caso la columna `contrasena`.
2. El total de filas de `v_usuario_seguro` debe coincidir exactamente con `SELECT COUNT(*) FROM usuario`.