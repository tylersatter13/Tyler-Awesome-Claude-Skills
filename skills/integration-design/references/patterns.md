# Integration patterns (Azure, .NET)

Adapt to what the repo already uses. These are starting points, not requirements.

## Typed partner client with resilience

```csharp
builder.Services
    .AddOptions<PartnerOptions>()
    .BindConfiguration("Partner")
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services
    .AddHttpClient<PartnerClient>((sp, http) =>
    {
        var options = sp.GetRequiredService<IOptions<PartnerOptions>>().Value;
        http.BaseAddress = new Uri(options.BaseUrl);
    })
    .AddHttpMessageHandler<PartnerAuthHandler>()   // adds the bearer token from a cached token provider
    .AddStandardResilienceHandler(o =>
    {
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(20);
        o.Retry.MaxRetryAttempts = 3;
        // Only retry non-idempotent calls when they carry an idempotency key.
        o.Retry.DisableForUnsafeHttpMethods();
    });
```

The client returns domain types, not partner DTOs:

```csharp
public sealed class PartnerClient(HttpClient http, ILogger<PartnerClient> logger)
{
    public async Task<Refund> GetRefundAsync(string id, CancellationToken ct)
    {
        var dto = await http.GetFromJsonAsync<PartnerRefundDto>($"v2/refunds/{id}", ct)
                  ?? throw new PartnerContractException("Empty refund body");
        return PartnerRefundMapper.ToDomain(dto);
    }
}
```

## Secrets and Azure clients via Managed Identity

```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri(builder.Configuration["KeyVault:Uri"]!), new DefaultAzureCredential());

builder.Services.AddAzureClients(clients =>
{
    clients.UseCredential(new DefaultAzureCredential());
    clients.AddServiceBusClientWithNamespace(builder.Configuration["ServiceBus:Namespace"]!);
    clients.AddBlobServiceClient(new Uri(builder.Configuration["Storage:BlobUri"]!));
});
```

## Outbox (publish after save)

Write the message to an `OutboxMessages` table in the same `SaveChangesAsync` as the business change.
A background service reads unsent rows, sends them to Service Bus with `MessageId` set to the
outbox row ID (so duplicate detection works), and marks them sent. If the repo uses MassTransit or
Wolverine, use their built-in EF Core outbox instead of hand-rolling one.

## Idempotent Service Bus consumer

- Use `ServiceBusProcessor` with `AutoCompleteMessages = false`.
- Before processing, check an `InboxMessages` table (or a unique constraint) for `MessageId`.
  If it's already there, complete the message and return.
- Process the message and insert the inbox row in one transaction, then complete the message.
- When a message can never succeed (bad payload), dead-letter it with a reason instead of letting it retry.

## Webhook receiver

```csharp
[HttpPost("api/v1/webhooks/partner")]
[AllowAnonymous] // authenticated by signature below
public async Task<IActionResult> Receive(CancellationToken ct)
{
    var body = await new StreamReader(Request.Body).ReadToEndAsync(ct);
    if (!signatureValidator.IsValid(Request.Headers["X-Signature"], body))
        return Unauthorized();

    await inbox.StoreIfNewAsync(eventId: ExtractEventId(body), body, ct); // dedupe on partner event ID
    return Accepted();                                                    // process asynchronously
}
```

Compare signatures with `CryptographicOperations.FixedTimeEquals`, and reject stale timestamps
to prevent replay.
