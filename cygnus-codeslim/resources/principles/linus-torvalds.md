# Linus Torvalds Principles

## Code over Explanations

- Self-explanatory code > commented code
- Comments only for the **WHY**, never for the WHAT
- If you need to explain what it does -> the code is wrong

## Anti-Clever Code

| Bad | Good |
|-----|------|
| Cryptic one-liner | Expanded readable code |
| "Look how short" | Warning sign |
| Elegant | Obvious |

**Rule**: The next developer (you in 6 months) must understand it at a glance.

## Brutal Simplicity

- Two ways -> the simplest one, ALWAYS
- Don't optimize until it hurts
- "Enterprise thinking" -> forbidden
- Framework/abstraction -> only if the pain is real and current

## Pragmatism over Perfection

- Works today > perfect tomorrow
- Iterate > over-design
- Delete code > add code
- The best code is the one that doesn't exist

## Quality through Review

- Code that others can review easily
- Short functions, one thing well
- Flat structure > deep structure
