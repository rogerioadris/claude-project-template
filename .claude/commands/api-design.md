# Design de API REST

## Objetivo

Convenções para design consistente de endpoints, paginação, versionamento e contratos de erro.

## Quando usar

- Criar novos endpoints
- Revisar consistência da API
- Definir contrato de paginação ou filtros

---

## Nomenclatura de Endpoints

```
GET    /api/v1/{resources}          → Listar (paginado)
GET    /api/v1/{resources}/{id}     → Detalhar
POST   /api/v1/{resources}          → Criar
PUT    /api/v1/{resources}/{id}     → Atualizar (completo)
PATCH  /api/v1/{resources}/{id}     → Atualizar (parcial)
DELETE /api/v1/{resources}/{id}     → Remover
```

### Regras

- Plural sempre: `/users`, `/transactions` — nunca `/user`
- Kebab-case para compostos: `/payment-methods` — nunca camelCase na URL
- Sem verbos na URL: `/users` — nunca `/getUsers` ou `/createUser`
- Recursos aninhados: `/users/{userId}/accounts` (máximo 2 níveis)

---

## Verbos HTTP e Status Codes

| Verbo | Sucesso | Corpo da resposta |
|-------|---------|-------------------|
| GET (lista) | 200 OK | `PagedResult<T>` |
| GET (detalhe) | 200 OK | `T` |
| POST | 201 Created | `{ id }` + header `Location` |
| PUT/PATCH | 200 OK | `T` atualizado |
| DELETE | 204 No Content | Vazio |

### Status de Erro

| Status | Quando usar | ErrorOr |
|--------|-------------|---------|
| 400 | Request malformado (JSON inválido) | Middleware |
| 401 | Não autenticado | `Error.Unauthorized` |
| 403 | Sem permissão | `Error.Forbidden` |
| 404 | Recurso não encontrado | `Error.NotFound` |
| 409 | Conflito (duplicata) | `Error.Conflict` |
| 422 | Validação de negócio falhou | `Error.Validation` |
| 429 | Rate limit excedido | Middleware |
| 500 | Erro interno | Middleware global |

---

## Paginação

### Request

```
GET /api/v1/users?page=1&pageSize=20&search=joao&sortBy=name&sortDir=asc
```

| Param | Tipo | Default | Máximo |
|-------|------|---------|--------|
| `page` | int | 1 | — |
| `pageSize` | int | 20 | 100 |
| `search` | string? | null | — |
| `sortBy` | string? | null | — |
| `sortDir` | string? | "asc" | — |

### Response (PagedResult<T>)

```json
{
  "data": [...],
  "page": 1,
  "pageSize": 20,
  "totalCount": 150,
  "totalPages": 8,
  "hasNextPage": true,
  "hasPreviousPage": false
}
```

### Query com Paginação

```csharp
public sealed record GetAllUsersQuery(
    int     Page     = 1,
    int     PageSize = 20,
    string? Search   = null,
    string? SortBy   = null,
    string? SortDir  = "asc"
) : IRequest<ErrorOr<PagedResult<UserDto>>>;
```

---

## Filtros

### Padrão para filtros múltiplos

```
GET /api/v1/transactions?status=completed&dateFrom=2026-01-01&dateTo=2026-03-31&minAmount=10000
```

- Valores monetários em centavos na query string (consistente com o domínio)
- Datas em ISO 8601: `2026-01-01`
- Enums como string: `status=completed`

---

## Versionamento

- Prefixo na URL: `/api/v1/`, `/api/v2/`
- Nunca por header ou query param
- Breaking change → nova versão
- Versão anterior mantida por 12 meses

### O que é Breaking Change

- Remover/renomear campo no response
- Alterar tipo de dado
- Remover/renomear endpoint
- Mudar método HTTP
- Tornar campo obrigatório no request

### O que NÃO é Breaking Change

- Adicionar campos opcionais no response
- Adicionar novos endpoints
- Adicionar valores em enums

---

## Contratos de Erro (ProblemDetails)

Todas as respostas de erro seguem RFC 7807:

```json
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "Example.NotFound",
  "status": 404,
  "detail": "'550e8400-...' não encontrado.",
  "traceId": "00-abc123..."
}
```

Mapeamento automático via `ErrorOrExtensions.ToActionResult()`.

---

## Swagger/OpenAPI

```csharp
// Anotar endpoints no Controller
[HttpGet]
[ProducesResponseType(typeof(PagedResult<UserDto>), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status401Unauthorized)]
public async Task<IActionResult> GetAll(...)

[HttpPost]
[ProducesResponseType(StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status409Conflict)]
[ProducesResponseType(StatusCodes.Status422UnprocessableEntity)]
public async Task<IActionResult> Create(...)
```

- Documentar todos os status codes possíveis
- Usar `[Produces("application/json")]` no Controller
- Marcar endpoints deprecated com `[Obsolete]` + Swagger annotation

---

## Rate Limiting (.NET 9)

```csharp
// Program.cs
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
    });

    options.AddSlidingWindowLimiter("login", opt =>
    {
        opt.PermitLimit = 5;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 2;
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

// Controller
[EnableRateLimiting("login")]
[HttpPost("login")]
public async Task<IActionResult> Login(...)
```

- `fixed`: limite geral para endpoints públicos (100 req/min)
- `login`: limite restritivo para autenticação (5 req/min com janela deslizante)
- Combine com Redis para cenários distribuídos (múltiplas instâncias)

---

## API Versioning com Asp.Versioning

```csharp
// NuGet: Asp.Versioning.Http, Asp.Versioning.Mvc.ApiExplorer
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
}).AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Controller
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public sealed class UsersController : ControllerBase { ... }
```

- Sempre via URL prefix (`/api/v1/`) — nunca header ou query param
- `ReportApiVersions = true` adiciona header `api-supported-versions` nas respostas
- `SubstituteApiVersionInUrl = true` permite que o Swagger gere URLs corretas

---

## Checklist

- [ ] Endpoints em plural, kebab-case, sem verbos
- [ ] Versionamento via `/api/v1/`
- [ ] Paginação com `PagedResult<T>` padronizado
- [ ] Status codes corretos (201 para POST, 204 para DELETE)
- [ ] ProblemDetails para todos os erros
- [ ] Swagger com `[ProducesResponseType]` em todas as actions
- [ ] Valores monetários em centavos (long) inclusive na API
- [ ] Datas em UTC (ISO 8601)
- [ ] Rate limiting configurado em endpoints públicos e de login
- [ ] API versioning via URL prefix (`/api/v1/`)
