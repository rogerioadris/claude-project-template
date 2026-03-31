# Autenticação e Autorização

## Objetivo

Guia completo para implementar e revisar autenticação (JWT + BCrypt) e autorização (roles + permissões) na stack .NET + Angular.

## Quando usar

- Implementar fluxo de login/logout/refresh
- Criar endpoints protegidos por role ou permissão
- Revisar segurança de autenticação existente
- Configurar guards e interceptors no Angular

---

## JWT

### Assinatura

- **RS256 (assimétrico) em produção** — a API assina com chave privada, valida com pública. Se a pública vazar, ninguém gera tokens.
- **HS256 apenas em dev** — secret simétrico, mais simples mas inseguro se vazar.

### Tempos de Expiração

| Token | Duração | Onde armazenar |
|-------|---------|----------------|
| Access token | 15 min a 1h | `localStorage` ou memória (Angular) |
| Refresh token | 7 a 30 dias | `httpOnly` cookie ou banco |

### Payload — O que incluir

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "role": "Admin",
  "jti": "unique-token-id",
  "exp": 1711839600
}
```

**Nunca no payload:** email, CPF, dados de cartão, senha. JWT é base64, não criptografia.

### Rotação de Refresh Token

1. Cliente envia refresh token
2. Backend valida e **invalida o token usado**
3. Backend emite novo access token + novo refresh token
4. Se token já invalidado for reutilizado → revogar **todos** os tokens do usuário (possível roubo)

### Blacklist via Redis (Logout/Revogação)

```csharp
// Ao fazer logout ou revogar acesso
await redis.StringSetAsync(
    $"token_blacklist:{jti}",
    "revoked",
    expiry: tokenRemainingLifetime // TTL = tempo restante do token
);

// No AuthorizationBehavior, antes de aceitar o token
var isBlacklisted = await redis.KeyExistsAsync($"token_blacklist:{jti}");
if (isBlacklisted) return Error.Unauthorized("Token.Revoked", "Token revogado.");
```

---

## Senhas (BCrypt)

- Hash com `BCrypt.Net-Next` custo >= 12 (~250ms por hash)
- Mínimo 8 caracteres, sem restrição de caracteres especiais
- Nunca MD5, SHA1 ou SHA256 para senhas

### Bloqueio Progressivo

Armazene tentativas no Redis: `login_attempts:{email}`

| Tentativas | Ação |
|-----------|------|
| 5 | Exigir CAPTCHA |
| 10 | Lockout 15 minutos |
| 20 | Lockout 1 hora + notificar admin |

```csharp
var attempts = await redis.StringIncrementAsync($"login_attempts:{email}");
await redis.KeyExpireAsync($"login_attempts:{email}", TimeSpan.FromMinutes(15));

if (attempts > 10)
    return Error.Forbidden("Auth.Locked", "Conta bloqueada temporariamente.");
if (attempts > 5)
    // exigir CAPTCHA no frontend
```

---

## Autorização — 3 Camadas

```
Frontend (UX)          → Esconde botões/rotas que o usuário não pode acessar
API (Controller)       → [Authorize(Policy = AppPolicies.XXX)]
Handler (Pipeline)     → IAuthorizedRequest + IPermissionRequiredRequest
```

**Regra:** O frontend é cosmético. A segurança real está no backend.

### Interfaces de Pipeline

```csharp
// Endpoint público (login, health, refresh)
public sealed record LoginCommand(string Email, string Password)
    : IRequest<ErrorOr<LoginResult>>;

// Apenas autenticado
public sealed record GetProfileQuery(Guid UserId)
    : IRequest<ErrorOr<UserDto>>, IAuthorizedRequest;

// Autenticado + permissão específica
public sealed record DeleteUserCommand(Guid Id)
    : IRequest<ErrorOr<Deleted>>, IAuthorizedRequest, IPermissionRequiredRequest
{
    public string RequiredPermission => AppPermissions.UsersDelete;
}
```

### IDOR (Insecure Direct Object Reference)

Mesmo com `IAuthorizedRequest`, valide no Handler se o recurso pertence ao usuário:

```csharp
// Ruim: qualquer usuário autenticado acessa qualquer conta
var account = await repo.GetByIdAsync(request.AccountId);

// Bom: valida que a conta pertence ao usuário logado
var account = await repo.GetByUserIdAndAccountIdAsync(currentUserId, request.AccountId);
if (account is null)
    return Error.Forbidden("Account.AccessDenied", "Sem acesso a esta conta.");
```

### Permissões por Negação

O template usa `UserPermissions` para registrar **negações** individuais. Permissões vêm dos roles; negações sobrescrevem.

Exemplo: Usuário tem role `Admin` (todas as permissões), mas `UserPermissions` nega `UsersDelete` → ele não pode deletar usuários.

---

## Angular — Auth no Frontend

### Auth Interceptor

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }
  return next(req);
};
```

### Error Interceptor (401/403)

```typescript
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        authService.logout(); // limpa token, redireciona para /login
      }
      if (error.status === 403) {
        router.navigate(['/unauthorized']);
      }
      return throwError(() => error);
    })
  );
};
```

### Auth Guard

```typescript
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) return true;

  router.navigate(['/login']);
  return false;
};
```

### Role Guard

```typescript
export const roleGuard: CanActivateFn = (route) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  const requiredRole = route.data['requiredRole'] as string;

  if (authService.hasRole(requiredRole)) return true;

  router.navigate(['/unauthorized']);
  return false;
};
```

### Esconder UI por Permissão

```typescript
// No componente
readonly canDelete = computed(() =>
  this.authService.hasPermission('users.delete')
);
```

```html
@if (canDelete()) {
  <button class="btn btn-ghost-danger" (click)="delete()">Excluir</button>
}
```

---

## Checklist

- [ ] JWT assinado com RS256 em produção
- [ ] Access token exp <= 1h
- [ ] Refresh token com rotação (invalida anterior ao usar)
- [ ] Blacklist de tokens via Redis no logout
- [ ] BCrypt custo >= 12
- [ ] Rate limiting no login (Redis: `login_attempts:{email}`)
- [ ] `IAuthorizedRequest` em todos os endpoints não-públicos
- [ ] `IPermissionRequiredRequest` com `AppPermissions.*` onde necessário
- [ ] IDOR validado no Handler (recurso pertence ao usuário?)
- [ ] Auth interceptor + error interceptor registrados em `app.config.ts`
- [ ] `authGuard` no layout principal
- [ ] `roleGuard` com `data: { requiredRole }` nas rotas restritas
- [ ] Frontend esconde UI mas nunca é a única proteção
- [ ] `appsettings.Development.json` no `.gitignore`
- [ ] Secrets rotacionados a cada 90 dias
