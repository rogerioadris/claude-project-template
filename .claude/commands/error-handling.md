# Tratamento de Erros

## Objetivo

Padronizar o tratamento de erros na aplicação: erros de negocio via ErrorOr, exceções de infraestrutura via middleware global, e interceptação de erros HTTP no frontend Angular.

## Quando usar

- Configurar tratamento global de exceções
- Mapear ErrorOr para respostas HTTP
- Criar interceptor de erros no Angular
- Distinguir erro de negócio vs erro de infraestrutura

---

## Regra Fundamental

| Tipo de erro | Mecanismo | Retry? |
|---|---|---|
| Erro de negócio | `ErrorOr<T>` — retorna `Error` no Handler | Nunca |
| Erro de infraestrutura | `try/catch` + logging + throw | Sim, com backoff |
| Erro de validação | `FluentValidation` via `ValidationBehavior` | Nunca |

> **Nunca lance exceção para erros de negócio** — use `ErrorOr`. Exceções são para falhas inesperadas (DB fora, timeout, rede).

---

## Global Exception Handler Middleware

```csharp
// {NomeProjeto}.API/Middlewares/GlobalExceptionHandlerMiddleware.cs
public sealed class GlobalExceptionHandlerMiddleware(
    RequestDelegate next,
    ILogger<GlobalExceptionHandlerMiddleware> logger
)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (Exception ex)
        {
            var correlationId = context.Items["CorrelationId"]?.ToString() ?? "N/A";

            logger.LogError(ex,
                "Exceção não tratada | CorrelationId: {CorrelationId} | Path: {Path}",
                correlationId, context.Request.Path);

            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/problem+json";

            var problem = new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "Erro interno do servidor",
                Detail = "Ocorreu um erro inesperado. Tente novamente mais tarde.",
                Instance = context.Request.Path,
                Extensions = { ["correlationId"] = correlationId }
            };

            await context.Response.WriteAsJsonAsync(problem);
        }
    }
}
```

### Registro

```csharp
// Registrar APÓS o CorrelationIdMiddleware e ANTES de UseRouting
app.UseMiddleware<CorrelationIdMiddleware>();
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
app.UseRouting();
```

---

## ProblemDetails Factory

```csharp
// {NomeProjeto}.API/Extensions/ProblemDetailsExtensions.cs
public static IServiceCollection AddProblemDetailsFactory(this IServiceCollection services)
{
    services.AddProblemDetails(options =>
    {
        options.CustomizeProblemDetails = context =>
        {
            context.ProblemDetails.Extensions["correlationId"] =
                context.HttpContext.Items["CorrelationId"]?.ToString();

            // Nunca expor stack trace em produção
            context.ProblemDetails.Extensions.Remove("exception");
        };
    });

    return services;
}
```

---

## ErrorOr para HTTP — ToActionResult

Extensão centralizada que mapeia `ErrorOr<T>` para `IActionResult`. Referência ao padrão existente:

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

        // Múltiplos erros de validação: retorna todos
        if (first.Type == ErrorType.Validation && errors.Count > 1)
        {
            var validationErrors = errors
                .ToDictionary(e => e.Code, e => new[] { e.Description });

            return new UnprocessableEntityObjectResult(
                new ValidationProblemDetails(validationErrors)
                {
                    Status = StatusCodes.Status422UnprocessableEntity
                });
        }

        return controller.Problem(
            statusCode: statusCode,
            title: first.Code,
            detail: first.Description);
    }
}
```

### Uso no Controller

```csharp
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateExampleCommand command)
{
    var result = await mediator.Send(command);

    return result.ToActionResult(this, value =>
        CreatedAtAction(nameof(GetById), new { id = value.Id }, value));
}
```

---

## Exception Filters para Cenários Específicos

Para exceções conhecidas que precisam de tratamento diferenciado (ex: concorrência):

```csharp
// {NomeProjeto}.API/Filters/DbConcurrencyExceptionFilter.cs
public sealed class DbConcurrencyExceptionFilter(
    ILogger<DbConcurrencyExceptionFilter> logger
) : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        if (context.Exception is not DbUpdateConcurrencyException ex) return;

        logger.LogWarning(ex, "Conflito de concorrência detectado");

        context.Result = new ConflictObjectResult(new ProblemDetails
        {
            Status = StatusCodes.Status409Conflict,
            Title = "Conflito de concorrência",
            Detail = "O registro foi modificado por outro usuário. Recarregue e tente novamente."
        });

        context.ExceptionHandled = true;
    }
}

