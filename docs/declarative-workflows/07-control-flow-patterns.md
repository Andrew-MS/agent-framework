# Control Flow Patterns

## Overview

Control flow actions enable you to build complex, branching workflows with conditional logic, loops, and non-linear execution paths. This document covers common patterns and best practices.

## Conditional Branching

### Simple If-Then

```yaml
- kind: ConditionGroup
  id: check_status
  conditions:
    - condition: =Local.Status = "Complete"
      id: if_complete
      actions:
        - kind: SendActivity
          id: send_success
          activity: "Task completed!"
```

### If-Then-Else

```yaml
- kind: ConditionGroup
  id: check_value
  conditions:
    - condition: =Local.Value > 10
      id: if_high
      actions:
        - kind: SendActivity
          id: send_high
          activity: "Value is high"
  
  elseActions:
    - kind: SendActivity
      id: send_low
      activity: "Value is low or equal to 10"
```

### Multiple Conditions (If-Else-If)

```yaml
- kind: ConditionGroup
  id: classify
  conditions:
    - condition: =Local.Score >= 90
      id: grade_a
      actions:
        - kind: SetVariable
          id: set_grade_a
          variable: Local.Grade
          value: "A"
    
    - condition: =Local.Score >= 80
      id: grade_b
      actions:
        - kind: SetVariable
          id: set_grade_b
          variable: Local.Grade
          value: "B"
    
    - condition: =Local.Score >= 70
      id: grade_c
      actions:
        - kind: SetVariable
          id: set_grade_c
          variable: Local.Grade
          value: "C"
  
  elseActions:
    - kind: SetVariable
      id: set_grade_f
      variable: Local.Grade
      value: "F"
```

## Loops

### Simple Counter Loop

```yaml
# Initialize counter
- kind: SetVariable
  id: init_count
  variable: Local.Counter
  value: 0

# Loop body
- kind: SendActivity
  id: loop_start
  activity: "Iteration {Local.Counter}"

# Increment
- kind: SetVariable
  id: increment
  variable: Local.Counter
  value: =Local.Counter + 1

# Check condition
- kind: ConditionGroup
  id: check_continue
  conditions:
    - condition: =Local.Counter < 5
      id: continue_loop
      actions:
        - kind: GotoAction
          id: repeat
          actionId: loop_start
```

### Foreach Loop

```yaml
- kind: Foreach
  id: process_items
  itemsProperty: =Local.Items
  actions:
    - kind: SendActivity
      id: process_item
      activity: "Processing: {ThisItem.name}"
    
    - kind: InvokeAzureAgent
      id: invoke_per_item
      agent:
        name: =ThisItem.agentName
      input:
        messages: =UserMessage(Local.Task)
```

### Loop with Break

```yaml
- kind: Foreach
  id: search_loop
  itemsProperty: =Local.Items
  actions:
    - kind: ConditionGroup
      id: check_found
      conditions:
        - condition: =ThisItem.id = Local.SearchId
          id: found_it
          actions:
            - kind: SetVariable
              id: save_result
              variable: Local.Result
              value: =ThisItem
            
            - kind: BreakLoop
              id: exit_loop
```

### Loop with Continue

```yaml
- kind: Foreach
  id: filter_process
  itemsProperty: =Local.Items
  actions:
    # Skip inactive items
    - kind: ConditionGroup
      id: check_active
      conditions:
        - condition: =Not(ThisItem.isActive)
          id: skip_inactive
          actions:
            - kind: ContinueLoop
              id: skip
    
    # Process active items
    - kind: SendActivity
      id: process
      activity: "Processing: {ThisItem.name}"
```

## Complex Patterns

### Retry Pattern

```yaml
# Initialize
- kind: SetVariable
  id: init_attempt
  variable: Local.Attempt
  value: 0

- kind: SetVariable
  id: init_success
  variable: Local.Success
  value: false

# Retry loop
- kind: InvokeAzureAgent
  id: attempt_operation
  agent:
    name: OperationAgent
  output:
    responseObject: Local.Result

# Check result
- kind: ConditionGroup
  id: check_result
  conditions:
    - condition: =Local.Result.success
      id: operation_succeeded
      actions:
        - kind: SetVariable
          id: mark_success
          variable: Local.Success
          value: true

# Increment attempt
- kind: SetVariable
  id: increment_attempt
  variable: Local.Attempt
  value: =Local.Attempt + 1

# Retry if needed
- kind: ConditionGroup
  id: retry_check
  conditions:
    - condition: =Not(Local.Success) And Local.Attempt < 3
      id: should_retry
      actions:
        - kind: SendActivity
          id: notify_retry
          activity: "Retrying... (attempt {Local.Attempt})"
        
        - kind: GotoAction
          id: goto_retry
          actionId: attempt_operation
```

