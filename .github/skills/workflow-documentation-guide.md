# Declarative Workflows Documentation Guide

## Overview

This guide helps AI agents navigate the comprehensive declarative workflows documentation in the Microsoft Agent Framework.

**Documentation Root:** `docs/declarative-workflows/`

## Documentation Structure (14 Files)

### Entry Point
- **00-README.md** - Start here for overview and navigation

### By Learning Phase

#### Beginner Path
1. `00-README.md` - Overview and navigation
2. `01-overview-and-architecture.md` - Understand concepts and architecture
3. `02-yaml-schema-reference.md` - Learn YAML syntax
4. `11-examples-and-samples.md` - Study working examples

#### Intermediate Path
1. `03-action-types-reference.md` - All 25+ action types
2. `04-event-types-and-processing.md` - Event system
3. `07-control-flow-patterns.md` - Conditionals and loops
4. `08-human-in-the-loop.md` - External input patterns

#### Advanced Path
1. `05-state-management-and-checkpointing.md` - State persistence
2. `06-agent-providers-and-integration.md` - Agent integration
3. `09-expressions-and-powerfx.md` - PowerFx expressions
4. `10-workflow-execution-and-runtime.md` - Runtime operations
5. `12-best-practices-and-patterns.md` - Production guidelines

#### Distributed/Scale Path
1. `13-distributed-and-scalable-workflows.md` - Distributed execution patterns

## Quick Topic Lookup

### YAML Syntax Questions → File 02
- Workflow structure
- Variable references (System.*, Local.*)
- Expression syntax (=...)
- Data types and collections

### Action Questions → File 03
- InvokeAzureAgent
- SetVariable, ResetVariable
- ConditionGroup, Foreach
- Question, SendActivity
- Complete action catalog with properties

### Event Questions → File 04
- WorkflowStartedEvent, WorkflowOutputEvent
- DeclarativeActionInvokedEvent, DeclarativeActionCompletedEvent
- ExecutorInvokedEvent, ExecutorCompletedEvent
- **ExternalInputRequest, ExternalInputResponse** ← External request handling
- Event processing patterns

### State Management Questions → File 05
- Variable scopes and lifecycle
- Checkpoint creation/restoration
- Storage implementations
- Recovery strategies

### Agent Integration Questions → File 06
- WorkflowAgentProvider base class
- Azure AI Foundry setup
- Custom provider implementation
- Function tools

### Control Flow Questions → File 07
- If-then-else (ConditionGroup)
- Loops (Foreach, GotoAction)
- Retry patterns
- State machines

### Human-in-the-Loop Questions → File 08
- **Question action**
- **External input request/response pattern** ← External request handling
- **External loop conditions**
- Approval workflows
- Tool approval

### PowerFx Expression Questions → File 09
- Operators and functions
- String operations
- Collection functions
- Conditionals and type conversion

### Runtime Questions → File 10
- DeclarativeWorkflowBuilder usage
- DeclarativeWorkflowOptions
- Error handling
- Checkpoint resume
- Telemetry

### Example Questions → File 11
- Simple examples (Hello World)
- Agent interaction patterns
- Multi-agent pipelines
- Approval workflows

### Best Practices Questions → File 12
- Design principles
- Security guidelines
- Performance optimization
- Testing strategies

### Distributed/Scale Questions → File 13
- Horizontal scaling patterns
- Distributed checkpoint storage
- Work distribution (queues, events)
- Checkpoint locking strategies
- Auto-scaling worker pools
- Performance at scale
- Monitoring distributed workflows

## Common Agent Tasks

### "How do I handle external requests?"
→ Files: **08-human-in-the-loop.md**, **04-event-types-and-processing.md**
- Complete coverage in file 08 sections:
  - External Input Pattern (lines 39-59)
  - Processing External Input Events (lines 85-135)
  - Handling Function Call Requests (lines 109-135)
- Event documentation in file 04:
  - ExternalInputRequest section (lines 331-367)
  - ExternalInputResponse section (lines 371-388)
- Additional context in file 10 (runtime execution)
- See also: external loop patterns, approval workflows

### "How do I build a workflow?"
→ Files: 10, 02, 03
1. File 10: Building Workflows section
2. File 02: YAML syntax and structure
3. File 03: Available actions

### "How do I invoke an agent?"
→ Files: 03, 06
1. File 03: InvokeAzureAgent action
2. File 06: Agent provider setup

### "How do I implement conditionals?"
→ Files: 03, 07
1. File 03: ConditionGroup action
2. File 07: Control flow patterns

### "How do I manage state?"
→ Files: 05, 02
1. File 05: Complete state management
2. File 02: Variable syntax

