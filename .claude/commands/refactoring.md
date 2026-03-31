# Refactoring

## Objetivo

Padrões seguros para refatorar código existente sem quebrar funcionalidades.

## Quando usar

- Renomear entidade ou feature
- Extrair lógica para Domain Service
- Extrair componente compartilhado no frontend
- Reorganizar estrutura de pastas

---

## Regra de Ouro

**Nunca refatore e adicione funcionalidade no mesmo commit.** Separe em:
1. Commit de refatoração (mesma funcionalidade, estrutura diferente)
2. Commit de funcionalidade nova

---

## Renomear Entidade/Feature (Backend)

### Arquivos afetados

```
Domain/Entities/{Old}.cs           → {New}.cs
Application/Common/Interfaces/I{Old}Repository.cs → I{New}Repository.cs
Application/Features/{Old}/        → {New}/
  Commands/Create{Old}/            → Create{New}/
  Commands/Update{Old}/            → Update{New}/
  Commands/Delete{Old}/            → Delete{New}/
  Queries/Get{Old}ById/            → Get{New}ById/
  Queries/GetAll{Old}s/            → GetAll{New}s/
Infrastructure/Repositories/{Old}Repository.cs → {New}Repository.cs
Infrastructure/Data/Configurations/{Old}Configuration.cs → {New}Configuration.cs
API/Controllers/{Old}sController.cs → {New}sController.cs
Tests/**/{Old}*                    → {New}*
```

### Passos

1. Renomear a entidade e propriedades no Domain
2. Renomear interface do repositório
3. Renomear Commands, Queries, Handlers, Validators, DTOs
4. Renomear repositório concreto e atualizar DI
5. Renomear Configuration EF Core (atenção: nome da tabela pode mudar)
6. Criar migration para renomear tabela: `RenameTable("OldName", "NewName")`
7. Renomear Controller
8. Atualizar testes
9. Rodar `dotnet build` + `dotnet test`

### Migration de Rename

```csharp
migrationBuilder.RenameTable(name: "OldEntities", newName: "NewEntities");
migrationBuilder.RenameColumn(
    name: "OldColumn",
    table: "NewEntities",
    newName: "NewColumn"
);
```

---

## Renomear Feature (Frontend)

### Arquivos afetados

```
features/{old}/
  components/{old}-list/    → {new}-list/
  components/{old}-form/    → {new}-form/
  services/{old}.service.ts → {new}.service.ts
  models/{old}.model.ts     → {new}.model.ts
  {old}.store.ts            → {new}.store.ts
  {old}.routes.ts           → {new}.routes.ts
  {old}.component.ts        → {new}.component.ts
app.routes.ts               → atualizar path e import
```

### Passos

1. Renomear pasta da feature
2. Renomear cada arquivo dentro
3. Atualizar seletores: `app-{old}-list` → `app-{new}-list`
4. Atualizar imports em `app.routes.ts`
5. Atualizar sidebar/menu se houver entrada
6. Rodar `npm run build` + `npm test`

---

## Extrair Domain Service

**Quando:** Lógica de negócio que envolve múltiplas entidades ou regras complexas demais para viver numa única entidade.

### Antes

```csharp
// Handler com lógica complexa
public async Task<ErrorOr<Guid>> Handle(TransferCommand request, ...)
{
    var source = await sourceRepo.GetByIdAsync(request.SourceId);
    var target = await targetRepo.GetByIdAsync(request.TargetId);
    // 30 linhas de validações e lógica de transferência...
}
```

### Depois

```csharp
// Domain Service em Application/Common/Services/
public sealed class TransferService(
    IAccountRepository accountRepo,
    IDistributedLockProvider lockProvider
)
{
    public async Task<ErrorOr<TransferResult>> ExecuteAsync(
        Guid sourceId, Guid targetId, long amountInCents, CancellationToken ct)
    {
        // Lógica de transferência encapsulada
        // RedLock nos dois saldos
        // Validações de negócio
    }
}

// Handler limpo
public async Task<ErrorOr<Guid>> Handle(TransferCommand request, ...)
{
    var result = await transferService.ExecuteAsync(
        request.SourceId, request.TargetId, request.AmountInCents, ct);
    // ...
}
```

---

## Extrair Shared Component (Frontend)

**Quando:** Componente usado em 2+ features.

### Passos

1. Mover de `features/{feature}/components/` para `shared/components/`
2. Garantir que é um "dumb component" (só `input()` e `output()`, sem services injetados)
3. Atualizar imports em todos os componentes que usam
4. Se tinha lógica de negócio → mover para o smart component pai

### Estrutura

```
shared/components/
  data-table/
    data-table.component.ts
    data-table.component.html
    data-table.component.scss
    data-table.component.spec.ts
```

---

## Checklist Pré-Refatoração

- [ ] Testes existentes passam (baseline)
- [ ] Commit limpo antes de começar (pode reverter)
- [ ] Escopo definido (o que vai mudar e o que NÃO vai)
- [ ] Sem funcionalidade nova misturada

## Checklist Pós-Refatoração

- [ ] `dotnet build` sem erros
- [ ] `dotnet test` sem falhas
- [ ] `npm run build` sem erros
- [ ] `npm test` sem falhas
- [ ] Migration criada se houve rename de tabela/coluna
- [ ] Imports e referências atualizados
- [ ] Commit de refatoração separado do commit de feature
