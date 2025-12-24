# Workflow Execution and Runtime

## Overview

This document covers the runtime aspects of declarative workflows: building, executing, configuration, error handling, and observability.

## Building Workflows

### From File

```csharp
using Microsoft.Agents.AI.Workflows.Declarative;

var options = new DeclarativeWorkflowOptions
{
    AgentProvider = agentProvider,
    CheckpointManager = checkpointManager
};

Workflow workflow = DeclarativeWorkflowBuilder.Build(
    "Marketing.yaml",
    options
);
```

### From TextReader

```csharp
using var reader = new StringReader(yamlContent);

Workflow workflow = DeclarativeWorkflowBuilder.Build(
    reader,
    options
);
```

### With Custom Input Transform

```csharp
Workflow workflow = DeclarativeWorkflowBuilder.Build<MyInputType>(
    "workflow.yaml",
    options,
    inputTransform: input => new ChatMessage(ChatRole.User, input.Text)
);
```

## DeclarativeWorkflowOptions

### Core Properties

```csharp
public class DeclarativeWorkflowOptions
{
    // Agent provider for AI agent integration
    public WorkflowAgentProvider AgentProvider { get; set; }
    
    // Checkpoint storage for state persistence
    public ICheckpointManager? CheckpointManager { get; set; }
    
    // PowerFx configuration
    public IDictionary<string, object> Configuration { get; set; }
    
    // Additional runtime options...
}
```

### Configuration Example

```csharp
var options = new DeclarativeWorkflowOptions
{
    AgentProvider = new AzureAgentProvider(
        endpoint: "https://your-project.openai.azure.com",
        credential: new AzureCliCredential(),
        deploymentName: "gpt-4"
    ),
    
    CheckpointManager = new FileSystemJsonCheckpointStore("./checkpoints"),
    
    Configuration = new Dictionary<string, object>
    {
        ["MaxRetries"] = 3,
        ["Timeout"] = TimeSpan.FromMinutes(5),
        ["CustomSetting"] = "value"
    }
};
```

## Executing Workflows

### Basic Execution

```csharp
await foreach (var evt in workflow.RunAsync("User's input"))
{
    if (evt is WorkflowOutputEvent output && output.Is<ChatMessage>(out var msg))
    {
        Console.WriteLine(msg.Text);
    }
}
```

### With Cancellation

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));

try
{
    await foreach (var evt in workflow.RunAsync("input", cts.Token))
    {
        // Process events
    }
}
catch (OperationCanceledException)
{
    Console.WriteLine("Workflow cancelled");
}
```

### Event Processing Pattern

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    switch (evt)
    {
        case WorkflowStartedEvent started:
            Console.WriteLine("Workflow started");
            break;
        
        case DeclarativeActionInvokedEvent invoked:
            Console.WriteLine($"Action: {invoked.ActionId}");
            break;
        
        case WorkflowOutputEvent output:
            if (output.Is<ChatMessage>(out var message))
            {
                await DisplayMessage(message);
            }
            break;
        
        case ExternalInputRequest inputRequest:
            var response = await PromptUser(inputRequest);
            // Resume workflow with response
            break;
        
        case WorkflowErrorEvent error:
            LogError(error);
            break;
    }
}
```

## Error Handling

### Try-Catch Pattern

```csharp
try
{
    await foreach (var evt in workflow.RunAsync(input))
    {
        // Process events
    }
}
catch (ExecutorException ex)
{
    Console.WriteLine($"Executor failed: {ex.ExecutorId}");
    Console.WriteLine($"Error: {ex.Message}");
}
catch (WorkflowException ex)
{
    Console.WriteLine($"Workflow error: {ex.Message}");
}
catch (Exception ex)
{
    Console.WriteLine($"Unexpected error: {ex.Message}");
}
```

### Error Events

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    switch (evt)
    {
        case ExecutorFailedEvent failed:
            Console.WriteLine($"Executor {failed.ExecutorId} failed");
            if (failed.Data is Exception ex)
            {
                Console.WriteLine($"  Exception: {ex.Message}");
            }
            // Decide whether to continue or abort
            break;
        
        case WorkflowErrorEvent workflowError:
            Console.WriteLine($"Workflow error: {workflowError.Data}");
            // Log and potentially retry or compensate
            break;
        
        case WorkflowWarningEvent warning:
            Console.WriteLine($"Warning: {warning.Data}");
            // Log but continue
            break;
    }
}
```

## Resuming from Checkpoint

### Load and Resume

```csharp
string runId = "workflow-run-123";
string checkpointId = "checkpoint-xyz";

// Load checkpoint
var checkpoint = await checkpointManager.GetCheckpointAsync(runId, checkpointId);

