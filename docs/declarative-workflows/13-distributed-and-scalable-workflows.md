# Distributed and Scalable Declarative Workflows

## Overview

This guide provides patterns and strategies for implementing declarative workflows at scale in distributed environments. It complements the core documentation with distributed systems considerations.

## Architecture Patterns

### Stateless Worker Pattern

Declarative workflows are designed to support stateless execution:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Worker 1   │     │  Worker 2   │     │  Worker 3   │
│  (Stateless)│     │  (Stateless)│     │  (Stateless)│
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                  ┌────────▼────────┐
                  │ Shared Checkpoint│
                  │     Storage      │
                  │ (Azure Blob/     │
                  │  CosmosDB/S3)    │
                  └──────────────────┘
```

**Key Principles:**
- Workers hold no state between requests
- All state persisted in checkpoint storage
- Any worker can resume any workflow
- Enables horizontal scaling

### Workflow Execution Models

#### Model 1: Pull-Based Execution

Workers poll a queue for workflow requests:

```csharp
// Worker process
while (true)
{
    var workItem = await queue.DequeueAsync();
    
    var options = new DeclarativeWorkflowOptions
    {
        CheckpointManager = new AzureBlobCheckpointStore(blobClient),
        AgentProvider = agentProvider
    };
    
    var workflow = DeclarativeWorkflowBuilder.Build(workItem.WorkflowFile, options);
    
    // Check for existing checkpoint
    var checkpoint = await checkpointManager.GetCheckpointAsync(
        workItem.RunId, 
        workItem.CheckpointId
    );
    
    if (checkpoint != null)
    {
        // Resume from checkpoint
        workflow = workflow.ResumeFromCheckpoint(checkpoint);
        await foreach (var evt in workflow.ResumeAsync(workItem.Input))
        {
            await ProcessEvent(evt, workItem);
        }
    }
    else
    {
        // Start new workflow
        await foreach (var evt in workflow.RunAsync(workItem.Input))
        {
            await ProcessEvent(evt, workItem);
        }
    }
}
```

#### Model 2: Event-Driven Execution

Workflows triggered by events from a message bus:

```csharp
// Subscribe to workflow start events
messageBus.Subscribe<WorkflowStartEvent>(async (evt) =>
{
    var workflow = await BuildWorkflow(evt.WorkflowId);
    await ExecuteWorkflow(workflow, evt.Input);
});

// Subscribe to workflow resume events
messageBus.Subscribe<WorkflowResumeEvent>(async (evt) =>
{
    var checkpoint = await checkpointManager.GetCheckpointAsync(
        evt.RunId, 
        evt.CheckpointId
    );
    
    if (checkpoint != null)
    {
        var workflow = await BuildWorkflow(evt.WorkflowId);
        var resumed = workflow.ResumeFromCheckpoint(checkpoint);
        await ExecuteWorkflow(resumed, evt.Input);
    }
});
```

## Checkpoint Storage for Scale

### Distributed Checkpoint Storage

#### Azure Blob Storage Implementation

```csharp
public class AzureBlobCheckpointStore : ICheckpointManager
{
    private readonly BlobContainerClient _containerClient;
    
    public AzureBlobCheckpointStore(BlobContainerClient containerClient)
    {
        _containerClient = containerClient;
    }
    
    public async Task SaveCheckpointAsync(
        Checkpoint checkpoint, 
        CancellationToken cancellationToken = default)
    {
        var blobName = $"{checkpoint.RunId}/{checkpoint.CheckpointId}.json";
        var blobClient = _containerClient.GetBlobClient(blobName);
        
        var json = JsonSerializer.Serialize(checkpoint);
        var content = BinaryData.FromString(json);
        
        // Use conditional upload for optimistic concurrency
        var options = new BlobUploadOptions
        {
            Conditions = new BlobRequestConditions
            {
                IfNoneMatch = ETag.All // Prevent overwrite
            }
        };
        
        try
        {
            await blobClient.UploadAsync(content, options, cancellationToken);
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            // Checkpoint already exists - this is expected in some scenarios
            // Log and continue or throw based on requirements
        }
    }
    