### "How do I create checkpoints?"
→ File: 05
- Checkpoint creation
- Storage implementations
- Resume from checkpoint

### "What events can I listen for?"
→ File: 04
- Complete event hierarchy
- Event processing patterns
- Streaming examples

### "How do I write PowerFx expressions?"
→ Files: 09, 02
1. File 09: Complete function reference
2. File 02: Expression syntax

### "How do I scale workflows horizontally?"
→ File: 13
- Stateless worker patterns
- Distributed storage
- Work distribution
- Auto-scaling strategies

### "How do I implement distributed checkpoints?"
→ Files: 13, 05
1. File 13: Azure Blob, CosmosDB, Redis examples
2. File 05: ICheckpointManager interface

## File Descriptions

| File | Lines | Focus | Use When |
|------|-------|-------|----------|
| 00 | 154 | Navigation | Starting point |
| 01 | 407 | Architecture | Understanding system design |
| 02 | 650 | YAML Syntax | Writing workflows |
| 03 | 793 | Actions | Using specific actions |
| 04 | 757 | Events | Processing workflow events |
| 05 | 821 | State | Managing variables/checkpoints |
| 06 | 402 | Agents | Integrating AI agents |
| 07 | 391 | Control Flow | Implementing logic |
| 08 | 276 | HITL | **Handling external requests** |
| 09 | 326 | Expressions | Writing formulas |
| 10 | 396 | Runtime | Executing workflows |
| 11 | 418 | Examples | Learning from samples |
| 12 | 520 | Best Practices | Production deployment |
| 13 | 800+ | Scale/Distributed | **Building at scale** |

## External Request Handling Coverage

**Primary Documentation:**
- **File 08 (human-in-the-loop.md)** - Complete guide
  - Question action (lines 7-37)
  - External Input Pattern (lines 39-59)
  - External Loop (lines 61-83)
  - Processing External Input Events (lines 85-107)
  - Handling Function Call Requests (lines 109-135)
  - Approval Workflows (lines 137-202)
  - Interactive Troubleshooting (lines 204-242)
  - Tool Approval Pattern (lines 244-259)
  - Best Practices (lines 261-270)

**Supporting Documentation:**
- **File 04 (event-types-and-processing.md)**
  - ExternalInputRequest event (lines 331-367)
  - ExternalInputResponse event (lines 371-388)
  - Event flow examples
  
- **File 10 (workflow-execution-and-runtime.md)**
  - Resume with input (lines 340-348)
  - Event processing patterns

- **File 05 (state-management-and-checkpointing.md)**
  - Checkpoint before external input (lines 240-244)
  - Resume after input (lines 369-377)

**Coverage is THOROUGH** - Multiple perspectives (action, event, pattern, example)

## Cross-References

Most files cross-reference related topics. Follow "Next Steps" sections at the end of each file.

## Examples Location

Working code examples are embedded throughout all files (50+ total). Complete workflow samples are in `workflow-samples/` directory.

## Quick Start for Agents

```
Question: "How do I...?"
├─ YAML syntax? → File 02
├─ Use an action? → File 03
├─ Handle an event? → File 04
├─ Manage state? → File 05
├─ Integrate agents? → File 06
├─ Implement logic? → File 07
├─ Handle external input? → Files 08 + 04 ★
├─ Write expressions? → File 09
├─ Execute workflows? → File 10
├─ See examples? → File 11
├─ Follow best practices? → File 12
└─ Scale horizontally? → File 13 ★★
```

★ = External request handling thoroughly documented
★★ = NEW: Distributed and scalable patterns

## Key Concepts Map

```
Workflow Definition (YAML)
├─ Trigger → File 02
├─ Actions → File 03
│  ├─ InvokeAzureAgent → Files 03, 06
│  ├─ SetVariable → Files 03, 05
│  ├─ ConditionGroup → Files 03, 07
│  ├─ Question → Files 03, 08 ★
│  └─ SendActivity → File 03
├─ Variables → Files 02, 05
│  ├─ System.* → Files 02, 05
│  └─ Local.* → Files 02, 05
└─ Expressions → Files 02, 09

Workflow Execution
├─ Build → File 10
├─ Run → File 10
├─ Events → File 04
│  ├─ WorkflowStartedEvent → File 04
│  ├─ ExecutorInvokedEvent → File 04
│  ├─ ExternalInputRequest → Files 04, 08 ★
│  └─ WorkflowOutputEvent → File 04
├─ State → File 05
│  ├─ Variables → File 05
│  └─ Checkpoints → File 05
└─ Errors → File 10

Agent Integration
├─ Provider → File 06
├─ Azure AI → File 06
├─ Functions → File 06
└─ Conversations → File 06

★ = External request handling
```

## Last Updated

2024-12-24
