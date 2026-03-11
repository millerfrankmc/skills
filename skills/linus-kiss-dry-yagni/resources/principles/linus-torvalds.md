# Principios de Linus Torvalds

## Código sobre Explicaciones

- Código que se explica solo > código comentado
- Comentarios solo para el **POR QUÉ**, nunca para el QUÉ
- Si necesitas explicar qué hace → el código está mal

## Anti-Clever Code

| Malo | Bueno |
|------|-------|
| One-liner cryptico | Código expandido legible |
| "Mira qué corto" | Warning sign |
| Elegante | Obvio |

**Regla**: El próximo desarrollador (tú en 6 meses) debe entenderlo de un vistazo.

## Simplicidad Brutal

- Dos formas → la más simple, SIEMPRE
- No optimizar hasta que duela
- "Enterprise thinking" → prohibido
- Framework/abstracción → solo si el dolor es real y actual

## Pragmatismo sobre Perfección

- Funciona hoy > perfecto mañana
- Iterar > diseñar en exceso
- Eliminar código > agregar código
- El mejor código es el que no existe

## Calidad por Revisión

- Código que otros pueden revisar fácilmente
- Funciones cortas, una cosa bien
- Estructura plana > estructura profunda