    public async Task<Checkpoint?> GetCheckpointAsync(
        string runId, 
        string checkpointId, 
        CancellationToken cancellationToken = default)
    {
        var blobName = $"{runId}/{checkpointId}.json";
        var blobClient = _containerClient.GetBlobClient(blobName);
        
        if (!await blobClient.ExistsAsync(cancellationToken))
        {
            return null;
        }
        
        var response = await blobClient.DownloadContentAsync(cancellationToken);
        var json = response.Value.Content.ToString();
        
        return JsonSerializer.Deserialize<Checkpoint>(json);
    }
    
    public async Task<IEnumerable<CheckpointInfo>> ListCheckpointsAsync(
        string runId, 
        CancellationToken cancellationToken = default)
    {
        var checkpoints = new List<CheckpointInfo>();
        var prefix = $"{runId}/";
        
        await foreach (var blob in _containerClient.GetBlobsAsync(
            prefix: prefix, 
            cancellationToken: cancellationToken))
        {
            var checkpointId = Path.GetFileNameWithoutExtension(blob.Name);
            checkpoints.Add(new CheckpointInfo(runId, checkpointId));
        }
        
        return checkpoints;
    }
}
```

#### CosmosDB Implementation Pattern

```csharp
public class CosmosDbCheckpointStore : ICheckpointManager
{
    private readonly Container _container;
    
    public async Task SaveCheckpointAsync(
        Checkpoint checkpoint, 
        CancellationToken cancellationToken = default)
    {
        var document = new CheckpointDocument
        {
            Id = $"{checkpoint.RunId}_{checkpoint.CheckpointId}",
            PartitionKey = checkpoint.RunId,
            Checkpoint = checkpoint,
            Timestamp = DateTime.UtcNow
        };
        
        // Use optimistic concurrency with ETag
        await _container.UpsertItemAsync(
            document, 
            new PartitionKey(document.PartitionKey),
            cancellationToken: cancellationToken
        );
    }
    
    // Additional methods...
}
```

### Checkpoint Locking Strategies

#### Optimistic Concurrency

Use ETags or version numbers to detect concurrent modifications:

```csharp
public class OptimisticLockCheckpointStore : ICheckpointManager
{
    public async Task SaveCheckpointAsync(Checkpoint checkpoint, ...)
    {
        int maxRetries = 3;
        for (int attempt = 0; attempt < maxRetries; attempt++)
        {
            try
            {
                // Read current version
                var existing = await GetCheckpointAsync(
                    checkpoint.RunId, 
                    checkpoint.CheckpointId
                );
                
                if (existing != null && existing.Version != checkpoint.Version)
                {
                    throw new ConcurrencyException("Checkpoint was modified");
                }
                
                // Increment version
                checkpoint.Version++;
                
                // Save with version check
                await SaveWithVersionCheckAsync(checkpoint);
                return;
            }
            catch (ConcurrencyException) when (attempt < maxRetries - 1)
            {
                // Retry with exponential backoff
                await Task.Delay(TimeSpan.FromMilliseconds(100 * Math.Pow(2, attempt)));
            }
        }
        
        throw new ConcurrencyException("Failed to save checkpoint after retries");
    }
}
```

#### Pessimistic Locking with Redis

```csharp
public class RedisLockCheckpointStore : ICheckpointManager
{
    private readonly IDatabase _redis;
    private readonly ICheckpointManager _underlyingStore;
    
