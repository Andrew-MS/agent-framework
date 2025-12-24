# Best Practices and Patterns

## Overview

This document provides best practices, design patterns, and guidelines for creating robust, maintainable declarative workflows.

## Workflow Design

### Keep Workflows Focused

✅ **DO**: Create workflows with a single, clear purpose

```yaml
# Good: Focused workflow
kind: Workflow
# Purpose: Customer onboarding process
trigger:
  kind: OnConversationStart
  id: customer_onboarding
```

❌ **AVOID**: Mixing unrelated concerns in one workflow

### Break Down Complexity

✅ **DO**: Use sub-workflows for complex processes

```yaml
# Main workflow
- kind: InvokeWorkflow
  id: validate_input
  workflow: "validation.yaml"

- kind: InvokeWorkflow
  id: process_data
  workflow: "processing.yaml"
```

❌ **AVOID**: Monolithic workflows with hundreds of actions

### Use Descriptive Names

✅ **DO**: Use clear, meaningful IDs and variable names

```yaml
- kind: SetVariable
  id: calculate_total_cost
  displayName: "Calculate Total Purchase Cost"
  variable: Local.TotalCost
  value: =Local.ItemCost + Local.TaxAmount
```

❌ **AVOID**: Cryptic or generic names

```yaml
- kind: SetVariable
  id: set_var_1
  variable: Local.X
  value: =Local.Y + Local.Z
```

## State Management

### Initialize Variables

✅ **DO**: Always initialize variables before use

```yaml
- kind: SetVariable
  id: init_counter
  variable: Local.Counter
  value: 0

- kind: SetVariable
  id: init_flag
  variable: Local.IsComplete
  value: false
```

### Clean Up State

✅ **DO**: Reset temporary variables when done

```yaml
- kind: ResetVariable
  id: clear_temp
  variable: Local.TempData
```

### Validate Before Access

✅ **DO**: Check for null/blank before accessing properties

```yaml
- kind: ConditionGroup
  id: safe_access
  conditions:
    - condition: =Not(IsBlank(Local.User)) And Not(IsBlank(Local.User.Name))
      id: has_name
      actions:
        - kind: SendActivity
          id: greet
          activity: "Hello, {Local.User.Name}!"
```

## Error Handling

### Implement Retry Logic

✅ **DO**: Add retry logic for transient failures

```yaml
- kind: SetVariable
  id: init_attempts
  variable: Local.Attempts
  value: 0

- kind: InvokeAzureAgent
  id: retry_operation
  agent:
    name: ExternalAgent
  output:
    responseObject: Local.Result

- kind: ConditionGroup
  id: check_retry
  conditions:
    - condition: =Not(Local.Result.success) And Local.Attempts < 3
      id: should_retry
      actions:
        - kind: SetVariable
          id: increment
          variable: Local.Attempts
          value: =Local.Attempts + 1
        
        - kind: GotoAction
          id: retry
          actionId: retry_operation
```

### Provide Meaningful Feedback

✅ **DO**: Send clear status messages

```yaml
- kind: SendActivity
  id: status_update
  activity: "Processing request... (Step 2 of 5)"
```

### Handle Edge Cases

✅ **DO**: Account for empty inputs, missing data

```yaml
- kind: ConditionGroup
  id: validate_input
  conditions:
    - condition: =IsBlank(System.LastMessage.Text)
      id: empty_input
      actions:
        - kind: SendActivity
          id: prompt_again
          activity: "Please provide your input."
        - kind: EndWorkflow
          id: end_early
```

## Performance

### Minimize Agent Calls

✅ **DO**: Cache agent results when appropriate

```yaml
- kind: ConditionGroup
  id: check_cache
  conditions:
    - condition: =IsBlank(Local.CachedData)
      id: cache_miss
      actions:
        - kind: InvokeAzureAgent
          id: fetch_data
          agent:
            name: DataAgent
          output:
            responseObject: Local.CachedData

# Use cached data
- kind: SendActivity
  id: use_cache
  activity: "{Local.CachedData}"
```

### Avoid Unnecessary Loops

✅ **DO**: Use efficient collection operations

```yaml
# Good: Single operation
- kind: SetVariable
  id: filter_active
  variable: Local.ActiveItems
  value: =Filter(Local.Items, isActive)
```

❌ **AVOID**: Looping when not needed

```yaml
# Avoid: Manual iteration
- kind: Foreach
  id: manual_filter
  itemsProperty: =Local.Items
  actions:
    - kind: ConditionGroup
      id: check_active
      conditions:
        - condition: =ThisItem.isActive
          id: is_active
          actions:
            # Add to list...
```

## Security

### Never Hardcode Secrets

❌ **AVOID**: Hardcoding credentials

```yaml
# BAD - Never do this
- kind: SetVariable
  id: bad_practice
  variable: Local.ApiKey
  value: "sk-12345abcdef"
```

✅ **DO**: Use secure configuration

```csharp
// Configure securely
var options = new DeclarativeWorkflowOptions
{
    Configuration = new Dictionary<string, object>
    {
        ["ApiKey"] = Environment.GetEnvironmentVariable("API_KEY")
    }
};
```

### Sanitize User Input

✅ **DO**: Validate and sanitize inputs

```yaml
- kind: ConditionGroup
  id: validate_input
  conditions:
    - condition: =Not(IsBlank(System.LastMessage.Text)) And Len(System.LastMessage.Text) <= 1000
      id: valid_input
      actions:
        # Process input
```

### Implement Authorization

