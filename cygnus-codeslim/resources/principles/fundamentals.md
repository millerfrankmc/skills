# KISS-DRY-YAGNI Fundamentals

This resource delves into the fundamental principles for those who want to understand them better.

## KISS - Keep It Simple, Stupid

### Origin
Principle designed by the US Navy in 1960. Simplicity should be the main goal.

### Why it matters
- Simple code = fewer bugs
- Simple code = easier to maintain
- Simple code = easier to understand
- Simple code = faster to develop

### The trap
"Simple" doesn't mean "primitive". It means:
- Easy to understand
- Easy to modify
- Easy to test
- Does one thing well

## DRY - Don't Repeat Yourself

### Origin
Coined by Andy Hunt and Dave Thomas in "The Pragmatic Programmer" (1999).

### The rule
"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."

### Key distinction
- **DRY**: Same concept, same code -> centralize
- **Not DRY**: Similar code, different concepts -> keep separate

### Distinction example
```python
# Same concept - DRY appropriate
def validate_email(email): ...
def validate_user_email(user): return validate_email(user.email)
def validate_admin_email(admin): return validate_email(admin.email)

# Different concepts - DO NOT apply DRY
def calculate_user_tax(user): ...  # User tax logic
def calculate_product_tax(product): ...  # Product tax logic
# Although they look similar, they are different domains
```

## YAGNI - You Aren't Gonna Need It

### Origin
Principle from Extreme Programming (XP). Attributed to Ron Jeffries.

### The philosophy
"Always implement things when you actually need them, never when you just foresee that you need them."

### Why it's hard
- Developers want to "do it right from the start"
- We fear that "it will be harder later"
- We like elegant architecture

### The reality
- 80% of "just in case" features are never used
- Adding flexibility has a cost now
- When you need the flexibility, it's no longer the right way

### Need test
Before adding something, ask yourself:
1. Do I need it TODAY?
2. Is there an explicit requirement?
3. Is there a user waiting for this?
4. Is the pain of not having it real and current?

If all are "no" -> YAGNI, don't implement it.
