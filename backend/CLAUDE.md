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

## Resumo das Convenções (específicas do backend)

- **Clean Architecture:** `Domain` → `Application` → `Infrastructure` → `API`
- **CQRS via MediatR:** toda operação passa por `Command` ou `Query`
- **Controllers** delegam 100% para `IMediator` — sem lógica de negócio
- **Query Handlers:** `.AsNoTracking()` + projeção `.Select()` para DTO
- Nunca exponha entidades de domínio — mapeie para DTO
- Construtor `private` para EF Core em todas as entidades

> Detalhes completos via `/backend-conventions`

> Pacotes NuGet e nomenclatura de projetos disponíveis via `/backend-templates`