// Registro
builder.Services.AddControllers(options =>
{
    options.Filters.Add<DbConcurrencyExceptionFilter>();
});
```

---

## Erros de Negócio no Handler (ErrorOr)

```csharp
// No Handler — NUNCA lance exceção para erro de negócio
public async Task<ErrorOr<PaymentResponse>> Handle(
    ProcessPaymentCommand request,
    CancellationToken cancellationToken)
{
    var customer = await customerRepository.GetByIdAsync(request.CustomerId, cancellationToken);
    if (customer is null)
        return Error.NotFound("Customer.NotFound", "Cliente não encontrado.");

    if (customer.IsBlocked)
        return Error.Forbidden("Customer.Blocked", "Cliente bloqueado para operações.");

    if (request.AmountInCents <= 0)
        return Error.Validation("Payment.InvalidAmount", "O valor deve ser maior que zero.");

    // Erro de infraestrutura — aqui SIM pode lançar exceção (DB fora, timeout)
    // O middleware global captura
    var payment = customer.CreatePayment(request.AmountInCents);
    await paymentRepository.AddAsync(payment, cancellationToken);
    await unitOfWork.SaveChangesAsync(cancellationToken);

    return payment.ToResponse();
}
```

---

## Logging de Erros com Serilog

### Structured logging com correlation ID

```csharp
// Erros de negócio — LogInformation (não é erro de sistema)
logger.LogInformation(
    "Erro de negócio: {ErrorCode} — {ErrorDescription} | CustomerId: {CustomerId}",
    error.Code, error.Description, request.CustomerId);

// Erros de infraestrutura — LogError com exceção
logger.LogError(ex,
    "Falha ao salvar pagamento | PaymentId: {PaymentId} | CorrelationId: {CorrelationId}",
    payment.Id, correlationId);

// NUNCA logar dados sensíveis (senhas, tokens, CPF, cartões)
```

### O que logar em cada nível

| Nível | Quando |
|---|---|
| `Information` | Erro de negócio (ErrorOr), fluxo normal |
| `Warning` | Token expirado, rate limit, tentativa suspeita |
| `Error` | Exceção de infraestrutura (DB, Redis, rede) |
| `Critical` | Serviço indisponível, dados corrompidos |

---

## Frontend — Error Interceptor (Angular)

### HTTP Error Interceptor

```typescript
// src/app/core/interceptors/error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';
import { NotificationService } from '../services/notification.service';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const notification = inject(NotificationService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      switch (error.status) {
        case 401:
          router.navigate(['/auth/login']);
          break;

        case 403:
          notification.error('Sem permissao para esta acao.');
          break;

        case 404:
          notification.error('Recurso nao encontrado.');
          break;

        case 409:
          notification.error('Conflito: o registro foi modificado. Recarregue a pagina.');
          break;

        case 422:
          // Erros de validacao — exibir detalhes
          const validationErrors = error.error?.errors;
          if (validationErrors) {
            const messages = Object.values(validationErrors).flat() as string[];
            notification.error(messages.join('\n'));
          } else {
            notification.error(error.error?.detail ?? 'Erro de validacao.');
          }
          break;

        case 500:
          notification.error('Erro interno. Tente novamente mais tarde.');
          break;

        default:
          notification.error('Erro inesperado.');
      }

      return throwError(() => error);
    })
  );
};
```

### Registro do Interceptor

```typescript
// src/app/app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { errorInterceptor } from './core/interceptors/error.interceptor';
import { authInterceptor } from './core/interceptors/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor])
    ),
    // ...
  ],
};
```

### Notification Service (exemplo com Tabler toast)

```typescript
// src/app/core/services/notification.service.ts
import { Injectable, signal } from '@angular/core';

export interface Notification {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  message: string;
}

@Injectable({ providedIn: 'root' })
export class NotificationService {
  readonly notifications = signal<Notification[]>([]);

  success(message: string): void { this.add('success', message); }
  error(message: string): void { this.add('error', message); }
  warning(message: string): void { this.add('warning', message); }
  info(message: string): void { this.add('info', message); }

  dismiss(id: string): void {
    this.notifications.update(list => list.filter(n => n.id !== id));
  }

  private add(type: Notification['type'], message: string): void {
    const id = crypto.randomUUID();
    this.notifications.update(list => [...list, { id, type, message }]);

    // Auto-dismiss apos 5 segundos
    setTimeout(() => this.dismiss(id), 5000);
  }
}
```

---

## Checklist

- [ ] `GlobalExceptionHandlerMiddleware` registrado no pipeline
- [ ] `ProblemDetails` configurado sem expor stack traces em produção
- [ ] `ErrorOrExtensions.ToActionResult` usado em todos os controllers
- [ ] Erros de negócio retornados via `ErrorOr` — nunca exceção
- [ ] Erros de validação via `FluentValidation` + `ValidationBehavior`
- [ ] Exception filters para cenários específicos (concorrência, etc.)
- [ ] Logging estruturado com `CorrelationId` em todos os erros
- [ ] Dados sensíveis NUNCA nos logs de erro
- [ ] `errorInterceptor` registrado no Angular `app.config.ts`
- [ ] Interceptor trata 401, 403, 404, 409, 422, 500
- [ ] `NotificationService` com signals para exibir erros ao usuário
