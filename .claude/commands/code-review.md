# Code Review

## Objetivo

Revisar código com checklist estruturado, específico para a stack .NET + Angular do projeto.

## Quando usar

- Antes de abrir PR
- Revisar código de outro desenvolvedor
- Auto-revisão após feature completa

## Como executar

1. Leia o código alterado (diff ou arquivos)
2. Aplique o checklist por categoria abaixo
3. Para cada problema: explique, mostre o trecho e a correção
4. Finalize com score (1-10) e 3 sugestões prioritárias

---

## Backend — Checklist

### Arquitetura e CQRS

- [ ] Controller delega 100% para `IMediator` — sem lógica de negócio
- [ ] Command/Query é `sealed record` implementando `IRequest<ErrorOr<T>>`
- [ ] Um Handler por arquivo, classe `sealed`
- [ ] Handler nunca injeta `IMediator`
- [ ] Cada Query tem seu próprio DTO

### ErrorOr e Fluxo

- [ ] Handler retorna `ErrorOr<T>` — nenhum `throw` para erros de negócio
- [ ] Erros usam helpers semânticos: `Error.Validation`, `.NotFound`, `.Conflict`
- [ ] Controller converte via `.ToActionResult(this, Ok)`
- [ ] Exceções de infra capturadas pelo middleware global

### Entidades de Domínio

- [ ] Construtor `private` para EF Core
- [ ] Sem setters públicos — mutação via métodos
- [ ] `DateTime.UtcNow` — nunca `DateTime.Now`
- [ ] `Guid.CreateVersion7()` — nunca `Guid.NewGuid()`
- [ ] Valores monetários em `long` (centavos)

### Queries e Performance

- [ ] `.AsNoTracking()` em todas as queries de leitura
- [ ] Projeção via `.Select()` para DTO — nunca retorna entidade
- [ ] Sem N+1 (verificar `.Include()` ou projeção)
- [ ] Paginação via `PagedResult<T>` quando aplicável

### Segurança

- [ ] `IAuthorizedRequest` em endpoints não-públicos
- [ ] `IPermissionRequiredRequest` com `AppPermissions.*` onde necessário
- [ ] IDOR validado — recurso pertence ao usuário?
- [ ] Sem dados sensíveis em logs
- [ ] `appsettings.Development.json` não commitado

### Auditoria

- [ ] `IAuditService.LogAsync` chamado após `SaveChangesAsync`
- [ ] Ações padronizadas: CREATE, UPDATE, DELETE

---

## Frontend — Checklist

### Componentes

- [ ] `standalone: true` — sem NgModules
- [ ] `ChangeDetectionStrategy.OnPush`
- [ ] Injeção via `inject()` — nunca `@Inject` no construtor
- [ ] `input()` / `output()` — nunca `@Input()` / `@Output()`
- [ ] Prefixo `app-` no seletor

### Templates

- [ ] `@if` / `@for` / `@switch` — nunca `*ngIf` / `*ngFor`
- [ ] `track` presente em todo `@for`
- [ ] Sem `any` — tipos explícitos ou `unknown`

### Estado e HTTP

- [ ] Signals para estado local e compartilhado
- [ ] RxJS apenas para HTTP e WebSockets
- [ ] Chamadas HTTP apenas em services — nunca em componentes
- [ ] Lazy loading em todas as features

### Segurança Frontend

- [ ] Sem `[innerHTML]` com dados do usuário não sanitizados
- [ ] Sem `bypassSecurityTrust*` desnecessário
- [ ] Guards aplicados nas rotas protegidas

---

## Output Esperado

```markdown
## Score: X/10

## Problemas Encontrados

### [Crítico] Título
- **Arquivo:** path/to/file.cs:42
- **Problema:** descrição
- **Correção:** código corrigido

### [Médio] Título
...

## Top 3 Sugestões Prioritárias
1. ...
2. ...
3. ...
```
