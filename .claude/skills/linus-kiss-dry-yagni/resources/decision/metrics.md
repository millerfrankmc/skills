# Métricas Objetivas

## Umbrales de Complejidad

| Métrica | Verde | Amarillo | Rojo |
|---------|-------|----------|------|
| **Líneas por función** | 1-20 | 21-50 | 50+ |
| **Niveles de anidamiento** | 1-2 | 3 | 4+ |
| **Parámetros por función** | 1-3 | 4 | 5+ |
| **Complejidad ciclomática** | 1-5 | 6-10 | 10+ |
| **Dependencias por archivo** | 1-5 | 6-10 | 10+ |
| **Funciones por clase** | 1-10 | 11-20 | 20+ |

## Cómo Medir

```bash
# Complejidad ciclomática (Python)
radon cc archivo.py -a

# Líneas de código
wc -l archivo.py

# Dependencias
grep -c "^import\|^from" archivo.py
```
