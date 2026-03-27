# Decision Framework

## Red Flags - Too Complex

- ❌ You can't explain it in 2-3 sentences
- ❌ Requires extensive documentation to understand
- ❌ New developers take weeks to modify it
- ❌ Has many interdependencies and side effects
- ❌ Bugs consistently appear in this component
- ❌ Deep nesting (>3 levels)
- ❌ Multiple abstractions layered on top of each other
- ❌ Over-engineered for current and foreseeable needs

## Green Flags - Appropriately Simple

- ✅ You can explain it clearly in 1-2 minutes
- ✅ New developers understand it quickly
- ✅ Does one thing well (single responsibility)
- ✅ Explicit and minimal dependencies
- ✅ Stable with few bugs
- ✅ Readable and self-documenting code
- ✅ Serves current needs without speculation

## KISS vs Over-Engineering

| Aspect | KISS | Over-Engineering |
|--------|------|------------------|
| **Scope** | Solves current problem | Anticipates future needs |
| **Code** | ~100-200 LOC | ~500+ LOC |
| **Time** | 1-2 weeks | 4-8+ weeks |
| **Maintenance** | Easy to modify | Complex to change |
| **Performance** | Sufficient | Highly optimized |
| **Bugs** | Few, obvious | Many, hidden |
