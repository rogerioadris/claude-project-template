# Criando um Novo Recurso — Backend

Ao adicionar qualquer novo recurso ao projeto, siga **sempre** esta sequência.
Não pule etapas nem inverta a ordem.

---

## 1 — Domain

Crie a entidade em `{NomeProjeto}.Domain/Entities/{Feature}.cs`

```csharp
// {NomeProjeto}.Domain/Entities/Example.cs
public sealed class Example
{
    // Construtor privado para uso exclusivo do EF Core
    private Example() { }

    // Construtor público para criação via domínio
    public Example(string name, string description)
    {
        Id          = Guid.NewGuid();
        Name        = name;
        Description = description;
        CreatedAt   = DateTime.UtcNow; // sempre UtcNow, nunca DateTime.Now
    }

    public Guid     Id          { get; private set; }
    public string   Name        { get; private set; } = default!;
    public string   Description { get; private set; } = default!;
    public DateTime CreatedAt   { get; private set; }
    public DateTime? UpdatedAt  { get; private set; }

    // Toda mutação via método público — sem setters públicos
    public void UpdateName(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new Exceptions.DomainException("Nome não pode ser vazio.");

        Name      = name;
        UpdatedAt = DateTime.UtcNow;
    }
}
```

---

## 2 — Interface do Repositório

Crie `I{Feature}Repository.cs` em `{NomeProjeto}.Application/Common/Interfaces/`

---

## 3 — Command (escrita)

Crie pasta `{NomeProjeto}.Application/Features/{Feature}/Commands/{Action}{Feature}/`

- Se o endpoint exige usuário autenticado, implemente também `IAuthorizedRequest`
- Se o endpoint exige permissão granular, implemente também `IPermissionRequiredRequest` retornando a constante via `AppPermissions.*` — sempre junto com `IAuthorizedRequest`

### Interfaces de Pipeline

```csharp
// Endpoint público
public sealed record LoginUserCommand(string Email, string Password)
    : IRequest<ErrorOr<LoginUserResult>>;

// Exige autenticação
public sealed record GetUserByIdQuery(Guid Id)
    : IRequest<ErrorOr<UserDto>>, IAuthorizedRequest;

// Exige autenticação + permissão granular via role
public sealed record DeleteUserCommand(Guid Id)
    : IRequest<ErrorOr<Deleted>>, IAuthorizedRequest, IPermissionRequiredRequest
{
    public string RequiredPermission => AppPermissions.UsersDelete;
}
```

### Command

```csharp
// {Action}{Feature}Command.cs
public sealed record CreateExampleCommand(
    string Name,
    string Description
) : IRequest<ErrorOr<Guid>>;
```

### Handler

```csharp
// {Action}{Feature}CommandHandler.cs
public sealed class CreateExampleCommandHandler(
    IExampleRepository exampleRepository,
    IAuditService      auditService,
    IUnitOfWork        unitOfWork
) : IRequestHandler<CreateExampleCommand, ErrorOr<Guid>>
{
    public async Task<ErrorOr<Guid>> Handle(CreateExampleCommand request, CancellationToken cancellationToken)
    {
        var exists = await exampleRepository.ExistsByNameAsync(request.Name, cancellationToken);
        if (exists)
            return Error.Conflict("Example.Duplicate", $"'{request.Name}' já existe."); // nunca throw

        var example = new Example(request.Name, request.Description);
        await exampleRepository.AddAsync(example, cancellationToken);
        await unitOfWork.SaveChangesAsync(cancellationToken); // save primeiro

        await auditService.LogAsync(                          // auditoria após save
            action:    "CREATE",
            tableName: "Examples",
            entityId:  example.Id.ToString(),
            newValues: JsonSerializer.Serialize(new { example.Name }),
            cancellationToken: cancellationToken);

        return example.Id;
    }
}
```

### Validator

```csharp
// {Action}{Feature}CommandValidator.cs
public sealed class CreateExampleCommandValidator : AbstractValidator<CreateExampleCommand>
{
    public CreateExampleCommandValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(150);
        RuleFor(x => x.Description).NotEmpty().MaximumLength(300);
    }
}
```

---

## 4 — Query (leitura)

Crie pasta `{NomeProjeto}.Application/Features/{Feature}/Queries/Get{Feature}ById/`

- Se o endpoint exige usuário autenticado, implemente também `IAuthorizedRequest`
- Se o endpoint exige permissão granular, implemente também `IPermissionRequiredRequest`

### Query + DTO

```csharp
// Get{Feature}ByIdQuery.cs
public sealed record GetExampleByIdQuery(Guid Id) : IRequest<ErrorOr<ExampleDto>>;

// ExampleDto.cs
public sealed record ExampleDto(Guid Id, string Name, string Description, DateTime CreatedAt);
```

### Query Handler

