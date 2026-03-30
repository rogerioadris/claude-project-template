# Arquitetura do Backend

## Stack

| Camada | Tecnologia | Versão |
|--------|-----------|--------|
| API | ASP.NET Core Web API | .NET 9 |
| Workers | .NET Background Services | .NET 9 |
| Mediator/CQRS | MediatR | 12.x |
| Resultado tipado | ErrorOr | 2.x |
| Validação | FluentValidation + MediatR Pipeline | 11.x |
| ORM | Entity Framework Core | 8.x |
| Banco | PostgreSQL | 16+ |
| Cache / Lock | Redis (StackExchange.Redis) | — |
| Mensageria | RabbitMQ | — |
| Autenticação | JWT Bearer + BCrypt (custom) | — |
| Testes | xUnit + Moq + FluentAssertions | — |

---

## Estrutura de Projetos

```
{NomeProjeto}.sln
├── src/
│   ├── {NomeProjeto}.Domain/          ← Entidades, enums, value objects, exceções
│   ├── {NomeProjeto}.Application/     ← Commands, Queries, Handlers, Behaviors, Interfaces
│   ├── {NomeProjeto}.Infrastructure/  ← EF Core, Repositórios, Redis, RabbitMQ, JWT, BCrypt
│   ├── {NomeProjeto}.API/             ← Controllers, Middleware, Program.cs
│   └── {NomeProjeto}.Workers/         ← Background Services
└── tests/
    └── {NomeProjeto}.Tests/           ← xUnit: unit + integration
```

---

## Estrutura de Pastas — Domain

```
{NomeProjeto}.Domain/
├── Entities/
│   └── {Entidade}.cs
├── ValueObjects/
│   └── {ValueObject}.cs
├── Enums/
│   └── {Enum}.cs
├── Constants/
│   ├── AppRoles.cs
│   ├── AppPolicies.cs
│   └── SeedIds.cs        ← UUIDs fixos dos seeds como constantes
└── Exceptions/
    └── DomainException.cs
```

---

## Estrutura de Pastas — Application

```
{NomeProjeto}.Application/
├── Common/
│   ├── Behaviors/
│   │   ├── LoggingBehavior.cs
│   │   ├── AuthorizationBehavior.cs
│   │   └── ValidationBehavior.cs
│   ├── Interfaces/
│   │   ├── IUnitOfWork.cs
│   │   ├── IPasswordHasher.cs
│   │   ├── IJwtService.cs
│   │   ├── IAuditService.cs
│   │   ├── IMessageBus.cs
│   │   ├── IDistributedLockProvider.cs
│   │   └── I{Feature}Repository.cs
│   ├── Services/
│   │   └── {DomainService}.cs
│   └── PagedResult.cs
└── Features/
    ├── Auth/Commands/
    └── {Feature}/Commands|Queries/
```

---

## Fluxo CQRS via MediatR

```
Controller
   │
   └─► IMediator.Send(command/query)
              │
              ▼
   [Pipeline Behaviors — em ordem]
   1. LoggingBehavior        → registra entrada/saída e tempo
   2. AuthorizationBehavior  → valida JWT e papel do usuário
   3. ValidationBehavior     → valida via FluentValidation
              │
              ▼
         Handler (lógica de negócio)
              │
              ├── Repositórios
              ├── IUnitOfWork    → persiste via EF Core
              ├── IAuditService  → registra em AuditLogs
              └── IMessageBus   → publica em RabbitMQ (quando necessário)
```

**Regras:**
- Controllers delegam 100% para `IMediator.Send()` — sem lógica de negócio
- Commands retornam `ErrorOr<Guid>` ou `ErrorOr<Unit>`
- Queries retornam `ErrorOr<TDto>` ou `ErrorOr<PagedResult<T>>`
- Handlers nunca lançam exceções para erros de negócio — usam `ErrorOr`
- Nunca injete `IMediator` dentro de um Handler

---

## Responsabilidade por Camada

| Camada | Responsabilidade |
|--------|-----------------|
| `Domain` | Entidades, enums, value objects, constantes, exceções de domínio |
| `Application` | Casos de uso, commands, queries, handlers, behaviors, serviços de domínio |
| `Infrastructure` | EF Core, PostgreSQL, Redis, RabbitMQ, JWT, BCrypt, repositórios |
| `API` | Controllers, middleware, configuração ASP.NET Core |
| `Workers` | Background Services para processamento assíncrono |
