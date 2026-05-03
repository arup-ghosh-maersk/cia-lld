# Event Handlers — Implementation Guide

> **Date:** May 3, 2026 | **Version:** 1.0 | **Status:** Production Ready (Phase 2)

## Table of Contents
1. [Overview](#overview)
2. [Setup & Configuration](#setup--configuration)
3. [AssetEventHandler Implementation](#asseteventhandler-implementation)
4. [NotificationHandler Implementation](#notificationhandler-implementation)
5. [DocumentHandler Implementation](#documenthandler-implementation)
6. [Error Handling & Retries](#error-handling--retries)
7. [Testing](#testing)
8. [Deployment](#deployment)

---

## Overview

**Event Handlers** are async consumers that react to events and perform side effects:
- **AssetEventHandler** — Processes asset changes, creates audit records
- **NotificationHandler** — Sends alerts to UI, email, webhooks
- **DocumentHandler** — Manages CDR scanning, file cleanup

### Architecture

```
ASSET_EVENT_STORE
        ↓ (Published)
        ├→ AssetEventHandler (Audit)
        ├→ NotificationHandler (Alert users)
        └→ DocumentHandler (Scan files)
```

### Key Principles
- ✅ **Async:** Non-blocking event processing
- ✅ **Idempotent:** Safe to process same event twice
- ✅ **Resilient:** Retry on failure, dead-letter on permanent failure
- ✅ **Ordered:** Events processed per-asset in sequence
- ✅ **Traceable:** Correlation IDs track requests

---

## Setup & Configuration

### Dependencies
```xml
<PackageReference Include="MassTransit" Version="8.0.0" />
<PackageReference Include="MassTransit.Azure.ServiceBus.Core" Version="8.0.0" />
<PackageReference Include="Serilog" Version="3.0.0" />
<PackageReference Include="Polly" Version="8.0.0" />
```

### Dependency Injection (Startup)

```csharp
public static class EventHandlerConfiguration
{
    public static IServiceCollection AddEventHandlers(
        this IServiceCollection services,
        IConfiguration config)
    {
        services.AddMassTransit(x =>
        {
            // Register consumer
            x.AddConsumer<AssetEventHandler>();
            x.AddConsumer<NotificationHandler>();
            x.AddConsumer<DocumentHandler>();
            
            // Configure Azure Service Bus transport
            x.UsingAzureServiceBus((context, cfg) =>
            {
                cfg.Host(config["AzureServiceBus:ConnectionString"]);
                
                // Configure endpoint for AssetEventHandler
                cfg.ReceiveEndpoint("asset-event-queue", e =>
                {
                    e.ConfigureConsumer<AssetEventHandler>(context);
                    e.ConcurrentMessageLimit = 10;
                    e.PrefetchCount = 50;
                    
                    // Retry policy: exponential backoff
                    e.UseMessageRetry(r =>
                    {
                        r.Exponential(
                            retryCount: 5,
                            initialInterval: TimeSpan.FromSeconds(1),
                            intervalIncrement: TimeSpan.FromSeconds(1),
                            intervalIncrementFactor: 2.0);
                    });
                });
                
                cfg.ReceiveEndpoint("notification-queue", e =>
                {
                    e.ConfigureConsumer<NotificationHandler>(context);
                });
                
                cfg.ReceiveEndpoint("document-scan-queue", e =>
                {
                    e.ConfigureConsumer<DocumentHandler>(context);
                });
            });
        });
        
        services.AddScoped<IAssetAuditService, AssetAuditService>();
        services.AddScoped<INotificationService, NotificationService>();
        services.AddScoped<IDocumentScanService, DocumentScanService>();
        
        return services;
    }
}

// In Program.cs
builder.Services.AddEventHandlers(builder.Configuration);
```

---

## AssetEventHandler Implementation

### Purpose
Creates audit records from asset events.

### Implementation

```csharp
public class AssetEventHandler : IConsumer<IAssetEvent>
{
    private readonly IAssetAuditService _auditService;
    private readonly IRepository _repo;
    private readonly ILogger<AssetEventHandler> _logger;
    
    public AssetEventHandler(
        IAssetAuditService auditService,
        IRepository repo,
        ILogger<AssetEventHandler> logger)
    {
        _auditService = auditService;
        _repo = repo;
        _logger = logger;
    }
    
    public async Task Consume(ConsumeContext<IAssetEvent> context)
    {
        var @event = context.Message;
        
        _logger.LogInformation(
            "Processing {EventType} for asset {AssetId}",
            @event.GetType().Name, @event.AssetMasterId);
        
        try
        {
            // Create audit record based on event type
            var audit = await CreateAuditRecordAsync(@event);
            
            if (audit != null)
            {
                await _repo.CreateAuditAsync(audit);
                _logger.LogInformation(
                    "Audit record created for {@Event}", @event);
            }
            
            // Publish derived events (e.g., AuditCreated)
            await context.Publish(
                new AuditRecordCreated { AuditId = audit.Id });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Error processing event {EventType} for asset {AssetId}",
                @event.GetType().Name, @event.AssetMasterId);
            throw; // Rethrow to trigger retry
        }
    }
    
    private async Task<AssetAudit> CreateAuditRecordAsync(IAssetEvent @event)
    {
        return @event switch
        {
            AssetCreated e => new AssetAudit
            {
                Id = Guid.NewGuid(),
                AssetMasterId = e.AssetMasterId,
                ChangeType = "CREATED",
                EntityType = "ASSET",
                FieldName = "asset_master",
                NewValue = $"Asset: {e.Name} (Code: {@e.AssetCode})",
                ChangedBy = e.EventUserId,
                ChangedByName = e.EventUserName,
                ChangedAt = e.EventTimestamp,
                CorrelationId = e.EventCorrelationId,
                IpAddress = e.EventMetadata?["source_ip"]?.ToString()
            },
            
            AssetAttributeChanged e => new AssetAudit
            {
                Id = Guid.NewGuid(),
                AssetMasterId = e.AssetMasterId,
                ChangeType = "UPDATED",
                EntityType = "ATTRIBUTE",
                FieldName = e.AttributeKey,
                OldValue = e.OldValue,
                NewValue = e.NewValue,
                ChangedBy = e.EventUserId,
                ChangedByName = e.EventUserName,
                ChangedAt = e.EventTimestamp,
                Notes = e.ChangeReason
            },
            
            AssetStatusChanged e => new AssetAudit
            {
                Id = Guid.NewGuid(),
                AssetMasterId = e.AssetMasterId,
                ChangeType = "STATUS_CHANGE",
                EntityType = "ASSET",
                FieldName = "status",
                OldValue = e.OldStatus,
                NewValue = e.NewStatus,
                ChangedBy = e.EventUserId,
                ChangedByName = e.EventUserName,
                ChangedAt = e.EventTimestamp,
                Notes = e.StatusReason
            },
            
            _ => null
        };
    }
}

// Event interfaces
public interface IAssetEvent
{
    Guid AssetMasterId { get; }
    Guid EventUserId { get; }
    string EventUserName { get; }
    DateTime EventTimestamp { get; }
    string EventCorrelationId { get; }
    Dictionary<string, object> EventMetadata { get; }
}

public class AssetCreated : IAssetEvent
{
    public Guid AssetMasterId { get; set; }
    public string AssetCode { get; set; }
    public string Name { get; set; }
    public Guid EventUserId { get; set; }
    public string EventUserName { get; set; }
    public DateTime EventTimestamp { get; set; }
    public string EventCorrelationId { get; set; }
    public Dictionary<string, object> EventMetadata { get; set; }
}

public class AssetAttributeChanged : IAssetEvent
{
    public Guid AssetMasterId { get; set; }
    public string AttributeKey { get; set; }
    public string OldValue { get; set; }
    public string NewValue { get; set; }
    public string ChangeReason { get; set; }
    public Guid EventUserId { get; set; }
    public string EventUserName { get; set; }
    public DateTime EventTimestamp { get; set; }
    public string EventCorrelationId { get; set; }
    public Dictionary<string, object> EventMetadata { get; set; }
}

public class AssetStatusChanged : IAssetEvent
{
    public Guid AssetMasterId { get; set; }
    public string OldStatus { get; set; }
    public string NewStatus { get; set; }
    public string StatusReason { get; set; }
    public Guid EventUserId { get; set; }
    public string EventUserName { get; set; }
    public DateTime EventTimestamp { get; set; }
    public string EventCorrelationId { get; set; }
    public Dictionary<string, object> EventMetadata { get; set; }
}
```

---

## NotificationHandler Implementation

### Purpose
Sends alerts via UI, email, and webhooks.

### Implementation

```csharp
public class NotificationHandler : IConsumer<IAssetEvent>
{
    private readonly INotificationService _notificationService;
    private readonly ILogger<NotificationHandler> _logger;
    
    public async Task Consume(ConsumeContext<IAssetEvent> context)
    {
        var @event = context.Message;
        
        _logger.LogInformation(
            "Sending notifications for {@EventType}",
            @event.GetType().Name);
        
        try
        {
            // Determine notification type
            var notifications = GetNotificationsForEvent(@event);
            
            // Send via all channels
            foreach (var notification in notifications)
            {
                await _notificationService.SendAsync(notification);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending notifications");
            throw;
        }
    }
    
    private List<Notification> GetNotificationsForEvent(IAssetEvent @event)
    {
        var notifications = new List<Notification>();
        
        var baseNotification = new Notification
        {
            Id = Guid.NewGuid(),
            CorrelationId = @event.EventCorrelationId,
            CreatedAt = @event.EventTimestamp,
            IsRead = false
        };
        
        // Add UI notification
        notifications.Add(new UiNotification
        {
            Title = GetTitle(@event),
            Message = GetMessage(@event),
            NotificationType = GetNotificationType(@event),
            TargetAssetId = @event.AssetMasterId
        });
        
        // Add email if critical event
        if (ShouldSendEmail(@event))
        {
            notifications.Add(new EmailNotification
            {
                ToAddress = GetRecipientEmail(@event),
                Subject = GetEmailSubject(@event),
                BodyHtml = GetEmailBodyHtml(@event)
            });
        }
        
        // Add webhook if configured
        if (ShouldSendWebhook(@event))
        {
            notifications.Add(new WebhookNotification
            {
                WebhookUrl = GetWebhookUrl(@event),
                Payload = JsonSerializer.Serialize(@event)
            });
        }
        
        return notifications;
    }
    
    private string GetTitle(IAssetEvent @event) => @event switch
    {
        AssetCreated e => $"Asset Created: {e.Name}",
        AssetStatusChanged e => $"Asset Status Changed: {e.NewStatus}",
        AssetAttributeChanged e => $"Attribute Updated: {e.AttributeKey}",
        _ => "Asset Event"
    };
    
    private string GetMessage(IAssetEvent @event) => @event switch
    {
        AssetStatusChanged e => 
            $"Status changed from {e.OldStatus} to {e.NewStatus}. Reason: {e.StatusReason}",
        AssetAttributeChanged e => 
            $"Changed from '{e.OldValue}' to '{e.NewValue}'",
        _ => "An asset event occurred"
    };
    
    private bool ShouldSendEmail(IAssetEvent @event) => @event switch
    {
        AssetStatusChanged e => e.NewStatus == "INACTIVE",
        _ => false
    };
}

public class NotificationService : INotificationService
{
    private readonly IUiNotificationService _uiService;
    private readonly IEmailService _emailService;
    private readonly IWebhookService _webhookService;
    private readonly IRepository _repo;
    private readonly ILogger<NotificationService> _logger;
    
    public async Task SendAsync(Notification notification)
    {
        var notifLog = new NotificationLog
        {
            Id = Guid.NewGuid(),
            NotificationType = notification.GetType().Name,
            Channel = GetChannel(notification),
            Status = "PENDING",
            CreatedAt = DateTime.UtcNow
        };
        
        try
        {
            await ExecuteNotificationAsync(notification);
            
            notifLog.Status = "SENT";
            notifLog.SentAt = DateTime.UtcNow;
        }
        catch (Exception ex)
        {
            notifLog.Status = "FAILED";
            notifLog.ErrorMessage = ex.Message;
            
            _logger.LogError(ex, "Failed to send notification");
            throw;
        }
        finally
        {
            await _repo.CreateNotificationLogAsync(notifLog);
        }
    }
    
    private async Task ExecuteNotificationAsync(Notification notification)
    {
        switch (notification)
        {
            case UiNotification ui:
                await _uiService.SendAsync(ui);
                break;
            case EmailNotification email:
                await _emailService.SendAsync(email);
                break;
            case WebhookNotification webhook:
                await _webhookService.SendAsync(webhook);
                break;
        }
    }
}
```

---

## DocumentHandler Implementation

### Purpose
Manages file scanning with Votiro CDR.

### Implementation

```csharp
public class DocumentHandler : IConsumer<DocumentUploadedEvent>
{
    private readonly IDocumentScanService _scanService;
    private readonly IVotiroService _votiroService;
    private readonly IRepository _repo;
    private readonly ILogger<DocumentHandler> _logger;
    
    public async Task Consume(ConsumeContext<DocumentUploadedEvent> context)
    {
        var @event = context.Message;
        
        _logger.LogInformation(
            "Processing document scan for attachment {@AttachmentId}",
            @event.AttachmentId);
        
        try
        {
            var attachment = await _repo.GetAttachmentAsync(@event.AttachmentId);
            if (attachment == null)
            {
                _logger.LogWarning("Attachment not found: {@AttachmentId}",
                    @event.AttachmentId);
                return;
            }
            
            // Create scan record
            var scan = new DocumentScan
            {
                Id = Guid.NewGuid(),
                AttachmentId = attachment.Id,
                ScanStatus = "PENDING",
                ScanStartedAt = DateTime.UtcNow,
                CreatedAt = DateTime.UtcNow
            };
            
            await _repo.CreateScanAsync(scan);
            
            // Upload file from blob storage to Votiro
            var fileStream = await _repo.GetAttachmentFileAsync(attachment.Id);
            var votiroJobId = await _votiroService.SendForScanAsync(
                fileStream, attachment.FileName);
            
            scan.VotiroJobId = votiroJobId;
            scan.ScanStatus = "SCANNING";
            await _repo.UpdateScanAsync(scan);
            
            _logger.LogInformation(
                "File sent to Votiro: JobId={JobId}", votiroJobId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error scanning document");
            throw;
        }
    }
}

// Webhook endpoint for Votiro callbacks
[ApiController]
[Route("api/v1/webhooks")]
public class WebhookController : ControllerBase
{
    private readonly IDocumentScanService _scanService;
    private readonly IRepository _repo;
    private readonly ILogger<WebhookController> _logger;
    
    [HttpPost("votiro/scan-complete")]
    public async Task<IActionResult> OnVotiroScanComplete(
        [FromBody] VotiroWebhookPayload payload,
        [FromServices] IPublishEndpoint publishEndpoint)
    {
        _logger.LogInformation("Votiro webhook: JobId={JobId}, Status={Status}",
            payload.JobId, payload.Status);
        
        try
        {
            // Find attachment by Votiro job ID
            var scan = await _repo.GetScanByVotiroJobIdAsync(payload.JobId);
            if (scan == null)
            {
                return NotFound();
            }
            
            // Update scan with results
            scan.ScanStatus = payload.Status;
            scan.RiskLevel = payload.RiskLevel;
            scan.ThreatsFound = JsonSerializer.Serialize(payload.Threats);
            scan.ScanCompletedAt = DateTime.UtcNow;
            
            // Determine if file safe to download
            scan.DownloadAllowed = payload.Status == "CLEAN" ||
                (payload.Status == "INFECTED" && payload.RemediationPossible);
            scan.DownloadReason = payload.RemediationPossible
                ? "File scanned and cleaned"
                : "File scanned but contains threats";
            
            if (payload.Status == "INFECTED" && payload.RemediationPossible)
            {
                scan.CleanFilePath = payload.CleanFilePath;
                scan.CleanFileSize = payload.CleanFileSize;
            }
            
            await _repo.UpdateScanAsync(scan);
            
            // Publish event for notification
            await publishEndpoint.Publish(new DocumentScanCompletedEvent
            {
                ScanId = scan.Id,
                AttachmentId = scan.AttachmentId,
                Status = scan.ScanStatus,
                RiskLevel = scan.RiskLevel,
                DownloadAllowed = scan.DownloadAllowed
            });
            
            return Ok();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing Votiro webhook");
            return StatusCode(500, ex.Message);
        }
    }
}
```

---

## Error Handling & Retries

### Retry Strategy

```csharp
// Exponential backoff: 1s, 2s, 4s, 8s, 16s (max 5 retries)
e.UseMessageRetry(r =>
{
    r.Exponential(
        retryCount: 5,
        initialInterval: TimeSpan.FromSeconds(1),
        intervalIncrement: TimeSpan.FromSeconds(1),
        intervalIncrementFactor: 2.0,
        // Skip retry for specific exceptions
        retryAfter: ctx =>
        {
            return ctx.Exception switch
            {
                ArgumentException => TimeSpan.Zero,  // Don't retry
                _ => TimeSpan.FromSeconds(1)
            };
        });
});
```

### Dead Letter Handling

```csharp
e.UseMessageRetry(r =>
{
    r.Exponential(retryCount: 5, initialInterval: TimeSpan.FromSeconds(1),
        intervalIncrement: TimeSpan.FromSeconds(1), intervalIncrementFactor: 2.0);
});

e.ReceiveObservers.Add(
    new DeadLetterObserver(_logger, _repo));

public class DeadLetterObserver : IReceiveObserver
{
    public async Task PostConsume<T>(ConsumeContext<T> context, 
        Exception exception) where T : class
    {
        _logger.LogError(exception, 
            "Message moved to dead letter: {MessageType}", 
            typeof(T).Name);
        
        // Create incident record for manual review
        await _repo.CreateDeadLetterIncidentAsync(new DeadLetterIncident
        {
            Id = Guid.NewGuid(),
            MessageType = typeof(T).Name,
            MessageContent = JsonSerializer.Serialize(context.Message),
            ExceptionMessage = exception.Message,
            CreatedAt = DateTime.UtcNow
        });
    }
}
```

### Idempotency

```csharp
// Prevent processing same event twice
public class IdempotencyFilter : IConsumeFilter
{
    private readonly IIdempotencyCache _cache;
    
    public async Task Send<T>(ConsumeContext<T> context, 
        IPipe<ConsumeContext<T>> next) where T : class
    {
        var eventId = context.Message.GetPropertyValue("event_id");
        
        if (_cache.Contains(eventId))
        {
            _logger.LogInformation("Event already processed: {EventId}", eventId);
            return;
        }
        
        try
        {
            await next.Send(context);
            _cache.Add(eventId, TimeSpan.FromHours(24));
        }
        catch
        {
            // On failure, don't mark as processed (allow retry)
            throw;
        }
    }
}
```

---

## Testing

### Unit Tests

```csharp
[TestClass]
public class AssetEventHandlerTests
{
    private Mock<IAssetAuditService> _auditServiceMock;
    private Mock<IRepository> _repoMock;
    private Mock<ILogger<AssetEventHandler>> _loggerMock;
    private AssetEventHandler _handler;
    
    [TestInitialize]
    public void Setup()
    {
        _auditServiceMock = new Mock<IAssetAuditService>();
        _repoMock = new Mock<IRepository>();
        _loggerMock = new Mock<ILogger<AssetEventHandler>>();
        
        _handler = new AssetEventHandler(
            _auditServiceMock.Object,
            _repoMock.Object,
            _loggerMock.Object);
    }
    
    [TestMethod]
    public async Task Consume_AssetCreated_CreatesAuditRecord()
    {
        // Arrange
        var @event = new AssetCreated
        {
            AssetMasterId = Guid.NewGuid(),
            AssetCode = "AST-001",
            Name = "Test Asset",
            EventUserId = Guid.NewGuid(),
            EventUserName = "test@company.com",
            EventTimestamp = DateTime.UtcNow,
            EventCorrelationId = "corr-123"
        };
        
        var context = Substitute.For<ConsumeContext<IAssetEvent>>();
        context.Message.Returns(@event);
        
        // Act
        await _handler.Consume(context);
        
        // Assert
        _repoMock.Verify(x => x.CreateAuditAsync(
            It.Is<AssetAudit>(a => a.AssetMasterId == @event.AssetMasterId)));
    }
}
```

### Integration Tests

```csharp
[TestClass]
public class EventHandlerIntegrationTests : IAsyncLifetime
{
    private TestHarness _harness;
    private IHost _host;
    
    public async Task InitializeAsync()
    {
        _host = new HostBuilder()
            .ConfigureServices((context, services) =>
            {
                services.AddMassTransitTestHarness(x =>
                {
                    x.AddConsumer<AssetEventHandler>();
                });
            })
            .Build();
        
        _harness = _host.Services.GetRequiredService<TestHarness>();
        await _host.StartAsync();
    }
    
    [TestMethod]
    public async Task Published_AssetCreated_ConsumerReceives()
    {
        // Arrange
        var @event = new AssetCreated { /* ... */ };
        
        // Act
        await _harness.Bus.Publish(@event);
        
        // Assert
        Assert.IsTrue(await _harness.Consumed.Any<AssetCreated>());
    }
    
    public async Task DisposeAsync()
    {
        await _host?.StopAsync();
        _host?.Dispose();
    }
}
```

---

## Deployment

### Azure Service Bus Topics

```csharp
// Automatic topic creation
cfg.ReceiveEndpoint("asset-events-subscription", e =>
{
    e.ConfigureConsumer<AssetEventHandler>(context);
    e.Bind<IAssetEvent>(x =>
    {
        x.ExchangeType = "Topic";
        x.Durable = true;
    });
});
```

### Health Check

```csharp
services
    .AddHealthChecks()
    .AddCheck("MassTransit", new MassTransitHealthCheck(_host));

app.MapHealthChecks("/health/handlers");
```

### Monitoring

```csharp
// Application Insights integration
services.AddApplicationInsightsTelemetry();

_logger.LogInformation("Handler started", new
{
    ConsumerType = "AssetEventHandler",
    Queue = "asset-event-queue",
    Timestamp = DateTime.UtcNow
});
```

---

## Related Documents
- [7_HANDLERS.md](./7_HANDLERS.md) — Handler architecture
- [6_EVENTS_AUDIT.md](./6_EVENTS_AUDIT.md) — Event types
- [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) — Handler reference
- [FAQ.md](./FAQ.md) — Handler FAQs

---

**Last Updated:** May 3, 2026 | **Next Review:** Phase 2 completion
