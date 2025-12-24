# Agent Providers and Integration

## Overview

Agent providers enable declarative workflows to interact with AI agent services. The `WorkflowAgentProvider` abstract class defines the contract for integrating various agent platforms into your workflows.

## WorkflowAgentProvider Base Class

```csharp
public abstract class WorkflowAgentProvider
{
    // Function tools that agents can invoke
    public IEnumerable<AIFunction>? Functions { get; init; }
    
    // Allow parallel function execution
    public bool AllowConcurrentInvocation { get; init; }
    
    // Allow multiple tool calls per response
    public bool AllowMultipleToolCalls { get; init; }
    
    // Core operations
    public abstract Task<string> CreateConversationAsync(CancellationToken cancellationToken);
    public abstract Task<ChatMessage> CreateMessageAsync(string conversationId, ChatMessage message, CancellationToken cancellationToken);
    public abstract Task<ChatMessage> GetMessageAsync(string conversationId, string messageId, CancellationToken cancellationToken);
    public abstract IAsyncEnumerable<ChatMessage> GetMessagesAsync(string conversationId, ...);
    public abstract IAsyncEnumerable<AgentRunResponseUpdate> InvokeAgentAsync(string agentId, ...);
}
```

## Azure AI Foundry Integration

### Setup

```csharp
using Microsoft.Agents.AI.Workflows.Declarative.AzureAI;

var agentProvider = new AzureAgentProvider(
    projectEndpoint: "https://your-project.openai.azure.com",
    credential: new AzureCliCredential(),
    deploymentName: "gpt-4"
);

var options = new DeclarativeWorkflowOptions
{
    AgentProvider = agentProvider
};
```

### Configuration Options

**Environment Variables:**
- `FOUNDRY_PROJECT_ENDPOINT` - Azure Foundry project endpoint
- `FOUNDRY_MODEL_DEPLOYMENT_NAME` - Model deployment name
- `FOUNDRY_CONNECTION_GROUNDING_TOOL` - Bing grounding connection name

**User Secrets (for development):**

```bash
cd dotnet/samples/GettingStarted/Workflows/Declarative/ExecuteWorkflow
dotnet user-secrets set "FOUNDRY_PROJECT_ENDPOINT" "https://..."
dotnet user-secrets set "FOUNDRY_MODEL_DEPLOYMENT_NAME" "gpt-4"
dotnet user-secrets set "FOUNDRY_CONNECTION_GROUNDING_TOOL" "mybinggrounding"
```

### Authentication

```csharp
// Using Azure CLI
var credential = new AzureCliCredential();

// Using Managed Identity
var credential = new ManagedIdentityCredential();

// Using Client Secret
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);

var provider = new AzureAgentProvider(endpoint, credential, deployment);
```

## Conversation Management

### Creating Conversations

```yaml
# Create a new conversation
- kind: CreateConversation
  id: create_conv
  conversationId: Local.NewConversationId

# Use the conversation
- kind: InvokeAzureAgent
  id: invoke_in_conv
  conversationId: =Local.NewConversationId
  agent:
    name: MyAgent
```

### Adding Messages

```yaml
- kind: AddConversationMessage
  id: add_message
  conversationId: =System.ConversationId
  message: =UserMessage("Additional context")
```

### Retrieving Messages

```yaml
- kind: RetrieveConversationMessages
  id: get_history
  conversationId: =System.ConversationId
  limit: 20
  newestFirst: true
  output: Local.MessageHistory
```

### Copying Conversations

```yaml
- kind: CopyConversationMessages
  id: copy_conv
  sourceConversationId: =System.ConversationId
  targetConversationId: =Local.ArchiveConversationId
  limit: 100
```

## Agent Invocation

### Basic Invocation

```yaml
- kind: InvokeAzureAgent
  id: invoke_agent
  agent:
    name: ResearchAgent
    version: "1.0"
  input:
    messages: =UserMessage(Local.Query)
  output:
    messages: Local.Response
```

### With Input Arguments

```yaml
- kind: InvokeAzureAgent
  id: invoke_with_args
  agent:
    name: DataAgent
  input:
    messages: =UserMessage(Local.Query)
    arguments:
      context: =Local.Context
      depth: "detailed"
      maxResults: 10
  output:
    responseObject: Local.AgentData
```

### With External Loop

```yaml
- kind: InvokeAzureAgent
  id: invoke_loop
  agent:
    name: InteractiveAgent
  input:
    externalLoop:
      when: =Not(Local.IsComplete)
  output:
    autoSend: true
    responseObject: Local.Status
```

## Function Tools

### Providing Function Tools

```csharp
var tools = new[]
{
    AIFunctionFactory.Create((string query) => SearchDatabase(query), "search"),
    AIFunctionFactory.Create((int id) => GetUser(id), "get_user"),
};

var provider = new AzureAgentProvider(endpoint, credential, deployment)
{
    Functions = tools,
    AllowConcurrentInvocation = true,
    AllowMultipleToolCalls = true
};
```

### External Tool Execution