if (checkpoint != null)
{
    // Rebuild workflow
    var workflow = DeclarativeWorkflowBuilder.Build("workflow.yaml", options);
    
    // Resume from checkpoint
    var resumedWorkflow = workflow.ResumeFromCheckpoint(checkpoint);
    
    // Continue execution (with or without additional input)
    await foreach (var evt in resumedWorkflow.ResumeAsync())
    {
        // Process events
    }
}
```

### Resume with Input

```csharp
// Resume and provide external input
var response = new ExternalInputResponse("User's response");

await foreach (var evt in resumedWorkflow.ResumeAsync(response))
{
    // Process events
}
```

## Observability

### Logging Events

```csharp
using Microsoft.Extensions.Logging;

ILogger logger = loggerFactory.CreateLogger("Workflow");

await foreach (var evt in workflow.RunAsync(input))
{
    logger.LogInformation($"Event: {evt.GetType().Name}");
    
    if (evt is DeclarativeActionInvokedEvent invoked)
    {
        logger.LogInformation($"  Action: {invoked.ActionId} ({invoked.ActionType})");
    }
    
    if (evt is WorkflowErrorEvent error)
    {
        logger.LogError($"Error: {error.Data}");
    }
}
```

### Telemetry Integration

```csharp
using System.Diagnostics;

var activitySource = new ActivitySource("DeclarativeWorkflow");

await foreach (var evt in workflow.RunAsync(input))
{
    if (evt is DeclarativeActionInvokedEvent invoked)
    {
        using var activity = activitySource.StartActivity(
            $"Action.{invoked.ActionType}",
            ActivityKind.Internal
        );
        
        activity?.SetTag("action.id", invoked.ActionId);
        activity?.SetTag("action.type", invoked.ActionType);
    }
}
```

### Performance Monitoring

```csharp
var stopwatch = Stopwatch.StartNew();
var actionTimings = new Dictionary<string, long>();

await foreach (var evt in workflow.RunAsync(input))
{
    switch (evt)
    {
        case DeclarativeActionInvokedEvent invoked:
            actionTimings[invoked.ActionId] = stopwatch.ElapsedMilliseconds;
            break;
        
        case DeclarativeActionCompletedEvent completed:
            if (actionTimings.TryGetValue(completed.ActionId, out var startTime))
            {
                var duration = stopwatch.ElapsedMilliseconds - startTime;
                Console.WriteLine($"{completed.ActionId}: {duration}ms");
            }
            break;
    }
}
```

## Code Generation (Ejection)

Convert YAML workflow to C# code:

```csharp
string sourceCode = DeclarativeWorkflowBuilder.Eject(
    "workflow.yaml",
    DeclarativeWorkflowLanguage.CSharp,
    workflowNamespace: "MyCompany.Workflows",
    workflowPrefix: "Generated"
);

File.WriteAllText("GeneratedWorkflow.cs", sourceCode);
```

## Best Practices

1. **Configure Timeouts** - Set reasonable timeouts for workflow execution
2. **Handle Cancellation** - Support cancellation tokens
3. **Log Events** - Capture important events for debugging
4. **Monitor Performance** - Track action execution times
5. **Validate Options** - Check configuration before building
6. **Secure Credentials** - Use secure credential stores
7. **Test Error Paths** - Verify error handling works correctly
8. **Use Checkpoints** - Enable checkpointing for long-running workflows
9. **Version Workflows** - Track workflow YAML versions
10. **Document Dependencies** - List required agents and services

## Troubleshooting

### Workflow Won't Start

```csharp
// Check options are configured
if (options.AgentProvider == null)
{
    throw new InvalidOperationException("AgentProvider is required");
}

// Verify YAML is valid
try
{
    var workflow = DeclarativeWorkflowBuilder.Build("workflow.yaml", options);
}
catch (YamlException ex)
{
    Console.WriteLine($"YAML parse error: {ex.Message}");
}
```

### Agent Not Found

```yaml
# Verify agent name matches exactly
- kind: InvokeAzureAgent
  agent:
    name: "ResearchAgent"  # Must match deployed agent name
```

### Checkpoint Load Fails

```csharp
var checkpoint = await checkpointManager.GetCheckpointAsync(runId, checkpointId);
if (checkpoint == null)
{
    Console.WriteLine("Checkpoint not found - starting from beginning");
    // Start new workflow run
}
```

## Next Steps

- See [Examples and Sample Workflows](./11-examples-and-samples.md)
- Review [Best Practices and Patterns](./12-best-practices-and-patterns.md)
- Explore sample workflows in the [workflow-samples directory](../../workflow-samples/)
