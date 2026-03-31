# Padroes Distribuidos

## Objetivo

Padroes para sistemas distribuidos: Outbox Pattern, Saga com MassTransit, idempotencia e distributed tracing com OpenTelemetry.

## Quando usar

- Garantir consistencia entre banco e mensageria (Outbox)
- Orquestrar fluxos de negocio com multiplos passos (Saga)
- Garantir idempotencia em operacoes distribuidas
- Configurar rastreamento distribuido (OpenTelemetry)

---

## Outbox Pattern

Problema: salvar no banco e publicar mensagem nao sao atomicos. Se o publish falhar apos o commit, a mensagem se perde.

Solucao: salvar a mensagem na mesma transacao do banco. Um background service despacha depois.

### Entidade OutboxMessage

```csharp
// {NomeProjeto}.Domain/Outbox/OutboxMessage.cs
public sealed class OutboxMessage
{
    public Guid Id { get; init; } = Guid.CreateVersion7();
    public string Type { get; init; } = string.Empty;
    public string Content { get; init; } = string.Empty;
    public DateTime CreatedAtUtc { get; init; } = DateTime.UtcNow;
    public DateTime? ProcessedAtUtc { get; private set; }
    public string? Error { get; private set; }
    public int RetryCount { get; private set; }

    public void MarkAsProcessed() => ProcessedAtUtc = DateTime.UtcNow;

    public void MarkAsFailed(string error)
    {
        Error = error;
        RetryCount++;
    }
}
```

### Configuracao EF Core

```csharp
// {NomeProjeto}.Infrastructure/Persistence/Configurations/OutboxMessageConfiguration.cs
public sealed class OutboxMessageConfiguration : IEntityTypeConfiguration<OutboxMessage>
{
    public void Configure(EntityTypeBuilder<OutboxMessage> builder)
    {
        builder.ToTable("outbox_messages");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Type).HasMaxLength(500).IsRequired();
        builder.Property(x => x.Content).IsRequired();
        builder.HasIndex(x => x.ProcessedAtUtc)
            .HasFilter("processed_at_utc IS NULL"); // indice parcial para pendentes
    }
}
```

### Salvando no Handler (mesma transacao)

```csharp
public async Task<ErrorOr<PaymentResponse>> Handle(
    ProcessPaymentCommand request,
    CancellationToken cancellationToken)
{
    var payment = customer.CreatePayment(request.AmountInCents);
    await paymentRepository.AddAsync(payment, cancellationToken);

    // Salvar mensagem no outbox — mesma transacao
    var outboxMessage = new OutboxMessage
    {
        Type = nameof(PaymentProcessed),
        Content = JsonSerializer.Serialize(new PaymentProcessed(
            payment.Id,
            payment.CustomerId,
            payment.AmountInCents,
            DateTime.UtcNow
        ))
    };
    dbContext.Set<OutboxMessage>().Add(outboxMessage);

    await unitOfWork.SaveChangesAsync(cancellationToken); // commit atomico

    return payment.ToResponse();
}
```

### OutboxProcessor (Background Service)

```csharp
// {NomeProjeto}.Workers/OutboxProcessor.cs
public sealed class OutboxProcessor(
    IServiceScopeFactory scopeFactory,
    IPublishEndpoint publishEndpoint,
    ILogger<OutboxProcessor> logger
) : BackgroundService
{
    private readonly TimeSpan _interval = TimeSpan.FromSeconds(5);
    private const int BatchSize = 50;
    private const int MaxRetries = 5;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("OutboxProcessor iniciado");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = scopeFactory.CreateScope();
                var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();

                var messages = await dbContext.Set<OutboxMessage>()
                    .Where(m => m.ProcessedAtUtc == null && m.RetryCount < MaxRetries)
                    .OrderBy(m => m.CreatedAtUtc)
                    .Take(BatchSize)
                    .ToListAsync(stoppingToken);

                foreach (var message in messages)
                {
                    try
                    {
                        var messageType = GetMessageType(message.Type);
                        var payload = JsonSerializer.Deserialize(message.Content, messageType)!;

                        await publishEndpoint.Publish(payload, stoppingToken);

                        message.MarkAsProcessed();
                        logger.LogInformation(
                            "Outbox message processada: {MessageId} ({Type})",
                            message.Id, message.Type);
                    }
                    catch (Exception ex)
                    {
                        message.MarkAsFailed(ex.Message);
                        logger.LogError(ex,
                            "Falha ao processar outbox message: {MessageId} (tentativa {Retry})",
                            message.Id, message.RetryCount);
                    }
                }

                if (messages.Count > 0)
                    await dbContext.SaveChangesAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Erro no OutboxProcessor");
            }

            await Task.Delay(_interval, stoppingToken);
        }
    }

    private static Type GetMessageType(string typeName)
    {
        // Mapear nomes para tipos — ajustar conforme necessidade
        return typeName switch
        {
            nameof(PaymentProcessed) => typeof(PaymentProcessed),
            nameof(OrderCreated) => typeof(OrderCreated),
            _ => throw new InvalidOperationException($"Tipo de mensagem desconhecido: {typeName}")
        };
    }
}
```

