# Background Workers e Consumers

## Objetivo

Padrões para Background Services (.NET) e consumers RabbitMQ (MassTransit).

## Quando usar

- Criar worker para processamento assíncrono
- Implementar consumer de fila RabbitMQ
- Configurar retry, DLQ e idempotência

---

## Background Service (.NET)

### Quando usar

- Tarefas periódicas (limpeza, relatórios, reconciliação)
- Processamento que não depende de mensagem externa
- Polling de APIs externas

### Template

```csharp
public sealed class CleanupExpiredTokensWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<CleanupExpiredTokensWorker> logger
) : BackgroundService
{
    private readonly TimeSpan _interval = TimeSpan.FromHours(1);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("CleanupExpiredTokensWorker iniciado");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = scopeFactory.CreateScope();
                var repo = scope.ServiceProvider.GetRequiredService<ITokenRepository>();
                var unitOfWork = scope.ServiceProvider.GetRequiredService<IUnitOfWork>();

                var removed = await repo.RemoveExpiredAsync(stoppingToken);
                await unitOfWork.SaveChangesAsync(stoppingToken);

                logger.LogInformation("Removidos {Count} tokens expirados", removed);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Erro no CleanupExpiredTokensWorker");
            }

            await Task.Delay(_interval, stoppingToken);
        }
    }
}
```

### Regras

- Sempre use `IServiceScopeFactory` — workers são singleton, DbContext é scoped
- Capture exceções dentro do loop — nunca deixe o worker morrer
- Use `CancellationToken` para graceful shutdown
- Logue início, execução e erros

### Registro

```csharp
builder.Services.AddHostedService<CleanupExpiredTokensWorker>();
```

---

## MassTransit Consumer (RabbitMQ)

### Setup

```xml
<PackageReference Include="MassTransit.RabbitMQ" Version="8.*" />
```

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumersFromNamespaceContaining<PaymentProcessedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("app");
            h.Password("app_dev_password");
        });

        cfg.ConfigureEndpoints(context);
    });
});
```

### Mensagem (Contrato)

```csharp
// Mensagens são records imutáveis — compartilhados entre publisher e consumer
public sealed record PaymentProcessed(
    Guid PaymentId,
    Guid CustomerId,
    long AmountInCents,
    DateTime ProcessedAtUtc
);
```

### Consumer

```csharp
public sealed class PaymentProcessedConsumer(
    INotificationService notificationService,
    ILogger<PaymentProcessedConsumer> logger
) : IConsumer<PaymentProcessed>
{
    public async Task Consume(ConsumeContext<PaymentProcessed> context)
    {
        var msg = context.Message;

        logger.LogInformation(
            "Processando notificação para pagamento {PaymentId}",
            msg.PaymentId
        );

        // Idempotência: verificar se já processou esta mensagem
        // Usar PaymentId como chave de idempotência

        await notificationService.SendPaymentConfirmationAsync(
            msg.CustomerId,
            msg.AmountInCents,
            context.CancellationToken
        );
    }
}
```

### Publicar Mensagem

```csharp
// No Handler, após SaveChangesAsync
await publishEndpoint.Publish(
    new PaymentProcessed(
        payment.Id,
        payment.CustomerId,
        payment.AmountInCents,
        DateTime.UtcNow
    ),
    cancellationToken
);
```

---

## Retry e Dead Letter Queue

### Configuração Global

```csharp
cfg.UseMessageRetry(r => r
    .Incremental(
        retryLimit: 3,
        initialInterval: TimeSpan.FromSeconds(1),
        intervalIncrement: TimeSpan.FromSeconds(2)
    )
);
```

### Por Consumer

```csharp
cfg.ReceiveEndpoint("payment-processed", e =>
{
    e.UseMessageRetry(r => r
        .Exponential(
            retryLimit: 5,
            minInterval: TimeSpan.FromSeconds(1),
            maxInterval: TimeSpan.FromMinutes(5),
            intervalDelta: TimeSpan.FromSeconds(2)
        )
    );

    // Dead Letter Queue automática após esgotar retries
    e.ConfigureConsumer<PaymentProcessedConsumer>(context);
});
```

### Regras de Retry

| Tipo de erro | Ação |
|-------------|------|
| Erro de negócio | Falha direta — sem retry |
| Erro de infra (DB, rede) | Retry com backoff exponencial |
| Mensagem malformada | Dead letter — sem retry |
| Timeout | Retry com intervalo crescente |

---

## Idempotência

Todo consumer deve ser idempotente — reprocessar a mesma mensagem não pode causar efeito duplo.

### Estratégias

1. **Verificar antes de processar:**
   ```csharp
   var alreadyProcessed = await repo.ExistsAsync(msg.PaymentId, ct);
   if (alreadyProcessed) return; // já processou, ignora
   ```

2. **Unique constraint no banco:**
   - Índice único na coluna de referência da mensagem
   - INSERT falha com conflito → consumer ignora

3. **Redis como cache de processamento:**
   ```csharp
   var key = $"processed:{msg.PaymentId}";
   var wasSet = await redis.StringSetAsync(key, "1", TimeSpan.FromHours(24), When.NotExists);
   if (!wasSet) return; // já processou
   ```

---

## Graceful Shutdown

```csharp
// MassTransit gerencia automaticamente via IHostedService
// Workers customizados: respeite o CancellationToken

protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        // trabalho...
        await Task.Delay(_interval, stoppingToken);
    }

    logger.LogInformation("Worker finalizando gracefully");
}
```

---

## Checklist

- [ ] Workers usam `IServiceScopeFactory` para criar scope
- [ ] Exceções capturadas dentro do loop — worker nunca morre
- [ ] `CancellationToken` respeitado para graceful shutdown
- [ ] Consumers são idempotentes
- [ ] Retry configurado com backoff para erros de infra
- [ ] Erro de negócio falha direto — sem retry
- [ ] Dead letter queue configurada
- [ ] Mensagens são records imutáveis
- [ ] Valores monetários em `long` (centavos) nas mensagens
- [ ] `DateTime.UtcNow` nas mensagens — nunca `DateTime.Now`
