# Framework de Decisión

## Red Flags - Demasiado Complejo

- ❌ No puedes explicarlo en 2-3 oraciones
- ❌ Requiere documentación extensa para entender
- ❌ Nuevos desarrolladores tardan semanas en modificarlo
- ❌ Tiene muchas interdependencias y efectos secundarios
- ❌ Los bugs aparecen consistentemente en este componente
- ❌ Anidamiento profundo (>3 niveles)
- ❌ Múltiples abstracciones una sobre otra
- ❌ Over-engineered para necesidades actuales y previsibles

## Green Flags - Apropiadamente Simple

- ✅ Puedes explicarlo claramente en 1-2 minutos
- ✅ Nuevos desarrolladores lo entienden rápido
- ✅ Hace una cosa bien (responsabilidad única)
- ✅ Dependencias explícitas y mínimas
- ✅ Estable con pocos bugs
- ✅ Código legible y autodocumentado
- ✅ Sirve necesidades actuales sin especulación

## KISS vs Over-Engineering

| Aspecto | KISS | Over-Engineering |
|---------|------|------------------|
| **Scope** | Resuelve problema actual | Anticipa necesidades futuras |
| **Código** | ~100-200 LOC | ~500+ LOC |
| **Tiempo** | 1-2 semanas | 4-8+ semanas |
| **Mantenimiento** | Fácil de modificar | Complejo de cambiar |
| **Performance** | Suficiente | Altamente optimizado |
| **Bugs** | Pocos, obvios | Muchos, ocultos |
