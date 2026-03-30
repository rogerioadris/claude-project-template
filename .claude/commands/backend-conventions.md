# Convenções do Backend

## Entidades de Domínio

- Construtor sem parâmetros `private` para uso exclusivo do EF Core.
- **Nenhum setter público** — toda mutação via métodos públicos (ex: `user.UpdateName(...)`).
- Comportamento de negócio vive na entidade, não em serviços.
- Use `DateTime.UtcNow` sempre — **nunca** `DateTime.Now`.

## C# e .NET

- **Nullable obrigatório**: todos os `.csproj` devem ter `<Nullable>enable</Nullable>` — trate nulos explicitamente.
- **Primary constructor** (C# 12) nos Handlers para injeção de dependência.
- **JSON**: configure `JsonNamingPolicy.CamelCase` globalmente — todas as respostas da API devem usar camelCase.
- **Argumentos de método**: sempre quebrar linha — um argumento por linha, com `)` em nova linha. Aplica-se a invocações e declarações (construtores, métodos, log, etc).

## ErrorOr

- Handlers retornam `ErrorOr<T>` — **nunca lance exceções** para erros de negócio.
- Use os helpers semânticos para criar erros:
  - `Error.Validation("Campo", "Mensagem")` — falha de validação de negócio
  - `Error.NotFound("Entidade", "Mensagem")` — recurso não encontrado
  - `Error.Conflict("Campo", "Mensagem")` — conflito de estado (ex: duplicata)
  - `Error.Failure("Código", "Mensagem")` — falha genérica
- Controllers consomem o resultado via `.ToActionResult()` (extensão em `{NomeProjeto}.API/Extensions/ErrorOrExtensions.cs`).
- Exceções lançadas por infraestrutura (EF Core) continuam sendo capturadas pelo middleware global de exceções.

## MediatR / CQRS

- `Command` e `Query` são sempre `sealed record` implementando `IRequest<ErrorOr<T>>`.
- Nome de Command no imperativo: `CreateUserCommand`, `DeleteUserCommand`.
- Nome de Query descritivo: `GetUserByIdQuery`, `GetAllUsersQuery`.
- **Um Handler por arquivo**, classe `sealed` — nunca agrupe múltiplos Handlers.
- **Nunca reutilize Handlers** entre features.
- **Nunca injete `IMediator` dentro de um Handler** — apenas repositórios e serviços.
- Cada Query tem seu próprio DTO — evite DTOs "universais".
- Pipeline Behaviors são a única exceção cross-cutting (logging, validação).
- **Query Handlers**: use `.AsNoTracking()` em todas as consultas de leitura; projete para DTO via `.Select()` — nunca retorne entidades do handler.

## Auditoria

- Todo CREATE, UPDATE e DELETE em entidades relevantes deve chamar `IAuditService.LogAsync(...)`.
- Ações padronizadas: `"CREATE"`, `"UPDATE"`, `"DELETE"`, `"LOGIN"`, `"LOGOUT"`, `"DENY_PERMISSION"`, `"RESTORE_PERMISSION"`.
- `oldValues` e `newValues` devem ser serializados como JSON via `System.Text.Json.JsonSerializer.Serialize(...)`.
- Nunca chame `IAuditService` antes de `SaveChangesAsync` — registre a auditoria após confirmar a operação.

## Controllers e API

- Actions com máximo 5–10 linhas: apenas receber, despachar via `IMediator`, responder.
- Nunca acesse `DbContext` diretamente no Handler — use repositório ou `IUnitOfWork`.
- Resultados `ErrorOr<T>` são convertidos para HTTP via `.ToActionResult(this, Ok)` — erros mapeiam automaticamente para `Problem()` com status HTTP semântico (404, 403, 409, 422, 401, 500).
- Nunca exponha entidades de domínio — sempre mapeie para DTO antes de retornar.
- Nunca commite segredos — gere `appsettings.Development.example.json` para configs sensíveis.

## Versionamento de API

- Todos os endpoints utilizam o prefixo `/v1/` na URL.
- Breaking changes exigem criação de uma nova versão (`/v2/`, `/v3/`, etc.).
- A versão anterior deve ser mantida ativa por **12 meses** após a publicação da nova.
- Endpoints depreciados devem ser marcados com `deprecated: true` no Swagger/OpenAPI.

### O que é Breaking Change

- Remoção ou renomeação de campo no response
- Alteração de tipo de dado de um campo existente
- Remoção ou renomeação de endpoint
- Mudança de método HTTP de um endpoint
- Alteração de campo obrigatório no request body

### O que NÃO é Breaking Change (não exige nova versão)

- Adição de novos campos opcionais no response
- Adição de novos endpoints
- Adição de novos valores em enums (desde que o cliente ignore valores desconhecidos)
- Melhorias de performance sem alteração de contrato

### Convenções

- Prefixo de versão sempre na URL: `/v1/`, `/v2/` — nunca versionar por header ou query param.
- Documentar no Swagger a data prevista de descontinuação ao depreciar um endpoint.

```
# Versão atual
GET /v1/users/{id}

# Após breaking change
GET /v2/users/{id}   ← nova versão
GET /v1/users/{id}   ← mantida por 12 meses, marcada como deprecated
```

## Nomenclatura de Arquivos

| Tipo | Padrão | Exemplo |
|------|--------|---------|
| Entidade | `PascalCase.cs` | `User.cs` |
| Command | `{Action}{Feature}Command.cs` | `CreateUserCommand.cs` |
| Handler | `{Action}{Feature}CommandHandler.cs` | `CreateUserCommandHandler.cs` |
| Validator | `{Action}{Feature}CommandValidator.cs` | `CreateUserCommandValidator.cs` |
| Query | `Get{Feature}By{Criteria}Query.cs` | `GetUserByIdQuery.cs` |
| DTO | `{Feature}Dto.cs` / `{Feature}SummaryDto.cs` | `UserDto.cs` |
| Repositório | `I{Feature}Repository.cs` | `IUserRepository.cs` |
| Controller | `{Feature}sController.cs` | `UsersController.cs` |
| Teste | `{Class}Tests.cs` | `CreateUserCommandHandlerTests.cs` |

## Pacotes NuGet Obrigatórios

```xml
<!-- Application -->
<PackageReference Include="MediatR" Version="12.*" />
<PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="11.*" />
<PackageReference Include="ErrorOr" Version="2.*" />

<!-- Infrastructure -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.*" />
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.*" />
<PackageReference Include="BCrypt.Net-Next" Version="4.*" />
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.*" />

<!-- API -->
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.*" />
```