    public async Task SaveCheckpointAsync(Checkpoint checkpoint, ...)
    {
        var lockKey = $"checkpoint:lock:{checkpoint.RunId}";
        var lockValue = Guid.NewGuid().ToString();
        
        // Acquire distributed lock
        bool acquired = await _redis.StringSetAsync(
            lockKey, 
            lockValue, 
            TimeSpan.FromSeconds(30), 
            When.NotExists
        );
        
        if (!acquired)
        {
            throw new LockException("Could not acquire checkpoint lock");
        }
        
        try
        {
            // Perform checkpoint save
            await _underlyingStore.SaveCheckpointAsync(checkpoint, cancellationToken);
        }
        finally
        {
            // Release lock (check value to prevent releasing someone else's lock)
            var script = @"
                if redis.call('get', KEYS[1]) == ARGV[1] then
                    return redis.call('del', KEYS[1])
                else
                    return 0
                end";
            
            await _redis.ScriptEvaluateAsync(script, new[] { (RedisKey)lockKey }, new[] { (RedisValue)lockValue });
        }
    }
}
```

## Work Distribution Patterns

### Queue-Based Distribution

#### Azure Service Bus Pattern

```csharp
public class WorkflowOrchestrator
{
    private readonly ServiceBusClient _serviceBusClient;
    private readonly ICheckpointManager _checkpointManager;
    
    public async Task StartWorkflowAsync(string workflowFile, object input)
    {
        var runId = Guid.NewGuid().ToString("N");
        
        var message = new ServiceBusMessage(JsonSerializer.Serialize(new
        {
            RunId = runId,
            WorkflowFile = workflowFile,
            Input = input,
            Action = "Start"
        }))
        {
            MessageId = runId,
            PartitionKey = runId // Ensure ordered processing
        };
        
        var sender = _serviceBusClient.CreateSender("workflow-queue");
        await sender.SendMessageAsync(message);
    }
    
    public async Task ResumeWorkflowAsync(
        string runId, 
        string checkpointId, 
        ExternalInputResponse input)
    {
        var message = new ServiceBusMessage(JsonSerializer.Serialize(new
        {
            RunId = runId,
            CheckpointId = checkpointId,
            Input = input,
            Action = "Resume"
        }))
        {
            MessageId = $"{runId}_{checkpointId}",
            PartitionKey = runId
        };
        
        var sender = _serviceBusClient.CreateSender("workflow-queue");
        await sender.SendMessageAsync(message);
    }
}

// Worker that processes queue messages
public class WorkflowWorker
{
    public async Task ProcessMessagesAsync(CancellationToken cancellationToken)
    {
        var receiver = _serviceBusClient.CreateReceiver("workflow-queue");
        
        while (!cancellationToken.IsCancellationRequested)
        {
            var messages = await receiver.ReceiveMessagesAsync(
                maxMessages: 1,
                maxWaitTime: TimeSpan.FromSeconds(5),
                cancellationToken: cancellationToken
            );
            
            foreach (var message in messages)
            {
                try
                {
                    await ProcessWorkflowMessageAsync(message);
                    await receiver.CompleteMessageAsync(message, cancellationToken);
                }
                catch (Exception ex)
                {
                    // Log error and abandon message for retry
                    await receiver.AbandonMessageAsync(message, cancellationToken);
                }
            }
        }
    }
}
```

### Event Stream Distribution

#### Kafka Pattern

```csharp
public class KafkaWorkflowProducer
{
    private readonly IProducer<string, WorkflowMessage> _producer;
    
    public async Task PublishWorkflowEventAsync(WorkflowMessage message)
    {
        await _producer.ProduceAsync(
            "workflow-events",
            new Message<string, WorkflowMessage>
            {
                Key = message.RunId, // Partition by RunId
                Value = message
            }
        );
    }
}

public class KafkaWorkflowConsumer
{
    private readonly IConsumer<string, WorkflowMessage> _consumer;
    
    public async Task ConsumeWorkflowEventsAsync(CancellationToken cancellationToken)
    {
        _consumer.Subscribe("workflow-events");
        
        while (!cancellationToken.IsCancellationRequested)
        {
            var consumeResult = _consumer.Consume(cancellationToken);
            
            await ProcessWorkflowMessageAsync(consumeResult.Message.Value);
            
            _consumer.Commit(consumeResult);
        }
    }
}
```

## Horizontal Scaling Patterns

### Auto-Scaling Worker Pools

```
┌──────────────────────────────────────────────────────────┐
│                    Load Balancer                          │
└────────────────────┬─────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
   ┌────▼─────┐ ┌───▼──────┐ ┌──▼───────┐
   │ Worker   │ │ Worker   │ │ Worker   │
   │ Pod 1    │ │ Pod 2    │ │ Pod N    │
   └────┬─────┘ └───┬──────┘ └──┬───────┘
        │           │            │
        └───────────┼────────────┘
                    │
         ┌──────────▼───────────┐
         │  Workflow Queue      │
         │  (Service Bus/SQS)   │
         └──────────────────────┘