✅ **DO**: Check permissions before sensitive operations

```yaml
- kind: ConditionGroup
  id: check_auth
  conditions:
    - condition: =Local.User.Role = "Admin"
      id: is_admin
      actions:
        # Perform admin operation
  
  elseActions:
    - kind: SendActivity
      id: unauthorized
      activity: "Unauthorized: Admin access required"
```

## Maintainability

### Document Complex Logic

✅ **DO**: Add comments explaining non-obvious logic

```yaml
#
# This workflow implements a three-tier approval process:
# 1. Manager approval for all requests
# 2. Director approval for requests > $10,000
# 3. VP approval for requests > $50,000
#
kind: Workflow
trigger:
  kind: OnConversationStart
  id: approval_workflow
  actions:
    # ...
```

### Version Your Workflows

✅ **DO**: Include version information

```yaml
#
# Customer Support Workflow
# Version: 2.1.0
# Last Updated: 2024-12-24
# Author: Support Team
#
kind: Workflow
```

### Use Consistent Formatting

✅ **DO**: Follow consistent YAML formatting

```yaml
# Consistent indentation (2 spaces)
- kind: ConditionGroup
  id: check_status
  conditions:
    - condition: =Local.Status = "Complete"
      id: is_complete
      actions:
        - kind: SendActivity
          id: notify
          activity: "Done!"
```

## Testing

### Test Happy Path

✅ **DO**: Verify main workflow path

```csharp
[Fact]
public async Task TestSuccessfulWorkflow()
{
    var workflow = DeclarativeWorkflowBuilder.Build("test.yaml", options);
    
    var results = new List<WorkflowEvent>();
    await foreach (var evt in workflow.RunAsync("test input"))
    {
        results.Add(evt);
    }
    
    Assert.Contains(results, e => e is WorkflowOutputEvent);
}
```

### Test Error Paths

✅ **DO**: Test error handling

```csharp
[Fact]
public async Task TestInvalidInput()
{
    var workflow = DeclarativeWorkflowBuilder.Build("test.yaml", options);
    
    await foreach (var evt in workflow.RunAsync(""))
    {
        if (evt is WorkflowErrorEvent error)
        {
            Assert.NotNull(error);
            return;
        }
    }
}
```

### Test Edge Cases

✅ **DO**: Test boundary conditions

- Empty inputs
- Maximum values
- Missing optional data
- Concurrent executions

## Monitoring

### Log Important Events

✅ **DO**: Add logging at key points

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    if (evt is DeclarativeActionInvokedEvent invoked)
    {
        logger.LogInformation($"Starting: {invoked.ActionId}");
    }
    
    if (evt is ExecutorFailedEvent failed)
    {
        logger.LogError($"Failed: {failed.ExecutorId}");
    }
}
```

### Track Metrics

✅ **DO**: Monitor workflow performance

```csharp
var metrics = new WorkflowMetrics();

await foreach (var evt in workflow.RunAsync(input))
{
    if (evt is DeclarativeActionInvokedEvent invoked)
    {
        metrics.StartAction(invoked.ActionId);
    }
    
    if (evt is DeclarativeActionCompletedEvent completed)
    {
        metrics.EndAction(completed.ActionId);
    }
}

// Report metrics
Console.WriteLine($"Total duration: {metrics.TotalDuration}ms");
Console.WriteLine($"Actions executed: {metrics.ActionCount}");
```

## Common Pitfalls

### Infinite Loops

❌ **AVOID**: Loops without proper termination

```yaml
# BAD: Can loop forever
- kind: GotoAction
  id: loop_start
  actionId: loop_start
```

✅ **DO**: Always include exit conditions

```yaml
- kind: SetVariable
  id: increment
  variable: Local.Counter
  value: =Local.Counter + 1

- kind: ConditionGroup
  id: check_limit
  conditions:
    - condition: =Local.Counter < 10
      id: continue
      actions:
        - kind: GotoAction
          id: loop
          actionId: increment
```

### Race Conditions

❌ **AVOID**: Assuming sequential execution in parallel scenarios

✅ **DO**: Design for concurrency when using parallel patterns

### Memory Leaks

❌ **AVOID**: Accumulating unbounded data

```yaml
# BAD: Grows without limit
- kind: SetVariable
  id: append_forever
  variable: Local.History
  value: =Concat(Local.History, [Local.NewItem])
```

✅ **DO**: Limit collection sizes

```yaml
# Good: Keep last N items
- kind: SetVariable
  id: append_limited
  variable: Local.History
  value: =Concat(Last(Local.History, 99), [Local.NewItem])
```

## Checklist

Before deploying a workflow, verify:

- [ ] All actions have unique, descriptive IDs
- [ ] Variables are initialized before use
- [ ] Error handling is implemented
- [ ] Loops have termination conditions
- [ ] Sensitive data is secured
- [ ] User inputs are validated
- [ ] Agent names are correct
- [ ] Checkpoint strategy is defined
- [ ] Logging is configured
- [ ] Testing is complete
- [ ] Documentation is updated

## Resources

- [YAML Schema Reference](./02-yaml-schema-reference.md)
- [Action Types Reference](./03-action-types-reference.md)
- [Examples and Sample Workflows](./11-examples-and-samples.md)

## Next Steps

- Review complete [workflow samples](../../workflow-samples/)
- Explore [.NET samples](../../dotnet/samples/GettingStarted/Workflows/)
- Join the [community Discord](https://discord.gg/b5zjErwbQM)