When an agent requests a tool not in the `Functions` collection, an `ExternalInputRequest` is emitted:

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    if (evt is ExternalInputRequest request)
    {
        // Extract function call
        var functionCall = request.AgentResponse.Message.Contents
            .OfType<FunctionCallContent>()
            .FirstOrDefault();
        
        if (functionCall != null)
        {
            // Execute function
            var result = await ExecuteFunction(functionCall.Name, functionCall.Arguments);
            
            // Create response
            var response = new ExternalInputResponse(
                new ChatMessage(ChatRole.Tool, 
                    new FunctionResultContent(functionCall.CallId, result))
            );
            
            // Resume workflow
            // (implementation depends on workflow API)
        }
    }
}
```

## Custom Agent Provider

### Implementation

```csharp
public class CustomAgentProvider : WorkflowAgentProvider
{
    private readonly IAgentService _agentService;
    
    public CustomAgentProvider(IAgentService agentService)
    {
        _agentService = agentService;
    }
    
    public override async Task<string> CreateConversationAsync(
        CancellationToken cancellationToken = default)
    {
        return await _agentService.CreateConversationAsync();
    }
    
    public override async Task<ChatMessage> CreateMessageAsync(
        string conversationId, 
        ChatMessage message, 
        CancellationToken cancellationToken = default)
    {
        return await _agentService.AddMessageAsync(conversationId, message);
    }
    
    public override async Task<ChatMessage> GetMessageAsync(
        string conversationId, 
        string messageId, 
        CancellationToken cancellationToken = default)
    {
        return await _agentService.GetMessageAsync(conversationId, messageId);
    }
    
    public override async IAsyncEnumerable<ChatMessage> GetMessagesAsync(
        string conversationId,
        int? limit = null,
        string? after = null,
        string? before = null,
        bool newestFirst = false,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var message in _agentService.GetMessagesAsync(conversationId, limit))
        {
            yield return message;
        }
    }
    
    public override async IAsyncEnumerable<AgentRunResponseUpdate> InvokeAgentAsync(
        string agentId,
        string? agentVersion,
        string? conversationId,
        IEnumerable<ChatMessage>? messages,
        IDictionary<string, object?>? inputArguments,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var update in _agentService.InvokeAsync(agentId, conversationId, messages))
        {
            yield return update;
        }
    }
}
```

### Usage

```csharp
var customProvider = new CustomAgentProvider(myAgentService);

var options = new DeclarativeWorkflowOptions
{
    AgentProvider = customProvider
};

var workflow = DeclarativeWorkflowBuilder.Build("workflow.yaml", options);
```

## Multi-Agent Patterns

### Sequential Agents

```yaml
- kind: InvokeAzureAgent
  id: agent1
  agent:
    name: ResearchAgent
  output:
    messages: Local.Research

- kind: InvokeAzureAgent
  id: agent2
  conversationId: =System.ConversationId
  agent:
    name: AnalysisAgent
  input:
    messages: =Local.Research
  output:
    messages: Local.Analysis
```

### Parallel Agents

```yaml
# Dispatch to multiple agents
- kind: SetVariable
  id: init_results
  variable: Local.Results
  value: =[]

- kind: Foreach
  id: invoke_agents
  itemsProperty: =Local.AgentList
  actions:
    - kind: InvokeAzureAgent
      id: invoke_agent
      agent:
        name: =ThisItem.name
      input:
        messages: =UserMessage(Local.Task)
      output:
        messages: Local.CurrentResult
    
    - kind: SetVariable
      id: collect_result
      variable: Local.Results
      value: =Concat(Local.Results, [Local.CurrentResult])
```

### Conditional Routing

```yaml
- kind: InvokeAzureAgent
  id: classifier
  agent:
    name: ClassifierAgent
  output:
    responseObject: Local.Classification

- kind: ConditionGroup
  id: route
  conditions:
    - condition: =Local.Classification.category = "Technical"
      id: route_tech
      actions:
        - kind: InvokeAzureAgent
          id: tech_agent
          agent:
            name: TechnicalSupportAgent
    
    - condition: =Local.Classification.category = "Billing"
      id: route_billing
      actions:
        - kind: InvokeAzureAgent
          id: billing_agent
          agent:
            name: BillingAgent
```

## Best Practices

1. **Reuse Conversations** - Create conversations once, reuse across agents
2. **Handle Timeouts** - Implement timeout handling for agent invocations
3. **Validate Responses** - Check response structure before using
4. **Limit Context** - Don't send entire conversation history unnecessarily
5. **Monitor Costs** - Track agent invocations for cost management
6. **Implement Retries** - Add retry logic for transient failures
7. **Secure Credentials** - Never hardcode API keys or credentials
8. **Version Agents** - Specify agent versions for reproducibility

## Next Steps

- Learn about [Control Flow Patterns](./07-control-flow-patterns.md)
- Understand [Human-in-the-Loop and External Input](./08-human-in-the-loop.md)
- Explore [Examples and Sample Workflows](./11-examples-and-samples.md)