```

**Scaling Metrics:**
- Queue depth (messages waiting)
- CPU utilization per worker
- Checkpoint storage latency
- Agent invocation rate

**Kubernetes HPA Example:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: workflow-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: workflow-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: servicebus_queue_length
      target:
        type: AverageValue
        averageValue: "10" # Scale up if >10 messages per pod
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Partitioning Strategies

#### By Workflow Type

```csharp
public class PartitionedWorkflowRouter
{
    public string DetermineQueue(string workflowFile)
    {
        // Route different workflow types to different queues
        return workflowFile.Contains("Customer") 
            ? "workflow-queue-customer"
            : workflowFile.Contains("Order")
            ? "workflow-queue-order"
            : "workflow-queue-default";
    }
}
```

#### By Priority

```csharp
public class PriorityWorkflowRouter
{
    public string DetermineQueue(WorkflowRequest request)
    {
        return request.Priority switch
        {
            Priority.High => "workflow-queue-high-priority",
            Priority.Normal => "workflow-queue-normal-priority",
            Priority.Low => "workflow-queue-low-priority",
            _ => "workflow-queue-default"
        };
    }
}
```

## Event Distribution for Scale

### Publishing Workflow Events to External Systems

```csharp
public class DistributedEventProcessor
{
    private readonly IEventBus _eventBus;
    
    public async Task ProcessWorkflowEventsAsync(Workflow workflow, object input)
    {
        await foreach (var evt in workflow.RunAsync(input))
        {
            // Publish events to distributed event bus
            if (evt is WorkflowOutputEvent output)
            {
                await _eventBus.PublishAsync(new
                {
                    EventType = "WorkflowOutput",
                    RunId = workflow.RunId,
                    Output = output.Data,
                    Timestamp = DateTime.UtcNow
                });
            }
            
            if (evt is ExternalInputRequest request)
            {
                await _eventBus.PublishAsync(new
                {
                    EventType = "ExternalInputRequired",
                    RunId = workflow.RunId,
                    Request = request,
                    Timestamp = DateTime.UtcNow
                });
            }
            
            // Local processing
            await ProcessEventLocally(evt);
        }
    }
}
```

### Event Sourcing Pattern

```csharp
public class EventSourcedWorkflow
{
    private readonly IEventStore _eventStore;
    
    public async Task ExecuteWithEventSourcingAsync(Workflow workflow, object input)
    {
        var runId = Guid.NewGuid().ToString();
        
        await foreach (var evt in workflow.RunAsync(input))
        {
            // Store every event for replay/audit
            await _eventStore.AppendEventAsync(new StoredEvent
            {
                RunId = runId,
                EventType = evt.GetType().Name,
                EventData = JsonSerializer.Serialize(evt),
                Timestamp = DateTime.UtcNow,
                SequenceNumber = await _eventStore.GetNextSequenceAsync(runId)
            });
            
            // Process event
            await ProcessEventAsync(evt);
        }
    }
    
    public async Task ReplayEventsAsync(string runId)
    {
        var events = await _eventStore.GetEventsAsync(runId);
        
        foreach (var storedEvent in events)
        {
            // Reconstruct workflow state from events
            // Useful for debugging or analytics
        }
    }
}
```

## Performance Optimization

### Checkpoint Size Optimization

```csharp
public class OptimizedCheckpoint : Checkpoint
{
    // Store only essential state
    public Dictionary<string, object> CompressedVariables { get; set; }
    
