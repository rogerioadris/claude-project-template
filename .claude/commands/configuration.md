# Configuracao e Secrets

## Objetivo

Padronizar o gerenciamento de configuracoes e secrets no backend (.NET) e frontend (Angular), garantindo seguranca e separacao por ambiente.

## Quando usar

- Adicionar nova configuracao ao projeto
- Configurar secrets para desenvolvimento ou producao
- Criar classes de options fortemente tipadas
- Configurar variaveis de ambiente no Docker ou CI/CD

---

## Options Pattern (.NET)

### Classe de Options

```csharp
// {NomeProjeto}.Infrastructure/Options/JwtOptions.cs
public sealed class JwtOptions
{
    public const string SectionName = "Jwt";

    public required string Issuer { get; init; }
    public required string Audience { get; init; }
    public required string SecretKey { get; init; }
    public int ExpirationMinutes { get; init; } = 60;
    public int RefreshExpirationDays { get; init; } = 7;
}
```

### Registro e Validacao

```csharp
// {NomeProjeto}.API/Extensions/OptionsExtensions.cs
public static IServiceCollection AddOptionsConfiguration(
    this IServiceCollection services,
    IConfiguration configuration)
{
    services.AddOptions<JwtOptions>()
        .Bind(configuration.GetSection(JwtOptions.SectionName))
        .ValidateDataAnnotations()
        .ValidateOnStart(); // Falha no startup se config invalida

    services.AddOptions<RedisOptions>()
        .Bind(configuration.GetSection(RedisOptions.SectionName))
        .ValidateOnStart();

    services.AddOptions<RabbitMqOptions>()
        .Bind(configuration.GetSection(RabbitMqOptions.SectionName))
        .ValidateOnStart();

    return services;
}
```

### Quando usar cada interface

| Interface | Lifetime | Quando usar |
|---|---|---|
| `IOptions<T>` | Singleton | Configuracao que nunca muda em runtime |
| `IOptionsSnapshot<T>` | Scoped | Configuracao que pode mudar entre requests (reloaded on change) |
| `IOptionsMonitor<T>` | Singleton | Singleton que precisa reagir a mudancas de config em runtime |

### Uso nos servicos

```csharp
// IOptions<T> — mais comum
public sealed class TokenService(IOptions<JwtOptions> jwtOptions)
{
    private readonly JwtOptions _jwt = jwtOptions.Value;

    public string GenerateToken(User user)
    {
        // usar _jwt.SecretKey, _jwt.ExpirationMinutes, etc.
    }
}

// IOptionsMonitor<T> — para singletons que precisam de reload
public sealed class CacheService(IOptionsMonitor<RedisOptions> redisOptions)
{
    public void DoWork()
    {
        var current = redisOptions.CurrentValue; // sempre atualizado
    }
}
```

---

## Estrutura de appsettings

### appsettings.json (commitado — valores genericos/vazios)

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "",
    "Redis": ""
  },
  "Jwt": {
    "Issuer": "",
    "Audience": "",
    "SecretKey": "",
    "ExpirationMinutes": 60,
    "RefreshExpirationDays": 7
  },
  "RabbitMq": {
    "Host": "",
    "Username": "",
    "Password": "",
    "VirtualHost": "/"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning"
      }
    }
  }
}
```

### appsettings.Development.json (NUNCA commitado)

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=app_db;Username=app;Password=app_dev_password",
    "Redis": "localhost:6379"
  },
  "Jwt": {
    "Issuer": "app-dev",
    "Audience": "app-dev",
    "SecretKey": "dev-secret-key-min-32-chars-long!!"
  },
  "RabbitMq": {
    "Host": "localhost",
    "Username": "app",
    "Password": "app_dev_password"
  }
}
```

### appsettings.Development.example.json (commitado — template)

Copia identica do `appsettings.Development.json` com valores placeholder. O desenvolvedor copia e preenche.

### appsettings.Production.json (valores via env vars)

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Warning"
    }
  }
}
```

> Em producao, use variaveis de ambiente para tudo sensivel. O ASP.NET Core automaticamente mapeia `ConnectionStrings__DefaultConnection` para `ConnectionStrings:DefaultConnection`.

---

## Secret Management

### Desenvolvimento — User Secrets

```bash
# Inicializar (uma vez por projeto)
dotnet user-secrets init --project src/{NomeProjeto}.API