### State Machine Pattern

```yaml
- kind: SetVariable
  id: init_state
  variable: Local.State
  value: "Start"

- kind: ConditionGroup
  id: state_machine
  conditions:
    - condition: =Local.State = "Start"
      id: state_start
      actions:
        - kind: SendActivity
          id: start_msg
          activity: "Starting..."
        - kind: SetVariable
          id: to_processing
          variable: Local.State
          value: "Processing"
        - kind: GotoAction
          id: continue_sm
          actionId: state_machine
    
    - condition: =Local.State = "Processing"
      id: state_processing
      actions:
        - kind: InvokeAzureAgent
          id: process
          agent:
            name: ProcessorAgent
          output:
            responseObject: Local.ProcessResult
        
        - kind: ConditionGroup
          id: check_process_result
          conditions:
            - condition: =Local.ProcessResult.success
              id: process_success
              actions:
                - kind: SetVariable
                  id: to_complete
                  variable: Local.State
                  value: "Complete"
          
          elseActions:
            - kind: SetVariable
              id: to_failed
              variable: Local.State
              value: "Failed"
        
        - kind: GotoAction
          id: continue_sm2
          actionId: state_machine
    
    - condition: =Local.State = "Complete"
      id: state_complete
      actions:
        - kind: SendActivity
          id: complete_msg
          activity: "Process completed successfully!"
    
    - condition: =Local.State = "Failed"
      id: state_failed
      actions:
        - kind: SendActivity
          id: failed_msg
          activity: "Process failed."
```

### Circuit Breaker Pattern

```yaml
- kind: SetVariable
  id: init_failures
  variable: Local.FailureCount
  value: 0

- kind: SetVariable
  id: init_circuit_open
  variable: Local.CircuitOpen
  value: false

# Check circuit breaker
- kind: ConditionGroup
  id: check_circuit
  conditions:
    - condition: =Local.CircuitOpen
      id: circuit_open
      actions:
        - kind: SendActivity
          id: circuit_msg
          activity: "Circuit breaker is open. Service unavailable."
        - kind: EndWorkflow
          id: end_circuit

# Attempt operation
- kind: InvokeAzureAgent
  id: risky_operation
  agent:
    name: ExternalAgent
  output:
    responseObject: Local.OpResult

# Check result
- kind: ConditionGroup
  id: check_op_result
  conditions:
    - condition: =Local.OpResult.success
      id: op_success
      actions:
        # Reset failure count on success
        - kind: SetVariable
          id: reset_failures
          variable: Local.FailureCount
          value: 0
  
  elseActions:
    # Increment failure count
    - kind: SetVariable
      id: inc_failures
      variable: Local.FailureCount
      value: =Local.FailureCount + 1
    
    # Open circuit if threshold reached
    - kind: ConditionGroup
      id: check_threshold
      conditions:
        - condition: =Local.FailureCount >= 5
          id: threshold_reached
          actions:
            - kind: SetVariable
              id: open_circuit
              variable: Local.CircuitOpen
              value: true
            
            - kind: SendActivity
              id: circuit_opened_msg
              activity: "Circuit breaker opened after {Local.FailureCount} failures"
```

## Best Practices

1. **Avoid Infinite Loops** - Always include termination conditions
2. **Limit Loop Iterations** - Use counters to prevent runaway loops
3. **Use Descriptive Conditions** - Make boolean expressions clear
4. **Handle Edge Cases** - Account for empty collections, null values
5. **Add Logging** - Use SendActivity to track control flow
6. **Keep It Simple** - Break complex logic into smaller workflows
7. **Document State Transitions** - Comment state machine transitions

## Next Steps

- Learn about [Human-in-the-Loop and External Input](./08-human-in-the-loop.md)
- Understand [Expressions and PowerFx](./09-expressions-and-powerfx.md)
- See [Examples and Sample Workflows](./11-examples-and-samples.md)
