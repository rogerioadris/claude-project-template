# Templates de Infraestrutura — Backend

Templates de configuração e pipeline do projeto. Para templates de features (Entity, Command, Query, Controller), use `/backend-new-feature`.

---

## Interfaces de Pipeline

Combinações de interfaces que determinam o comportamento do pipeline para cada Command/Query:

| Interfaces implementadas | Comportamento |
|---|---|
| Nenhuma | Endpoint público (ex: login, refresh) |
| `IAuthorizedRequest` | Exige usuário autenticado |
| `IAuthorizedRequest` + `IPermissionRequiredRequest` | Exige autenticação + permissão via role (respeitando negações individuais) |

> Permissões vêm exclusivamente dos roles. `UserPermissions` registra apenas negações.
> Use sempre constantes via `AppPermissions.*` — nunca strings literais.

---

## ValidationBehavior

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators
) : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!validators.Any()) return await next();

        var failures = validators
            .Select(v => v.Validate(new ValidationContext<TRequest>(request)))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Count == 0) return await next();

        var errors = failures
            .Select(f => Error.Validation(f.PropertyName, f.ErrorMessage))
            .ToList();

        return (TResponse)(dynamic)errors; // conversão implícita List<Error> → ErrorOr<T>
    }
}
```

---

## ErrorOrExtensions

```csharp
// {NomeProjeto}.API/Extensions/ErrorOrExtensions.cs
public static class ErrorOrExtensions
{
    public static IActionResult ToActionResult<T>(
        this ErrorOr<T>        result,
        ControllerBase         controller,
        Func<T, IActionResult> onSuccess)
        => result.IsError
            ? controller.MapErrors(result.Errors)
            : onSuccess(result.Value);

    private static IActionResult MapErrors(this ControllerBase controller, IReadOnlyList<Error> errors)
    {
        var first      = errors[0];
        var statusCode = first.Type switch
        {
            ErrorType.NotFound     => StatusCodes.Status404NotFound,
            ErrorType.Unauthorized => StatusCodes.Status401Unauthorized,
            ErrorType.Forbidden    => StatusCodes.Status403Forbidden,
            ErrorType.Conflict     => StatusCodes.Status409Conflict,
            ErrorType.Validation   => StatusCodes.Status422UnprocessableEntity,
            _                      => StatusCodes.Status500InternalServerError,
        };
        return controller.Problem(statusCode: statusCode, title: first.Code, detail: first.Description);
    }
}
```

---

## DependencyInjection (Application)

```csharp
public static IServiceCollection AddApplicationServices(this IServiceCollection services)
{
    services.AddMediatR(cfg =>
    {
        cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
        // Ordem importa: Logging → Authorization → Permission → Validation → Handler
        cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));
        cfg.AddOpenBehavior(typeof(AuthorizationBehavior<,>));
        cfg.AddOpenBehavior(typeof(PermissionBehavior<,>));
        cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
    });
    services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
    return services;
}
```

---

## Program.cs

`Program.cs` deve conter **apenas chamadas de alto nível** — toda a lógica de configuração fica em métodos de extensão em `{NomeProjeto}.API/Extensions/`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.AddApiServices();
var app = builder.Build();
await app.InitializeDatabaseAsync();
app.UseApiPipeline();
app.Run();
```