# Definir um secret
dotnet user-secrets set "Jwt:SecretKey" "minha-chave-secreta-dev-muito-longa" --project src/{NomeProjeto}.API

# Listar secrets
dotnet user-secrets list --project src/{NomeProjeto}.API

# Remover
dotnet user-secrets remove "Jwt:SecretKey" --project src/{NomeProjeto}.API
```

> User Secrets sobrescrevem `appsettings.Development.json`. Ideais para secrets que nao devem existir nem em arquivo local.

### Producao — Variaveis de Ambiente

```bash
# Convencao do ASP.NET Core: __ substitui : na hierarquia
export ConnectionStrings__DefaultConnection="Host=prod-db;..."
export Jwt__SecretKey="producao-chave-segura-256-bits"
export RabbitMq__Host="rabbitmq-prod"
export RabbitMq__Username="app_prod"
export RabbitMq__Password="senha-segura-producao"
```

### Ordem de Precedencia (maior para menor)

1. Variaveis de ambiente
2. User Secrets (apenas em Development)
3. `appsettings.{Environment}.json`
4. `appsettings.json`

---

## Docker — Variaveis de Ambiente

### docker-compose.override.yml (desenvolvimento)

```yaml
services:
  api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Host=postgres;Port=5432;Database=app_db;Username=app;Password=app_dev_password
      - ConnectionStrings__Redis=redis:6379
      - Jwt__SecretKey=dev-secret-key-min-32-chars-long!!
      - RabbitMq__Host=rabbitmq
```

### docker-compose.prod.yml

```yaml
services:
  api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    env_file:
      - .env.production  # arquivo com secrets — NUNCA commitado
```

### .env.production (NUNCA commitado)

```env
ConnectionStrings__DefaultConnection=Host=prod-db;Port=5432;Database=app_db;Username=app_prod;Password=SENHA_SEGURA
Jwt__SecretKey=chave-producao-256-bits-minimo
RabbitMq__Host=rabbitmq-prod
RabbitMq__Password=SENHA_SEGURA_RABBITMQ
```

---

## Angular — Environments

### environment.ts (desenvolvimento)

```typescript
// src/environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:5000/api',
  appName: 'MeuApp (Dev)',
};
```

### environment.prod.ts (producao)

```typescript
// src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: '/api',
  appName: 'MeuApp',
};
```

### Uso nos servicos

```typescript
import { environment } from '../../../environments/environment';

@Injectable({ providedIn: 'root' })
export class ExampleService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = `${environment.apiUrl}/examples`;

  getAll(): Observable<Example[]> {
    return this.http.get<Example[]>(this.apiUrl);
  }
}
```

### Build com environment

```bash
# Desenvolvimento (padrao)
ng serve

# Producao
ng build --configuration=production
```

> O Angular CLI substitui `environment.ts` por `environment.prod.ts` automaticamente no build de producao (configurado em `angular.json`).

---

## .gitignore (obrigatorio)

```gitignore
# Backend
appsettings.Development.json
appsettings.*.local.json
*.pfx

# Docker
.env.production
.env.staging
.env.local

# Nunca commitar
**/secrets/
```

---

## Checklist

- [ ] Classes de Options criadas para cada secao de configuracao
- [ ] `ValidateOnStart()` em todas as options obrigatorias
- [ ] `appsettings.json` commitado com valores vazios/genericos
- [ ] `appsettings.Development.json` no `.gitignore`
- [ ] `appsettings.Development.example.json` commitado como template
- [ ] User Secrets configurado para desenvolvimento
- [ ] Variaveis de ambiente usadas em producao — nunca secrets em arquivo commitado
- [ ] `.env.production` no `.gitignore`
- [ ] `environment.ts` e `environment.prod.ts` configurados no Angular
- [ ] Ordem de precedencia de configuracao entendida pela equipe
- [ ] Nenhum secret hardcoded no codigo-fonte
