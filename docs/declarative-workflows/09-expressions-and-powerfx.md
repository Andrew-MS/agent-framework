# Expressions and PowerFx

## Overview

PowerFx is the low-code formula language used in declarative workflows for expressions, calculations, and data transformations. All expressions are prefixed with `=`.

## Basic Expressions

### Literal Values

```yaml
# Number
value: =42

# String
value: ="Hello"

# Boolean
value: =true

# Null
value: =Blank()
```

### Variable References

```yaml
# Local variable
value: =Local.Counter

# System variable
value: =System.ConversationId

# Property access
value: =Local.User.Name
```

## Operators

### Arithmetic

```yaml
value: =Local.A + Local.B      # Addition
value: =Local.A - Local.B      # Subtraction
value: =Local.A * Local.B      # Multiplication
value: =Local.A / Local.B      # Division
value: =Mod(Local.A, Local.B)  # Modulo
```

### Comparison

```yaml
condition: =Local.A = Local.B   # Equals
condition: =Local.A <> Local.B  # Not equals
condition: =Local.A > Local.B   # Greater than
condition: =Local.A >= Local.B  # Greater or equal
condition: =Local.A < Local.B   # Less than
condition: =Local.A <= Local.B  # Less or equal
```

### Logical

```yaml
condition: =Local.A And Local.B      # Logical AND
condition: =Local.A Or Local.B       # Logical OR
condition: =Not(Local.A)             # Logical NOT
condition: =Local.A And Not(Local.B) # Combined
```

## String Functions

```yaml
# Concatenation
value: =Local.FirstName & " " & Local.LastName

# String interpolation (in activity strings)
activity: "Hello, {Local.Name}!"

# Upper/Lower case
value: =Upper(Local.Text)
value: =Lower(Local.Text)

# Trim
value: =Trim(Local.Text)

# Substring
value: =Mid(Local.Text, 1, 5)

# Find
value: =Find("search", Local.Text)

# Replace
value: =Substitute(Local.Text, "old", "new")

# Length
value: =Len(Local.Text)
```

## Collection Functions

### Search and Filter

```yaml
# Find item in array
value: =Search(Local.Items, "targetName", name)

# Filter array
value: =Filter(Local.Items, Status = "Active")

# First item
value: =First(Local.Items)

# Last item
value: =Last(Local.Items)
```

### Transformation

```yaml
# ForAll (map)
value: =ForAll(Local.Items, Upper(name))

# Concatenate with separator
value: =Concat(Local.Items, name, ", ")

# Count
value: =CountRows(Local.Items)
```

### Creation

```yaml
# Create array
value: =-
  =[
    "Item1",
    "Item2",
    "Item3"
  ]

# Create object
value: |-
  ={
    name: "Value",
    count: 42,
    nested: {key: "value"}
  }
```

## Conditional Functions

### If

```yaml
# Simple if
value: =If(Local.Score > 80, "Pass", "Fail")

# Nested if
value: =If(
  Local.Score >= 90, "A",
  Local.Score >= 80, "B",
  Local.Score >= 70, "C",
  "F"
)
```

### IsBlank

```yaml
# Check if variable is empty/null
condition: =IsBlank(Local.OptionalValue)

# Check if not blank
condition: =Not(IsBlank(Local.RequiredValue))
```

## Type Functions

### Conversion

```yaml
# To text
value: =Text(Local.Number)

# To number
value: =Value(Local.TextNumber)

# To boolean
value: =Boolean(Local.Value)
```

### Type Checking

```yaml
# Check if number
condition: =IsNumeric(Local.Value)

# Check if blank
condition: =IsBlank(Local.Value)
```

## Date/Time Functions

```yaml
# Current date/time
value: =Now()

# Today (date only)
value: =Today()

# Add days
value: =DateAdd(Today(), 7, Days)

# Date difference
value: =DateDiff(Local.StartDate, Local.EndDate, Days)

# Format date
value: =Text(Local.Date, "yyyy-MM-dd")
```

## Math Functions

```yaml
# Absolute value
value: =Abs(Local.Number)

# Round
value: =Round(Local.Number, 2)

# Min/Max
value: =Min(Local.A, Local.B, Local.C)
value: =Max(Local.A, Local.B, Local.C)

# Sum
value: =Sum(Local.Numbers)

# Average
value: =Average(Local.Numbers)
```

## Custom Workflow Functions

### UserMessage

Create a user message:

```yaml
value: =UserMessage(Local.Text)
value: =UserMessage("Static text")
```

### MessageText

Extract text from a message:

```yaml
value: =MessageText(Local.AgentResponse)
```

## Common Patterns

### Null Coalescing

```yaml
# Use default if blank
value: =If(IsBlank(Local.Value), "DefaultValue", Local.Value)
```

### Safe Property Access

```yaml
# Check before accessing
value: =If(
  Not(IsBlank(Local.Object)),
  Local.Object.Property,
  "Not available"
)
```

### Array Building

```yaml
# Start with empty
- kind: SetVariable
  id: init
  variable: Local.Results
  value: =[]

# Add items
- kind: SetVariable
  id: append
  variable: Local.Results
  value: =Concat(Local.Results, [Local.NewItem])
```

### Complex Conditions

```yaml
condition: |-
  =And(
    Local.Status = "Active",
    Local.Count > 0,
    Not(IsBlank(Local.Name))
  )
```

## Expression Best Practices

1. **Keep It Simple** - Break complex expressions into multiple steps
2. **Use Variables** - Store intermediate results
3. **Add Whitespace** - Format multi-line expressions for readability
4. **Validate Nulls** - Check for blank values before access
5. **Type Safety** - Ensure correct types in operations
6. **Test Expressions** - Validate formulas before deployment

## PowerFx Resources

- [PowerFx Overview](https://learn.microsoft.com/power-platform/power-fx/overview)
- [Formula Reference](https://learn.microsoft.com/power-platform/power-fx/formula-reference)
- [Expression Grammar](https://learn.microsoft.com/power-platform/power-fx/expression-grammar)

## Next Steps

- Understand [Workflow Execution and Runtime](./10-workflow-execution-and-runtime.md)
- See [Examples and Sample Workflows](./11-examples-and-samples.md)
- Review [Best Practices and Patterns](./12-best-practices-and-patterns.md)
