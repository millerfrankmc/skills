---
name: linus-kiss-dry-yagni
description: Aplica KISS, DRY, YAGNI y principios de Linus Torvalds automáticamente. Sin confirmaciones. Devuelve código limpio + resumen de correcciones.
---

# KISS-DRY-YAGNI + Linus Torvalds

Skill directiva de simplicidad. Aplica correcciones automáticamente.

## Quick Reference

| Situación | Recurso |
|-----------|---------|
| Aplicar correcciones | Usa tablas abajo |
| Diseñar desde cero | `resources/design/kiss-driven-design.md` |
| Ejemplos Python | `resources/examples/python.md` |
| Ejemplos TypeScript | `resources/examples/typescript.md` |
| Ejemplos Go | `resources/examples/go.md` |
| Ejemplos Kotlin | `resources/examples/kotlin.md` |
| Decisión arquitectónica | `resources/decision/framework.md` |
| Medir complejidad | `resources/decision/metrics.md` |
| Principios Linus | `resources/principles/linus-torvalds.md` |
| Case studies | `resources/cases/case-studies.md` |
| Estrategias simplificación | `resources/strategies/simplification-strategies.md` |
| Fundamentos | `resources/principles/fundamentals.md` |

## Reglas de Aplicación Automática

### KISS - Simplificar

| Si detectas | Aplicar |
|------------|---------|
| Función hace 2+ cosas | Dividir en funciones separadas |
| 4+ niveles de anidamiento | Extraer función o usar guard clauses |
| 5+ parámetros | Usar objeto/config struct |
| Nombre necesita comentario | Renombrar |
| Existe solución más simple | Usarla |

### DRY - Centralizar

| Si detectas | Aplicar |
|------------|---------|
| Código idéntico 2+ veces | Extraer función |
| Constantes repetidas | Centralizar |
| Validaciones similares | Crear utilidad común |

**Advertencia**: Código que PARECE igual pero representa conceptos distintos → mantener separado.

### YAGNI - Eliminar

| Si detectas | Aplicar |
|------------|---------|
| Feature "por si acaso" | Eliminar |
| Abstracción sin uso actual | Eliminar |
| Interface con 1 implementación | Eliminar |
| Configuración no requerida | Eliminar |

### Orden de Prioridad

1. **YAGNI** → Eliminar primero
2. **KISS** → Simplificar segundo
3. **DRY** → Centralizar tercero

### Excepción: Archivos Esenciales de Proyecto

Archivos de configuración de proyecto (go.mod, package.json, pyproject.toml, Cargo.toml, build.gradle, pom.xml, etc.) **NO son YAGNI** - son necesarios para que el código funcione. Incluirlos siempre al crear proyectos nuevos.

## Formato de Salida

**IMPORTANTE**: Siempre ESCRIBIR los archivos al sistema. No solo mostrar código.

```
### Archivos Creados
- path/to/archivo.go (descripción breve)

### Correcciones Aplicadas
- [KISS] Descripción del cambio
- [YAGNI] Descripción del cambio
- [DRY] Descripción del cambio
- [LINUS] Descripción del cambio
```

**Sin preguntas. Sin confirmaciones. Escribir archivos directamente.**

## Cómo Usar

`/kiss-dry-yagni [código o descripción]`

Aplica a:
- Código nuevo generado
- Código existente a revisar
- Diseños de arquitectura

NO aplica cuando:
- No has invocado la skill
- Pides explícitamente prototipo/throwaway
