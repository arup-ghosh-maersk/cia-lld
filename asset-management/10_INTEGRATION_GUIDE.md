# Integration Guide — External Services

> **Date:** May 3, 2026 | **Version:** 1.0 | **Status:** Production Ready (Phase 2)

## Table of Contents
1. [Overview](#overview)
2. [Votiro CDR Integration](#votiro-cdr-integration)
3. [Azure Blob Storage](#azure-blob-storage)
4. [Email Service](#email-service)
5. [Third-Party Webhooks](#third-party-webhooks)
6. [Configuration](#configuration)
7. [Troubleshooting](#troubleshooting)

---

## Overview

The Asset Master integrates with external services for:
- **Votiro** — Malware detection & remediation (CDR)
- **Azure Blob Storage** — Document storage
- **SendGrid/SMTP** — Email notifications
- **Webhooks** — Real-time data sync with external systems

### Integration Flow

```
┌──────────────────────┐
│ Asset Master         │
└──────────┬───────────┘
           ├→ File uploaded
           │
           ├─→ Azure Blob Storage (store file)
           │
           ├─→ Votiro (scan for malware)
           │
           ├─→ SendGrid (notify users)
           │
           └─→ Webhooks (notify subscribers)
```

---

## Votiro CDR Integration

### Overview
**Votiro** provides Content Disarm & Reconstruct to remove malware from documents while preserving content.

### Supported Formats
- Microsoft Office: DOCX, XLSX, PPTX
- PDF documents
- Images: PNG, JPEG, GIF, BMP
- Archives: ZIP, RAR, TAR, 7Z
- Plain text files

### API Endpoints
```
Production: https://api.votiro.com/v1
Staging:    https://staging.votiro.com/v1
```

### Implementation

#### 1. Register Service

```csharp
public static class VotiroIntegration
{
    public static IServiceCollection AddVotiroService(
        this IServiceCollection services,
        IConfiguration config)
    {
        var settings = config.GetSection("Votiro")
            .Get<VotiroSettings>();
        
        services.AddSingleton(settings);
        
        services.AddHttpClient<IVotiroService, VotiroService>(client =>
        {
            client.BaseAddress = new Uri(settings.ApiUrl);
            client.DefaultRequestHeaders.Add(
                "Authorization", $"Bearer {settings.ApiKey}");
            client.DefaultRequestHeaders.Add("User-Agent", "AssetMaster/1.0");
        });
        
        return services;
    }
}
```

#### 2. Settings (appsettings.json)

```json
{
  "Votiro": {
    "ApiKey": "your-api-key-here",
    "ApiUrl": "https://api.votiro.com/v1",
    "ScanTimeout": "00:10:00",
    "WebhookCallbackUrl": "https://assetmaster.company.com/api/webhooks/votiro/scan-complete",
    "MaxFileSize": 104857600
  }
}
```

#### 3. Service Implementation

```csharp
public interface IVotiroService
{
    Task<string> SendForScanAsync(Stream fileStream, string fileName);
    Task<ScanResultDto> GetScanResultAsync(string votiroJobId);
    Task<Stream> DownloadCleanFileAsync(string votiroJobId);
}

public class VotiroService : IVotiroService
{
    private readonly HttpClient _client;
    private readonly VotiroSettings _settings;
    private readonly ILogger<VotiroService> _logger;
    
    public VotiroService(
        HttpClient client,
        VotiroSettings settings,
        ILogger<VotiroService> logger)
    {
        _client = client;
        _settings = settings;
        _logger = logger;
    }
    
    public async Task<string> SendForScanAsync(Stream fileStream, string fileName)
    {
        var content = new MultipartFormDataContent();
        content.Add(new StreamContent(fileStream), "file", fileName);
        
        // Optional: Add webhook callback URL for async notification
        content.Add(
            new StringContent(_settings.WebhookCallbackUrl),
            "callbackUrl");
        
        var response = await _client.PostAsync("/scan", content);
        
        if (!response.IsSuccessStatusCode)
        {
            var errorContent = await response.Content.ReadAsStringAsync();
            throw new VotiroException(
                $"Failed to send file to Votiro: {response.StatusCode}",
                errorContent);
        }
        
        var result = await response.Content.ReadAsAsync<VotiroSubmitResponse>();
        
        _logger.LogInformation(
            "File {FileName} sent to Votiro with Job ID: {JobId}",
            fileName, result.JobId);
        
        return result.JobId;
    }
    
    public async Task<ScanResultDto> GetScanResultAsync(string votiroJobId)
    {
        var response = await _client.GetAsync($"/status/{votiroJobId}");
        
        if (!response.IsSuccessStatusCode)
            throw new VotiroException($"Failed to get scan status for {votiroJobId}");
        
        var result = await response.Content.ReadAsAsync<VotiroScanResponse>();
        
        return new ScanResultDto
        {
            JobId = result.JobId,
            Status = result.Status,
            RiskLevel = result.RiskLevel,
            Threats = result.Threats ?? new List<string>(),
            CleanFilePath = result.CleanFileUrl,
            CompletedAt = DateTime.UtcNow
        };
    }
    
    public async Task<Stream> DownloadCleanFileAsync(string votiroJobId)
    {
        var response = await _client.GetAsync(
            $"/file/{votiroJobId}/download");
        
        if (!response.IsSuccessStatusCode)
            throw new VotiroException($"Failed to download clean file");
        
        return await response.Content.ReadAsStreamAsync();
    }
}

public class VotiroSettings
{
    public string ApiKey { get; set; }
    public string ApiUrl { get; set; }
    public TimeSpan ScanTimeout { get; set; }
    public string WebhookCallbackUrl { get; set; }
    public long MaxFileSize { get; set; }
}

public class VotiroSubmitResponse
{
    [JsonPropertyName("job_id")]
    public string JobId { get; set; }
}

public class VotiroScanResponse
{
    [JsonPropertyName("job_id")]
    public string JobId { get; set; }
    
    [JsonPropertyName("status")]
    public string Status { get; set; }  // PENDING, READY, COMPLETE
    
    [JsonPropertyName("risk_level")]
    public string RiskLevel { get; set; }  // CLEAN, LOW, MEDIUM, HIGH
    
    [JsonPropertyName("threats")]
    public List<string> Threats { get; set; }
    
    [JsonPropertyName("clean_file_url")]
    public string CleanFileUrl { get; set; }
}
```

#### 4. Webhook Handler

```csharp
[ApiController]
[Route("api/v1/webhooks")]
public class VotiroWebhookController : ControllerBase
{
    private readonly IDocumentScanService _scanService;
    private readonly ILogger<VotiroWebhookController> _logger;
    
    [HttpPost("votiro/scan-complete")]
    [AllowAnonymous]  // Votiro sends without auth
    public async Task<IActionResult> OnScanComplete(
        [FromBody] VotiroWebhookPayload payload)
    {
        _logger.LogInformation(
            "Received Votiro webhook: JobId={JobId}, Status={Status}",
            payload.JobId, payload.Status);
        
        try
        {
            await _scanService.UpdateScanResultAsync(payload.JobId, payload.Status);
            return Ok(new { success = true });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing Votiro webhook");
            return BadRequest(ex.Message);
        }
    }
}

public class VotiroWebhookPayload
{
    [JsonPropertyName("job_id")]
    public string JobId { get; set; }
    
    [JsonPropertyName("status")]
    public string Status { get; set; }
    
    [JsonPropertyName("risk_level")]
    public string RiskLevel { get; set; }
    
    [JsonPropertyName("threats")]
    public List<string> Threats { get; set; }
}
```

---

## Azure Blob Storage

### Overview
All documents stored in Azure Blob Storage with encryption at rest.

### Configuration

```json
{
  "AzureStorage": {
    "ConnectionString": "DefaultEndpointsProtocol=https;...",
    "ContainerName": "asset-documents",
    "SasTokenExpirationMinutes": 15,
    "MaxFileSizeBytes": 104857600
  }
}
```

### Implementation

```csharp
public interface IStorageService
{
    Task<string> UploadAsync(IFormFile file, Guid assetId, Guid attachmentId);
    Task<Stream> DownloadAsync(string blobPath);
    Task DeleteAsync(string blobPath);
    Task<string> GenerateSasUrlAsync(string blobPath, int expirationMinutes);
}

public class AzureBlobStorageService : IStorageService
{
    private readonly BlobContainerClient _containerClient;
    private readonly ILogger<AzureBlobStorageService> _logger;
    
    public AzureBlobStorageService(IConfiguration config,
        ILogger<AzureBlobStorageService> logger)
    {
        var settings = config.GetSection("AzureStorage").Get<StorageSettings>();
        
        var blobServiceClient = new BlobServiceClient(
            settings.ConnectionString);
        _containerClient = blobServiceClient.GetBlobContainerClient(
            settings.ContainerName);
        _logger = logger;
    }
    
    public async Task<string> UploadAsync(
        IFormFile file, Guid assetId, Guid attachmentId)
    {
        // Create blob path: assets/{assetId}/{attachmentId}/filename
        var blobPath = $"assets/{assetId}/{attachmentId}/{file.FileName}";
        var blobClient = _containerClient.GetBlobClient(blobPath);
        
        // Upload with encryption
        await blobClient.UploadAsync(
            file.OpenReadStream(),
            overwrite: true,
            cancellationToken: CancellationToken.None);
        
        _logger.LogInformation(
            "File uploaded to blob: {BlobPath}", blobPath);
        
        return blobPath;
    }
    
    public async Task<Stream> DownloadAsync(string blobPath)
    {
        var blobClient = _containerClient.GetBlobClient(blobPath);
        var download = await blobClient.DownloadAsync();
        return download.Value.Content;
    }
    
    public async Task DeleteAsync(string blobPath)
    {
        var blobClient = _containerClient.GetBlobClient(blobPath);
        await blobClient.DeleteAsync();
    }
    
    public async Task<string> GenerateSasUrlAsync(
        string blobPath, int expirationMinutes)
    {
        var blobClient = _containerClient.GetBlobClient(blobPath);
        
        // Generate read-only SAS token
        var sasUri = blobClient.GenerateSasUri(
            BlobSasPermissions.Read,
            DateTimeOffset.UtcNow.AddMinutes(expirationMinutes));
        
        return sasUri.ToString();
    }
}
```

### Data Lifecycle

```
Day 0:    File uploaded → Scanned by Votiro
Day 7:    Can be archived to cool storage
Day 90:   Can be archived to archive storage
Day 2555: Deleted per retention policy
```

---

## Email Service

### Overview
SendGrid sends notifications (asset creation, status changes, scan results).

### Configuration

```json
{
  "Email": {
    "Provider": "SendGrid",
    "ApiKey": "SG.xxx",
    "FromAddress": "noreply@company.com",
    "FromName": "Asset Master",
    "SmtpAlternative": {
      "Host": "smtp.company.com",
      "Port": 587,
      "Username": "user@company.com",
      "Password": "password"
    }
  }
}
```

### Implementation

```csharp
public interface IEmailService
{
    Task SendAsync(EmailMessage message);
    Task SendBulkAsync(List<EmailMessage> messages);
}

public class SendGridEmailService : IEmailService
{
    private readonly SendGridClient _client;
    private readonly ILogger<SendGridEmailService> _logger;
    
    public SendGridEmailService(IConfiguration config,
        ILogger<SendGridEmailService> logger)
    {
        var settings = config.GetSection("Email").Get<EmailSettings>();
        _client = new SendGridClient(settings.ApiKey);
        _logger = logger;
    }
    
    public async Task SendAsync(EmailMessage message)
    {
        var msg = new SendGridMessage()
        {
            From = new EmailAddress("noreply@company.com", "Asset Master"),
            Subject = message.Subject,
            HtmlContent = message.BodyHtml
        };
        
        msg.AddTo(new EmailAddress(message.ToAddress));
        
        var response = await _client.SendEmailAsync(msg);
        
        if (!response.IsSuccessStatusCode)
        {
            _logger.LogError(
                "Failed to send email to {Email}: {StatusCode}",
                message.ToAddress, response.StatusCode);
            throw new EmailException($"SendGrid returned {response.StatusCode}");
        }
        
        _logger.LogInformation("Email sent to {Email}", message.ToAddress);
    }
}

public class EmailMessage
{
    public string ToAddress { get; set; }
    public string Subject { get; set; }
    public string BodyHtml { get; set; }
}
```

### Email Templates

#### Asset Created

```html
<h2>Asset Created</h2>
<p>A new asset has been created:</p>
<table>
  <tr><td>Asset Code:</td><td>{{AssetCode}}</td></tr>
  <tr><td>Name:</td><td>{{Name}}</td></tr>
  <tr><td>Status:</td><td>{{Status}}</td></tr>
  <tr><td>Created By:</td><td>{{CreatedBy}}</td></tr>
</table>
<a href="{{AssetUrl}}">View Asset</a>
```

#### Document Scan Complete

```html
<h2>Document Scan Complete</h2>
<p>Document: {{FileName}}</p>
<p>Status: <strong>{{ScanStatus}}</strong></p>
<p>Risk Level: {{RiskLevel}}</p>

{{#if DownloadAllowed}}
  <p><a href="{{DownloadUrl}}">Download Document</a></p>
{{/if}}

{{#if ThreatsFound}}
  <p><strong>Threats Detected:</strong></p>
  <ul>
    {{#each Threats}}
      <li>{{this}}</li>
    {{/each}}
  </ul>
{{/if}}
```

---

## Third-Party Webhooks

### Overview
External systems can subscribe to asset events via webhooks.

### Configuration

```json
{
  "Webhooks": {
    "Timeout": "00:00:30",
    "RetryCount": 3,
    "RetryDelayMs": 5000,
    "EnableTLS12": true
  }
}
```

### Implementation

```csharp
public interface IWebhookService
{
    Task SubscribeAsync(string url, string[] eventTypes);
    Task UnsubscribeAsync(string url);
    Task PublishAsync(string eventType, object payload);
}

public class WebhookService : IWebhookService
{
    private readonly HttpClient _client;
    private readonly IRepository _repo;
    private readonly ILogger<WebhookService> _logger;
    
    public async Task PublishAsync(string eventType, object payload)
    {
        var subscriptions = await _repo.GetWebhookSubscriptionsAsync(eventType);
        
        foreach (var subscription in subscriptions)
        {
            _ = Task.Run(async () =>
            {
                await SendWebhookWithRetryAsync(subscription.Url, eventType, payload);
            });
        }
    }
    
    private async Task SendWebhookWithRetryAsync(
        string url, string eventType, object payload)
    {
        var content = JsonContent.Create(new WebhookPayload
        {
            EventType = eventType,
            Timestamp = DateTime.UtcNow,
            Data = payload
        });
        
        int retryCount = 0;
        while (retryCount < 3)
        {
            try
            {
                var response = await _client.PostAsync(url, content);
                response.EnsureSuccessStatusCode();
                
                _logger.LogInformation(
                    "Webhook delivered: {Url} for {EventType}",
                    url, eventType);
                return;
            }
            catch (Exception ex)
            {
                retryCount++;
                _logger.LogWarning(ex,
                    "Webhook delivery failed (retry {Retry}/3): {Url}",
                    retryCount, url);
                
                if (retryCount < 3)
                    await Task.Delay(5000 * retryCount);
            }
        }
        
        _logger.LogError(
            "Webhook delivery failed after 3 retries: {Url}", url);
    }
}

public class WebhookPayload
{
    public string EventType { get; set; }
    public DateTime Timestamp { get; set; }
    public object Data { get; set; }
}
```

### Supported Events
- `asset.created`
- `asset.updated`
- `asset.attribute_changed`
- `asset.status_changed`
- `document.uploaded`
- `document.scan_complete`

---

## Configuration

### appsettings.Production.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Warning"
    }
  },
  "Votiro": {
    "ApiUrl": "https://api.votiro.com/v1",
    "ApiKey": "${VOTIRO_API_KEY}",
    "ScanTimeout": "00:10:00"
  },
  "AzureStorage": {
    "ConnectionString": "${AZURE_STORAGE_CONNECTION_STRING}",
    "ContainerName": "asset-documents"
  },
  "Email": {
    "ApiKey": "${SENDGRID_API_KEY}",
    "FromAddress": "noreply@company.com"
  },
  "AzureServiceBus": {
    "ConnectionString": "${AZURE_SERVICEBUS_CONNECTION_STRING}"
  }
}
```

### Secrets Management (User Secrets for Dev)

```bash
dotnet user-secrets set "Votiro:ApiKey" "your-api-key"
dotnet user-secrets set "Email:ApiKey" "your-sendgrid-key"
dotnet user-secrets set "AzureStorage:ConnectionString" "your-connection-string"
```

---

## Troubleshooting

### Votiro: File Not Scanning

**Symptom:** `scan_status = PENDING` for >30 minutes

```sql
-- Check scan queue
SELECT * FROM DOCUMENT_SCAN
WHERE scan_status = 'PENDING'
  AND created_at < NOW() - INTERVAL '30 minutes';

-- Check Votiro job status
SELECT votiro_job_id FROM DOCUMENT_SCAN
WHERE id = '<scan_id>'
```

**Solution:**
1. Check Votiro API key in configuration
2. Verify file size < 100MB
3. Check Azure → Votiro network connectivity
4. Review Votiro API logs

### Azure: File Upload Fails

**Symptom:** 403 Forbidden from Azure Storage

```
Error: "Authentication failure. The specified account does not exist."
```

**Solution:**
```csharp
// Verify connection string
var connectionString = config["AzureStorage:ConnectionString"];
var client = new BlobServiceClient(connectionString);
await client.GetAccountInfoAsync();  // Test connectivity
```

### Email: Not Delivered

**Symptom:** Emails not received

```sql
-- Check notification log
SELECT * FROM NOTIFICATION_LOG
WHERE notification_type = 'EmailNotification'
  AND status = 'FAILED'
ORDER BY created_at DESC;
```

**Solution:**
1. Verify SendGrid API key
2. Check spam folder
3. Review SendGrid logs at sendgrid.com
4. Verify FromAddress is verified in SendGrid

---

## Related Documents
- [5_COMPONENTS_ATTACHMENTS.md](./5_COMPONENTS_ATTACHMENTS.md) — File management
- [7_HANDLERS.md](./7_HANDLERS.md) — Webhook handlers
- [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) — Integration quick ref

---

**Last Updated:** May 3, 2026 | **Next Review:** Phase 2 completion
