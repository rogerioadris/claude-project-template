# Logging e Observabilidade

## Objetivo

Configurar structured logging, correlation IDs e health checks para monitoramento da aplicação.

## Quando usar

- Configurar logging no projeto
- Adicionar health checks para novos serviços
- Investigar problemas em produção
- Revisar o que está sendo logado

---

## Serilog — Setup

### Pacotes NuGet

```xml
<PackageReference Include="Serilog.AspNetCore" Version="8.*" />
<PackageReference Include="Serilog.Sinks.Console" Version="5.*" />
<PackageReference Include="Serilog.Sinks.Seq" Version="7.*" />
<PackageReference Include="Serilog.Enrichers.Environment" Version="2.*" />
<PackageReference Include="Serilog.Enrichers.Thread" Version="3.*" />
```

### Program.cs

```csharp
builder.Host.UseSerilog((context, config) => config
    .ReadFrom.Configuration(context.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithThreadId()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {CorrelationId} {Message:lj}{NewLine}{Exception}")
    .WriteTo.Seq("http://localhost:5341") // opcional: Seq para dev
);
```

### appsettings.json

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning"
      }
    }
  }
}
```

---

## Correlation ID

Rastrear uma requisição através de todos os logs e serviços.

### Middleware

```csharp
public sealed class CorrelationIdMiddleware(RequestDelegate next)
{
    private const string Header = "X-Correlation-Id";

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers[Header].FirstOrDefault()
            ?? Guid.CreateVersion7().ToString();

        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers[Header] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await next(context);
        }
    }
}

// Registrar antes de outros middlewares
app.UseMiddleware<CorrelationIdMiddleware>();
```

---

## LoggingBehavior (MediatR Pipeline)

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger
) : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {RequestName}", requestName);

        var stopwatch = Stopwatch.StartNew();
        var response = await next();
        stopwatch.Stop();

        logger.LogInformation(
            "Handled {RequestName} in {ElapsedMs}ms",
            requestName,
            stopwatch.ElapsedMilliseconds
        );

        return response;
    }
}
```

---

## O Que Logar vs Nunca Logar

### Logar (Information)

- Nome do request/command sendo executado
- Tempo de execução de handlers
- IDs de entidades criadas/modificadas
- Erros de negócio (tipo e código)
- Ações de auditoria

### Logar (Warning)

- Rate limit atingido
- Token expirado/inválido
- Tentativas de acesso não autorizado
- Queries lentas (> 500ms)

### Logar (Error)

- Exceções não tratadas
- Falha de conexão com DB/Redis/RabbitMQ
- Timeout em serviços externos

### NUNCA Logar

- Senhas (hash ou plain text)
- Tokens JWT completos
- Dados de cartão de crédito
- CPF, RG, dados pessoais
- Connection strings com senha
- API keys e secrets

---

## Health Checks

### Pacotes

```xml
<PackageReference Include="AspNetCore.HealthChecks.NpgSql" Version="8.*" />
<PackageReference Include="AspNetCore.HealthChecks.Redis" Version="8.*" />
<PackageReference Include="AspNetCore.HealthChecks.RabbitMQ" Version="8.*" />
```

### Configuração

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "postgresql")
    .AddRedis(redisConnection, name: "redis")
    .AddRabbitMQ(rabbitConnectionString, name: "rabbitmq");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // apenas verifica se a app responde
});
```

### Endpoints

| Endpoint | Uso |
|----------|-----|
| `/health` | Status completo (DB, Redis, RabbitMQ) |
| `/health/ready` | Kubernetes readiness probe |
| `/health/live` | Kubernetes liveness probe |

---

## Checklist

- [ ] Serilog configurado com structured logging
- [ ] Correlation ID propagado via middleware e header
- [ ] LoggingBehavior no pipeline MediatR
- [ ] Nível mínimo: Information (Warning para EF Core e ASP.NET)
- [ ] Health checks para PostgreSQL, Redis e RabbitMQ
- [ ] Dados sensíveis NUNCA nos logs
- [ ] Queries lentas logadas como Warning
