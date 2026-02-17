# Análisis de Complejidad

Guía para identificar y medir complejidad en código.

## Tipos de Complejidad

### Complejidad Esencial vs Accidental

| Tipo | Descripción | Ejemplo |
|------|-------------|---------|
| **Esencial** | Inherente al problema | Validar una transacción bancaria |
| **Accidental** | Creada por nosotros | Framework de validación custom |

**Objetivo**: Reducir complejidad accidental, no esencial.

## Métricas de Complejidad

### 1. Complejidad Ciclomática

Mide el número de caminos independientes a través del código.

```
Complejidad = 1 + (número de decisiones)
```

**Decisiones**: if, else, elif, for, while, and, or, try, except

| Valor | Interpretación |
|-------|----------------|
| 1-5 | Simple |
| 6-10 | Moderado |
| 11-20 | Complejo |
| 20+ | Muy complejo, refactorizar |

### 2. Acoplamiento

Grado de dependencia entre módulos.

**Señales de alto acoplamiento**:
- Cambiar un módulo rompe otro
- Imports circulares
- Muchos parámetros entre funciones
- Tests que fallan por cambios en otros módulos

### 3. Cohesión

Grado en que elementos de un módulo pertenecen juntos.

| Nivel | Descripción |
|-------|-------------|
| **Alta** | Todo en el módulo contribuye a una responsabilidad |
| **Media** | Algunas cosas relacionadas, algunas no |
| **Baja** | Elementos sin relación clara |

**Objetivo**: Alta cohesión, bajo acoplamiento.

## Indicadores de Over-Engineering

### Nivel 1: Sospechoso
- Interface con 1 implementación
- Clase abstracta con 1 subclase
- Factory que siempre devuelve la misma clase
- Configuración para 1 caso

### Nivel 2: Probable
- Builder para 2-3 parámetros
- Strategy pattern con 1 estrategia
- Observer con 1 suscriptor
- Plugin system sin plugins

### Nivel 3: Definitivo
- "Por si en el futuro..."
- "Para que sea más flexible..."
- "Para seguir el patrón..."
- Código que nunca se ejecuta en producción

## Herramientas de Análisis

### Python
```bash
# Complejidad ciclomática
radon cc archivo.py -a

# Puntos calientes de complejidad
radon cc archivo.py -a -s

# Métricas raw
radon raw archivo.py
```

### JavaScript/TypeScript
```bash
# ESLint con reglas de complejidad
eslint --rule 'complexity: ["error", 10]' archivo.js

# Análisis con SonarJS
npx sonarjs
```

### Go
```bash
# Complejidad ciclomática
gocyclo archivo.go

# Líneas por función
gofmt -s archivo.go | gofmt -d
```

### Kotlin
```bash
# Detekt para análisis
detekt-cli --input archivo.kt
```