    // Reference large data instead of embedding
    public string ConversationHistoryBlobUrl { get; set; }
    
    public void Compress()
    {
        // Compress large variables
        foreach (var kvp in this.Variables.Where(v => IsLarge(v.Value)))
        {
            var compressed = Compress(kvp.Value);
            this.CompressedVariables[kvp.Key] = compressed;
            this.Variables.Remove(kvp.Key);
        }
    }
}
```

### Parallel Agent Invocation

```csharp
// When multiple independent agent calls are needed
public async Task ExecuteParallelAgentsAsync()
{
    var tasks = new[]
    {
        InvokeAgentAsync("ResearchAgent", input1),
        InvokeAgentAsync("AnalysisAgent", input2),
        InvokeAgentAsync("SummaryAgent", input3)
    };
    
    var results = await Task.WhenAll(tasks);
    
    // Combine results
}
```

### Caching Strategies

```csharp
public class CachedAgentProvider : WorkflowAgentProvider
{
    private readonly IDistributedCache _cache;
    private readonly WorkflowAgentProvider _underlying;
    
    public override async IAsyncEnumerable<AgentRunResponseUpdate> InvokeAgentAsync(
        string agentId, ...)
    {
        var cacheKey = $"agent:{agentId}:{ComputeHash(messages)}";
        
        // Check cache
        var cached = await _cache.GetAsync(cacheKey);
        if (cached != null)
        {
            yield return DeserializeCachedResponse(cached);
            yield break;
        }
        
        // Invoke and cache
        var responses = new List<AgentRunResponseUpdate>();
        await foreach (var update in _underlying.InvokeAgentAsync(agentId, ...))
        {
            responses.Add(update);
            yield return update;
        }
        
        await _cache.SetAsync(cacheKey, SerializeResponses(responses), 
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1) });
    }
}
```

## Monitoring and Observability at Scale

### Distributed Tracing

```csharp
public class TracedWorkflowExecutor
{
    private readonly ActivitySource _activitySource;
    
    public async Task ExecuteAsync(Workflow workflow, object input)
    {
        using var activity = _activitySource.StartActivity("ExecuteWorkflow");
        activity?.SetTag("workflow.id", workflow.Id);
        activity?.SetTag("workflow.runId", workflow.RunId);
        
        await foreach (var evt in workflow.RunAsync(input))
        {
            using var eventActivity = _activitySource.StartActivity(
                $"ProcessEvent.{evt.GetType().Name}",
                ActivityKind.Internal,
                activity.Context
            );
            
            eventActivity?.SetTag("event.type", evt.GetType().Name);
            
            await ProcessEventAsync(evt);
        }
    }
}
```

### Metrics Collection

```csharp
public class MetricsCollector
{
    private readonly IMeterFactory _meterFactory;
    private readonly Counter<long> _workflowsStarted;
    private readonly Counter<long> _workflowsCompleted;
    private readonly Histogram<double> _workflowDuration;
    private readonly Counter<long> _checkpointsCreated;
    
    public MetricsCollector(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("DeclarativeWorkflows");
        
        _workflowsStarted = meter.CreateCounter<long>("workflows.started");
        _workflowsCompleted = meter.CreateCounter<long>("workflows.completed");
        _workflowDuration = meter.CreateHistogram<double>("workflow.duration", "ms");
        _checkpointsCreated = meter.CreateCounter<long>("checkpoints.created");
    }
    
    public void RecordWorkflowMetrics(Workflow workflow, TimeSpan duration, bool completed)
    {
        _workflowsStarted.Add(1, new TagList { { "workflow.type", workflow.Type } });
        
        if (completed)
        {
            _workflowsCompleted.Add(1, new TagList { { "workflow.type", workflow.Type } });
            _workflowDuration.Record(duration.TotalMilliseconds, 
                new TagList { { "workflow.type", workflow.Type } });
        }
    }
}
```

## Failure Recovery Patterns

### Automatic Retry with Exponential Backoff

```csharp
public class ResilientWorkflowExecutor
{
    public async Task<bool> ExecuteWithRetryAsync(
        Workflow workflow, 
        object input,
        int maxRetries = 3)
    {
        for (int attempt = 0; attempt < maxRetries; attempt++)
        {
            try
            {
                await foreach (var evt in workflow.RunAsync(input))
                {
                    await ProcessEventAsync(evt);
                }
                return true;
            }
            catch (Exception ex) when (IsTransient(ex) && attempt < maxRetries - 1)
            {
                var delay = TimeSpan.FromSeconds(Math.Pow(2, attempt));
                await Task.Delay(delay);
            }
        }
        
        return false;
    }
    
