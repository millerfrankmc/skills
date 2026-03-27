# Complexity Analysis

Guide to identify and measure complexity in code.

## Types of Complexity

### Essential vs Accidental Complexity

| Type | Description | Example |
|------|-------------|---------|
| **Essential** | Inherent to the problem | Validate a bank transaction |
| **Accidental** | Created by us | Custom validation framework |

**Goal:** Reduce accidental complexity, not essential.

## Complexity Metrics

### 1. Cyclomatic Complexity

Measures the number of independent paths through code.

```
Complexity = 1 + (number of decisions)
```

**Decisions**: if, else, elif, for, while, and, or, try, except

| Value | Interpretation |
|-------|----------------|
| 1-5 | Simple |
| 6-10 | Moderate |
| 11-20 | Complex |
| 20+ | Very complex, refactor |

### 2. Coupling

Degree of dependency between modules.

**Signs of high coupling**:
- Changing one module breaks another
- Circular imports
- Many parameters between functions
- Tests that fail due to changes in other modules

### 3. Cohesion

Degree to which elements of a module belong together.

| Level | Description |
|-------|-------------|
| **High** | Everything in the module contributes to one responsibility |
| **Medium** | Some things related, some not |
| **Low** | Elements without clear relation |

**Goal:** High cohesion, low coupling.

## Over-Engineering Indicators

### Level 1: Suspicious
- Interface with 1 implementation
- Abstract class with 1 subclass
- Factory that always returns the same class
- Configuration for 1 case

### Level 2: Probable
- Builder for 2-3 parameters
- Strategy pattern with 1 strategy
- Observer with 1 subscriber
- Plugin system without plugins

### Level 3: Definitive
- "Just in case in the future..."
- "To make it more flexible..."
- "To follow the pattern..."
- Code that never runs in production

## Analysis Tools

### Python
```bash
# Cyclomatic complexity
radon cc file.py -a

# Complexity hotspots
radon cc file.py -a -s

# Raw metrics
radon raw file.py
```

### JavaScript/TypeScript
```bash
# ESLint with complexity rules
eslint --rule 'complexity: ["error", 10]' file.js

# Analysis with SonarJS
npx sonarjs
```

### Go
```bash
# Cyclomatic complexity
gocyclo file.go

# Lines per function
gofmt -s file.go | gofmt -d
```

### Kotlin
```bash
# Detekt for analysis
detekt-cli --input file.kt
```