---

## Saga Pattern com MassTransit

### Quando usar

- Fluxos de negocio com multiplos passos (ex: pedido → pagamento → envio)
- Cada passo pode falhar e precisa de compensacao
- Coordenacao entre multiplos servicos

### State Machine — Processamento de Pedido

```csharp
// {NomeProjeto}.Workers/Sagas/OrderStateMachine.cs

// Estados
public sealed class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = string.Empty;
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public long AmountInCents { get; set; }
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? PaidAtUtc { get; set; }
    public DateTime? ShippedAtUtc { get; set; }
    public string? FailureReason { get; set; }
}

// Eventos (contratos)
public sealed record OrderSubmitted(Guid OrderId, Guid CustomerId, long AmountInCents);
public sealed record PaymentCompleted(Guid OrderId, DateTime PaidAtUtc);
public sealed record PaymentFailed(Guid OrderId, string Reason);
public sealed record OrderShipped(Guid OrderId, DateTime ShippedAtUtc);
public sealed record ShipmentFailed(Guid OrderId, string Reason);

// Comandos de compensacao
public sealed record RefundPayment(Guid OrderId, long AmountInCents);
public sealed record CancelOrder(Guid OrderId, string Reason);
```

```csharp
// State Machine
public sealed class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    // Estados
    public State Submitted { get; private set; } = null!;
    public State Paid { get; private set; } = null!;
    public State Shipped { get; private set; } = null!;
    public State Failed { get; private set; } = null!;
    public State Cancelled { get; private set; } = null!;

    // Eventos
    public Event<OrderSubmitted> OrderSubmittedEvent { get; private set; } = null!;
    public Event<PaymentCompleted> PaymentCompletedEvent { get; private set; } = null!;
    public Event<PaymentFailed> PaymentFailedEvent { get; private set; } = null!;
    public Event<OrderShipped> OrderShippedEvent { get; private set; } = null!;
    public Event<ShipmentFailed> ShipmentFailedEvent { get; private set; } = null!;

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        // Correlacao por OrderId
        Event(() => OrderSubmittedEvent, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentCompletedEvent, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentFailedEvent, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => OrderShippedEvent, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => ShipmentFailedEvent, x => x.CorrelateById(ctx => ctx.Message.OrderId));

        // Fluxo: Initial → Submitted → Paid → Shipped (ou Failed/Cancelled)
        Initially(
            When(OrderSubmittedEvent)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId = ctx.Message.OrderId;
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.AmountInCents = ctx.Message.AmountInCents;
                    ctx.Saga.CreatedAtUtc = DateTime.UtcNow;
                })
                .TransitionTo(Submitted)
        );

        During(Submitted,
            When(PaymentCompletedEvent)
                .Then(ctx => ctx.Saga.PaidAtUtc = ctx.Message.PaidAtUtc)
                .TransitionTo(Paid),

            When(PaymentFailedEvent)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                // Compensacao: cancelar pedido
                .Publish(ctx => new CancelOrder(ctx.Saga.OrderId, ctx.Message.Reason))
                .TransitionTo(Cancelled)
        );

        During(Paid,
            When(OrderShippedEvent)
                .Then(ctx => ctx.Saga.ShippedAtUtc = ctx.Message.ShippedAtUtc)
                .TransitionTo(Shipped)
                .Finalize(),

            When(ShipmentFailedEvent)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                // Compensacao: estornar pagamento
                .Publish(ctx => new RefundPayment(ctx.Saga.OrderId, ctx.Saga.AmountInCents))
                .TransitionTo(Failed)
        );
    }
}
```

