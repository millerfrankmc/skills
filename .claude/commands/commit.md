# Análisis y Commit Automático

Necesito que generes un resumen completo de los cambios realizados en el proyecto y crees un commit automático con un mensaje descriptivo

## Tareas a realizar:

1. **Analizar el estado actual de git**:
   - Obtener el listado de archivos modificados con `git status`
   - Revisar los cambios específicos con `git diff`
   - Analizar el historial de commits recientes con `git log --oneline -5`

2. **Generar resumen de cambios**:
   - Identificar las áreas del proyecto afectadas (frontend, backend, configuración, etc.)
   - Agrupar cambios por funcionalidad o componente
   - Extraer los cambios más importantes y su impacto

3. **Crear un changelog comprimido**:
   - Resumir los cambios en formato conciso pero descriptivo
   - Incluir contexto técnico relevante
   - Mencionar archivos clave modificados

4. **Generar mensaje de commit**:
   - Crear un título claro y descriptivo (máximo 50 caracteres)
   - Incluir cuerpo descriptivo con los detalles principales
   - Usar formato convencional de commits cuando aplique
   - Agregar pie de página con información adicional si es necesario

5. **Ejecutar el commit**:
   - Agregar los archivos modificados al staging area
   - Crear el commit con el mensaje generado
   - Verificar que el commit se haya creado exitosamente

## Formato del mensaje de commit:

```
tipo(ámbito): descripción corta

- Cambio principal 1: breve descripción
- Cambio principal 2: breve descripción
- Cambio principal 3: breve descripción

Detalles adicionales sobre el impacto o contexto técnico.

```

## Tipos de commit comunes (con ejemplos):
- `feat`: nueva funcionalidad (ej: "feat(auth): agregar sistema de login JWT")
- `fix`: corrección de errores (ej: "fix(reports): resolver error de carga en reporte R007")
- `refactor`: refactorización sin cambios funcionales (ej: "refactor(components): optimizar renderizado de tabla")
- `style`: cambios de formato/estilo (ej: "style(css): ajustar márgenes en responsive")
- `docs`: documentación (ej: "docs(api): actualizar documentación de endpoints")
- `test`: pruebas (ej: "test(unit): agregar pruebas para servicio de usuarios")
- `chore`: tareas de mantenimiento (ej: "chore(deps): actualizar dependencias del proyecto")

## Ámbitos comunes:
- `frontend`: cambios en el frontend
- `backend`: cambios en el backend
- `api`: cambios en la API
- `ui`: cambios en componentes de interfaz
- `config`: cambios en configuración
- `docs`: cambios en documentación

Por favor, ejecuta este análisis completo y realiza el commit automático con un mensaje descriptivo basado en los cambios actuales.