    private bool IsTransient(Exception ex)
    {
        return ex is TimeoutException 
            || ex is HttpRequestException
            || ex is TaskCanceledException;
    }
}
```

### Circuit Breaker for Agent Calls

```csharp
public class CircuitBreakerAgentProvider : WorkflowAgentProvider
{
    private readonly CircuitBreakerPolicy _circuitBreaker;
    
    public CircuitBreakerAgentProvider()
    {
        _circuitBreaker = Policy
            .Handle<Exception>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromMinutes(1)
            );
    }
    
    public override async IAsyncEnumerable<AgentRunResponseUpdate> InvokeAgentAsync(
        string agentId, ...)
    {
        await foreach (var update in _circuitBreaker.ExecuteAsync(async () =>
        {
            return await base.InvokeAgentAsync(agentId, ...);
        }))
        {
            yield return update;
        }
    }
}
```

## Best Practices for Distributed Workflows

### Do's:
1. ✅ Use stateless workers with external checkpoint storage
2. ✅ Implement idempotent checkpoint saves
3. ✅ Partition workflows by type or priority
4. ✅ Use distributed locks for critical sections
5. ✅ Implement circuit breakers for external dependencies
6. ✅ Monitor queue depth and latency
7. ✅ Use distributed tracing for end-to-end visibility

### Don'ts:
1. ❌ Store state in worker memory
2. ❌ Use local file storage for checkpoints in distributed environments
3. ❌ Ignore checkpoint save failures
4. ❌ Create unbounded checkpoint sizes
5. ❌ Block workers on synchronous operations
6. ❌ Skip error handling in distributed scenarios

## Reference Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        API Gateway                           │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
   ┌────▼─────┐ ┌───▼──────┐ ┌──▼───────┐
   │Workflow  │ │Workflow  │ │Workflow  │
   │Worker 1  │ │Worker 2  │ │Worker N  │
   └────┬─────┘ └───┬──────┘ └──┬───────┘
        │           │            │
        └───────────┼────────────┘
                    │
      ┌─────────────┼─────────────┐
      │             │             │
  ┌───▼──────┐ ┌───▼──────┐ ┌───▼──────┐
  │Azure     │ │Service   │ │Azure AI  │
  │Blob      │ │Bus       │ │Foundry   │
  │(Checkpts)│ │(Queue)   │ │(Agents)  │
  └──────────┘ └──────────┘ └──────────┘
      │             │             │
      └─────────────┼─────────────┘
                    │
            ┌───────▼────────┐
            │ Application    │
            │ Insights       │
            │ (Monitoring)   │
            └────────────────┘
```

## Summary

This guide provides patterns for implementing declarative workflows at scale:

- **Stateless Workers**: Enable horizontal scaling
- **Distributed Storage**: Support durable checkpoints
- **Work Distribution**: Queue-based and event-driven patterns
- **Locking Strategies**: Optimistic and pessimistic concurrency
- **Monitoring**: Distributed tracing and metrics
- **Resilience**: Retry, circuit breaker, and recovery patterns

Combined with the core documentation, you have everything needed to build a production-scale distributed workflow service.

## See Also

- [State Management and Checkpointing](./05-state-management-and-checkpointing.md) - Core checkpoint concepts
- [Workflow Execution and Runtime](./10-workflow-execution-and-runtime.md) - Execution patterns
- [Best Practices and Patterns](./12-best-practices-and-patterns.md) - General best practices