```csharp
// Get{Feature}ByIdQueryHandler.cs
public sealed class GetExampleByIdQueryHandler(IExampleRepository exampleRepository)
    : IRequestHandler<GetExampleByIdQuery, ErrorOr<ExampleDto>>
{
    public async Task<ErrorOr<ExampleDto>> Handle(GetExampleByIdQuery request, CancellationToken cancellationToken)
    {
        var example = await exampleRepository.GetByIdAsync(request.Id, cancellationToken); // .AsNoTracking() no repositório

        if (example is null)
            return Error.NotFound("Example.NotFound", $"'{request.Id}' não encontrado.");

        return new ExampleDto(example.Id, example.Name, example.Description, example.CreatedAt);
    }
}
```

### Query com Paginação

```csharp
public sealed record GetAllExamplesQuery(
    int     Page     = 1,
    int     PageSize = 20,
    string? Search   = null
) : IRequest<ErrorOr<PagedResult<ExampleDto>>>;
```

---

## 5 — Repositório

Implemente `{Feature}Repository.cs` em `{NomeProjeto}.Infrastructure/Repositories/`

Registre em `{NomeProjeto}.Infrastructure/DependencyInjection.cs`:

```csharp
services.AddScoped<I{Feature}Repository, {Feature}Repository>();
```

---

## 6 — Configuração EF Core

Crie `{Feature}Configuration.cs` em `{NomeProjeto}.Infrastructure/Data/Configurations/` implementando `IEntityTypeConfiguration<{Feature}>`

O `AppDbContext` descobre automaticamente via `ApplyConfigurationsFromAssembly`.

---

## 7 — Migration

```bash
dotnet ef migrations add Add{Feature} \
  --project {NomeProjeto}.Infrastructure \
  --startup-project {NomeProjeto}.API
dotnet ef database update \
  --project {NomeProjeto}.Infrastructure \
  --startup-project {NomeProjeto}.API
```

---

## 8 — Controller

Crie `{Feature}sController.cs` em `{NomeProjeto}.API/Controllers/`

```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize(Policy = AppPolicies.CustomerOnly)]
public sealed class ExamplesController(IMediator mediator) : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAll(
        [FromQuery] int page = 1, [FromQuery] int pageSize = 20,
        [FromQuery] string? search = null, CancellationToken cancellationToken = default)
    {
        var result = await mediator.Send(new GetAllExamplesQuery(page, pageSize, search), cancellationToken);
        return result.ToActionResult(this, Ok);
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var result = await mediator.Send(new GetExampleByIdQuery(id), cancellationToken);
        return result.ToActionResult(this, Ok);
    }

    [HttpPost]
    [Authorize(Policy = AppPolicies.AdminOnly)]
    public async Task<IActionResult> Create([FromBody] CreateExampleCommand command, CancellationToken cancellationToken)
    {
        var result = await mediator.Send(command, cancellationToken);
        return result.ToActionResult(this, id => CreatedAtAction(nameof(GetById), new { id }, new { id }));
    }

    [HttpDelete("{id:guid}")]
    [Authorize(Policy = AppPolicies.AdminOnly)]
    public async Task<IActionResult> Delete(Guid id, CancellationToken cancellationToken)
    {
        var result = await mediator.Send(new DeleteExampleCommand(id), cancellationToken);
        return result.ToActionResult(this, _ => NoContent());
    }
}
```

- Actions com no máximo 5–10 linhas
- Delegar 100% para `IMediator`
- Converter resultado via `.ToActionResult(this, Ok)`
- Aplique `[Authorize(Policy = AppPolicies.XXX)]` conforme necessidade

---

## 9 — Testes

- Crie `{NomeProjeto}.Tests/Unit/Features/{Feature}/{Action}{Feature}CommandHandlerTests.cs`
- Crie `{NomeProjeto}.Tests/Unit/Features/{Feature}/Get{Feature}ByIdQueryHandlerTests.cs`
- Crie `{NomeProjeto}.Tests/Integration/Controllers/{Feature}sControllerTests.cs`
  - Use `WebApplicationFactory<Program>` com banco em memória ou container de teste
  - Cubra os fluxos: sucesso, não encontrado, não autorizado, conflito
- Execute: `dotnet test --filter "FullyQualifiedName~{Feature}"`

---

## Checklist Rápido

- [ ] Entidade criada com construtor privado e sem setters públicos
- [ ] Interface de repositório em `Application/Common/Interfaces/`
- [ ] Command + Handler + Validator criados
- [ ] Query + Handler + DTO criados
- [ ] Handler retorna `ErrorOr<T>` — nenhum `throw` para erros de negócio
- [ ] `SaveChangesAsync` chamado antes de `IAuditService.LogAsync`
- [ ] Repositório implementado e registrado no DI
- [ ] Configuração EF Core criada em `Data/Configurations/`
- [ ] Migration criada e aplicada
- [ ] Controller criado com `[Authorize(Policy = ...)]` apropriado
- [ ] Entidade nunca exposta na response — sempre DTO
- [ ] `IAuthorizedRequest` implementado nos Commands/Queries que exigem usuário autenticado
- [ ] `IPermissionRequiredRequest` implementado quando a operação exige permissão granular
- [ ] Permissão referenciada via `AppPermissions.*` — nunca string literal
- [ ] `.AsNoTracking()` nas queries de leitura
- [ ] Testes unitários para Handler de Command e Query
- [ ] Testes de integração para o Controller (sucesso, 404, 401, 409)