### Registro

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderStateMachine, OrderState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Optimistic;
            r.AddDbContext<DbContext, AppDbContext>();
        });

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

---

## Idempotencia

Toda operacao distribuida deve ser idempotente. Estrategias (referencia: `/background-workers`):

### 1. Chave de idempotencia no banco

```csharp
// Indice unico na coluna de referencia
builder.HasIndex(x => x.ExternalReference).IsUnique();

// No Handler: INSERT com conflito = operacao ja processada
```

### 2. Cache Redis com TTL

```csharp
var idempotencyKey = $"idempotent:{message.OrderId}:{message.GetType().Name}";
var wasSet = await redis.StringSetAsync(idempotencyKey, "1", TimeSpan.FromHours(24), When.NotExists);
if (!wasSet)
{
    logger.LogInformation("Mensagem ja processada: {Key}", idempotencyKey);
    return; // ignora duplicata
}
```

### 3. MassTransit InMemoryOutbox

```csharp
// Garante que mensagens publicadas dentro de um consumer so sao enviadas apos sucesso
cfg.ReceiveEndpoint("order-processing", e =>
{
    e.UseInMemoryOutbox(context);
    e.ConfigureConsumer<OrderProcessingConsumer>(context);
});
```

---

## Distributed Tracing com OpenTelemetry

### Pacotes NuGet

```xml
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.*" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.*" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.*" />
<PackageReference Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" Version="1.*" />
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.*" />
```

### Configuracao no Program.cs

```csharp
// {NomeProjeto}.API/Extensions/ObservabilityExtensions.cs
public static IServiceCollection AddObservability(
    this IServiceCollection services,
    IConfiguration configuration)
{
    services.AddOpenTelemetry()
        .ConfigureResource(resource => resource
            .AddService(
                serviceName: "MeuApp.API",
                serviceVersion: Assembly.GetExecutingAssembly()
                    .GetName().Version?.ToString() ?? "1.0.0"
            ))
        .WithTracing(tracing => tracing
            .AddAspNetCoreInstrumentation(opts =>
            {
                // Filtrar health checks do tracing
                opts.Filter = ctx => !ctx.Request.Path.StartsWithSegments("/health");
            })
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddSource("MassTransit") // MassTransit emite traces automaticamente
            .AddOtlpExporter(opts =>
            {
                opts.Endpoint = new Uri(
                    configuration["OpenTelemetry:Endpoint"] ?? "http://localhost:4317"
                );
            }))
        .WithMetrics(metrics => metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddOtlpExporter());

    return services;
}
```

### Propagacao do Correlation ID

O OpenTelemetry propaga automaticamente o `TraceId` via headers W3C (`traceparent`). Para manter compatibilidade com o `X-Correlation-Id` customizado:

```csharp
public sealed class CorrelationIdMiddleware(RequestDelegate next)
{
    private const string Header = "X-Correlation-Id";

    public async Task InvokeAsync(HttpContext context)
    {
        // Usar TraceId do OpenTelemetry como correlation ID quando disponivel
        var correlationId = context.Request.Headers[Header].FirstOrDefault()
            ?? Activity.Current?.TraceId.ToString()
            ?? Guid.CreateVersion7().ToString();

        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers[Header] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await next(context);
        }
    }
}
```

### Docker Compose — Jaeger (observabilidade local)

```yaml
# Adicionar ao docker-compose.yml
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: app-jaeger
    ports:
      - "4317:4317"   # OTLP gRPC
      - "16686:16686" # UI
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

> Acesse `http://localhost:16686` para visualizar traces no Jaeger.

---

## Checklist

- [ ] Outbox Pattern implementado para garantir consistencia banco + mensageria
- [ ] `OutboxMessage` salva na mesma transacao do comando
- [ ] `OutboxProcessor` roda como Background Service com retry e max retries
- [ ] Indice parcial na tabela `outbox_messages` para mensagens pendentes
- [ ] Saga State Machine definida com estados, eventos e compensacoes
- [ ] Saga persistida no banco via EF Core com concorrencia otimista
- [ ] Transacoes compensatorias implementadas para falhas em cada passo
- [ ] Idempotencia garantida em todos os consumers e handlers
- [ ] OpenTelemetry configurado com tracing e metricas
- [ ] Health checks filtrados do tracing
- [ ] Correlation ID integrado com TraceId do OpenTelemetry
- [ ] Jaeger ou outro backend de tracing configurado para desenvolvimento
