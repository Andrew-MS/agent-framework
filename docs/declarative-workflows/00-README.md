# Declarative Workflows Documentation

This comprehensive guide covers all aspects of implementing and working with declarative workflows in the Microsoft Agent Framework for .NET.

## What are Declarative Workflows?

Declarative Workflows is a no-code platform for orchestrating AI agents to accomplish complex, multi-step tasks. It allows you to design, execute, and monitor workflows using simple declarative YAML configurations—no coding required. By connecting multiple AI agents and services, it enables automation of sophisticated processes that traditionally require custom engineering.

## Documentation Structure

This documentation is organized by domain to help you quickly find the information you need:

### Core Concepts

1. **[Overview and Architecture](./01-overview-and-architecture.md)**
   - Workflow fundamentals
   - Architecture components
   - Execution model
   - Relationship to code-based workflows

2. **[YAML Schema Reference](./02-yaml-schema-reference.md)**
   - Workflow structure
   - Required and optional properties
   - Trigger definitions
   - Schema versioning

### Actions and Control Flow

3. **[Action Types Reference](./03-action-types-reference.md)**
   - Foundry Actions (InvokeAzureAgent, CreateConversation, etc.)
   - State Management Actions (SetVariable, ResetVariable, etc.)
   - Control Flow Actions (ConditionGroup, Foreach, GotoAction, etc.)
   - Human Input Actions (Question)
   - Complete action catalog with parameters

4. **[Control Flow Patterns](./07-control-flow-patterns.md)**
   - Conditional logic (ConditionGroup, ConditionItem)
   - Loops and iteration (Foreach, BreakLoop, ContinueLoop)
   - Branching and routing
   - GotoAction for non-linear flows

### Events and State

5. **[Event Types and Event Processing](./04-event-types-and-processing.md)**
   - Workflow events (WorkflowStartedEvent, WorkflowOutputEvent, etc.)
   - Declarative action events (DeclarativeActionInvokedEvent, DeclarativeActionCompletedEvent)
   - Executor events
   - External input events
   - Event streaming patterns

6. **[State Management and Checkpointing](./05-state-management-and-checkpointing.md)**
   - Variable scopes (Local, System)
   - State persistence
   - Checkpoint creation and restoration
   - Resume from checkpoint
   - CheckpointInfo structure

### Integration and Execution

7. **[Agent Providers and Integration](./06-agent-providers-and-integration.md)**
   - WorkflowAgentProvider base class
   - Azure AI Foundry agent integration
   - Custom agent provider implementation
   - Conversation management
   - Function tool integration

8. **[Human-in-the-Loop and External Input](./08-human-in-the-loop.md)**
   - Question action
   - External input requests and responses
   - ExternalInputRequest/ExternalInputResponse events
   - External loop patterns
   - Human approval workflows

9. **[Expressions and PowerFx](./09-expressions-and-powerfx.md)**
   - PowerFx expression syntax
   - Built-in functions
   - Variable references
   - Conditional expressions
   - Data transformations

10. **[Workflow Execution and Runtime](./10-workflow-execution-and-runtime.md)**
    - DeclarativeWorkflowBuilder
    - DeclarativeWorkflowOptions
    - Workflow lifecycle
    - Error handling
    - Observability and telemetry

### Practical Guides

11. **[Examples and Sample Workflows](./11-examples-and-samples.md)**
    - Simple workflows
    - Multi-agent orchestration (DeepResearch, MathChat)
    - Customer support automation
    - Marketing workflows
    - Code examples in C#

12. **[Best Practices and Patterns](./12-best-practices-and-patterns.md)**
    - Workflow design principles
    - Error handling strategies
    - Performance optimization
    - Security considerations
    - Testing strategies
    - Common pitfalls and solutions

13. **[Distributed and Scalable Workflows](./13-distributed-and-scalable-workflows.md)**
    - Stateless worker patterns
    - Distributed checkpoint storage implementations
    - Horizontal scaling strategies
    - Work distribution patterns (queues, events)
    - Checkpoint locking and concurrency
    - Performance optimization at scale
    - Monitoring and observability
    - Reference architecture for production

## Quick Start

To get started with declarative workflows:

1. Install the required package:
   ```bash
   dotnet add package Microsoft.Agents.AI.Workflows.Declarative
   ```

2. Create a simple workflow YAML file (e.g., `hello.yaml`):
   ```yaml
   kind: Workflow
   trigger:
     kind: OnConversationStart
     id: hello_workflow
     actions:
       - kind: SendActivity
         id: send_greeting
         activity: Hello, World!
   ```

3. Build and execute the workflow:
   ```csharp
   using Microsoft.Agents.AI.Workflows.Declarative;
   
   var options = new DeclarativeWorkflowOptions
   {
       // Configure your options
   };
   
   Workflow workflow = DeclarativeWorkflowBuilder.Build("hello.yaml", options);
   await workflow.RunAsync("Your input message");
   ```

## Additional Resources

- [Workflow Samples Directory](../../workflow-samples/README.md) - Sample YAML workflows
- [.NET Workflow Samples](../../dotnet/samples/GettingStarted/Workflows/) - Code-based workflow examples
- [Python Workflow Samples](../../python/samples/getting_started/workflows/) - Python workflow examples
- [Main Repository README](../../README.md) - Overview of the entire Agent Framework

## Need Help?

- **Discord**: Join our [community Discord](https://discord.gg/b5zjErwbQM)
- **Issues**: File issues on [GitHub](https://github.com/microsoft/agent-framework/issues)
- **Documentation**: Visit [MS Learn Documentation](https://learn.microsoft.com/en-us/agent-framework/)

---

Last Updated: December 2024
