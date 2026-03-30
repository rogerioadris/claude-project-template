# CLAUDE.md — Backend

Orientações específicas para o backend. Consulte o `CLAUDE.md` raiz para idioma, stack e regras absolutas.

## Skills do Backend

- `/backend-new-feature` — ao criar novo recurso (entidade + command + query + controller + testes)
- `/backend-architecture` — arquitetura Clean Architecture e CQRS
- `/backend-conventions` — convenções detalhadas (ErrorOr, auditoria, versionamento)
- `/backend-templates` — templates de infraestrutura (ValidationBehavior, DI, Program.cs)

## Comandos Comuns

```bash
# Restaurar e compilar
dotnet restore && dotnet build

# Executar a API
dotnet run --project src/{NomeProjeto}.API

# Modo watch
dotnet watch run --project src/{NomeProjeto}.API

# Testes
dotnet test

# Criar migration
dotnet ef migrations add <Nome> --project src/{NomeProjeto}.Infrastructure --startup-project src/{NomeProjeto}.API

# Aplicar migrations
dotnet ef database update --project src/{NomeProjeto}.Infrastructure --startup-project src/{NomeProjeto}.API

# Docker Compose (infra local)
docker compose up -d
```

## Resumo das Convenções

- **Clean Architecture:** `Domain` → `Application` → `Infrastructure` → `API`
- **CQRS via MediatR:** toda operação passa por `Command` ou `Query`
- **ErrorOr:** handlers retornam `ErrorOr<T>` — nunca use `throw` para erros de negócio
- **Controllers** delegam 100% para `IMediator` — sem lógica de negócio
- **Entidades:** sem setters públicos; toda mutação via métodos; construtor privado para EF Core
- **`long`** para valores monetários — nunca `decimal` ou `float`
- **`DateTime.UtcNow`** sempre — nunca `DateTime.Now`
- **Query Handlers:** `.AsNoTracking()` + projeção `.Select()` para DTO
- Nunca exponha entidades de domínio — mapeie para DTO
- **Commit git** ao final de cada conjunto de alterações (mensagem em pt-br)

> Detalhes completos em [`docs/conventions.md`](docs/conventions.md)

## Pacotes NuGet Principais

```xml
<!-- Application -->
MediatR 12.*, FluentValidation 11.*, ErrorOr 2.*

<!-- Infrastructure -->
Microsoft.EntityFrameworkCore 8.*, Npgsql.EntityFrameworkCore.PostgreSQL 8.*
BCrypt.Net-Next 4.*, Microsoft.AspNetCore.Authentication.JwtBearer 8.*
StackExchange.Redis, MassTransit.RabbitMQ (ou RabbitMQ.Client)

<!-- API -->
Swashbuckle.AspNetCore 6.*

<!-- Tests -->
xUnit, Moq, FluentAssertions
```

## Nomenclatura de Projetos

```
{NomeProjeto}.Domain
{NomeProjeto}.Application
{NomeProjeto}.Infrastructure
{NomeProjeto}.API
{NomeProjeto}.Workers          ← Background Services
{NomeProjeto}.Tests
```
