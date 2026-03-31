# Performance

## Objetivo

Diagnosticar e otimizar performance no backend (.NET/PostgreSQL/Redis) e frontend (Angular).

## Quando usar

- Endpoint lento (> 500ms)
- Página Angular com carregamento lento
- Bundle size grande
- Queries pesadas no banco

---

## EF Core — Problemas Comuns

### N+1

**Sintoma:** Query executa 1 + N consultas ao banco (1 para a lista, N para cada item).

```csharp
// Ruim: N+1
var users = await context.Users.ToListAsync(); // 1 query
foreach (var user in users)
    Console.WriteLine(user.Orders.Count); // N queries

// Bom: Include
var users = await context.Users
    .Include(u => u.Orders)
    .ToListAsync(); // 1 query com JOIN

// Melhor: Projeção (só traz o necessário)
var users = await context.Users
    .Select(u => new UserDto(u.Id, u.Name, u.Orders.Count))
    .ToListAsync(); // 1 query otimizada
```

### Queries Sem AsNoTracking

```csharp
// Ruim: EF Core rastreia entidades desnecessariamente em leitura
var users = await context.Users.ToListAsync();

// Bom: sem tracking em queries de leitura
var users = await context.Users.AsNoTracking().ToListAsync();
```

### Compiled Queries

Para queries executadas com alta frequência:

```csharp
private static readonly Func<AppDbContext, Guid, Task<User?>> GetByIdCompiled =
    EF.CompileAsyncQuery((AppDbContext ctx, Guid id) =>
        ctx.Users.AsNoTracking().FirstOrDefault(u => u.Id == id));

// Uso
var user = await GetByIdCompiled(context, userId);
```

### Split Queries

Quando Include traz muitas coleções (explosão cartesiana):

```csharp
var users = await context.Users
    .Include(u => u.Orders)
    .Include(u => u.Addresses)
    .AsSplitQuery() // divide em múltiplas queries em vez de JOIN gigante
    .ToListAsync();
```

---

## PostgreSQL — Diagnóstico

### EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT * FROM "Users" WHERE "Email" = 'test@example.com';
```

**O que procurar:**
- `Seq Scan` em tabela grande → falta índice
- `Nested Loop` com muitas linhas → possível N+1 ou falta de índice
- `Sort` sem índice → criar índice na coluna de ordenação
- Custo alto → otimizar query ou adicionar índice

### Slow Query Log

```sql
-- Ativar log de queries lentas (> 100ms)
ALTER SYSTEM SET log_min_duration_statement = 100;
SELECT pg_reload_conf();
```

### Índices Não Utilizados

```sql
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Redis — Otimização de Cache

### Cache-Aside Pattern

```csharp
public async Task<UserDto?> GetByIdCachedAsync(Guid id, CancellationToken ct)
{
    var cacheKey = $"user:{id}";
    var cached = await redis.StringGetAsync(cacheKey);

    if (cached.HasValue)
        return JsonSerializer.Deserialize<UserDto>(cached!);

    var user = await repository.GetByIdAsync(id, ct);
    if (user is null) return null;

    var dto = MapToDto(user);
    await redis.StringSetAsync(
        cacheKey,
        JsonSerializer.Serialize(dto),
        TimeSpan.FromMinutes(5)
    );

    return dto;
}
```

### Invalidação

```csharp
// Após UPDATE ou DELETE, invalide o cache
await redis.KeyDeleteAsync($"user:{userId}");

// Para listas, invalide com padrão
// Opção 1: TTL curto (1-5 min) para listas
// Opção 2: Invalidar chave da lista ao modificar item
```

### Serialização Eficiente

```csharp
// Ruim: JSON para dados grandes
await redis.StringSetAsync(key, JsonSerializer.Serialize(bigObject));

// Melhor: MessagePack ou protobuf para dados grandes e frequentes
```

---

## Angular — Bundle Size

### Analisar

```bash
npm run build -- --stats-json
npx webpack-bundle-analyzer dist/stats.json
```

### Otimizações

1. **Lazy loading** — toda feature via `loadChildren`/`loadComponent`
2. **Tree shaking** — importar apenas o necessário:
   ```typescript
   // Ruim: importa toda a lib
   import * as _ from 'lodash';

   // Bom: importa só a função
   import { debounce } from 'lodash-es';
   ```
3. **Imagens** — usar WebP, lazy loading nativo com `loading="lazy"`
4. **Fontes** — `font-display: swap` para não bloquear render

### Signals — Performance

```typescript
// computed() só recalcula quando dependências mudam
readonly filteredItems = computed(() =>
  this.items().filter(i => i.name.includes(this.search()))
);

// Evite trabalho pesado em templates — use computed
// Ruim: {{ items().filter(...).length }} no template
// Bom: {{ filteredCount() }} com computed
```

### OnPush + Signals = Performance Automática

Com `OnPush` + Signals, Angular só re-renderiza quando:
- Um `input()` muda
- Um `signal()` usado no template muda
- Um evento é disparado no componente

---

## Checklist

- [ ] Sem N+1 (usar Include ou projeção Select)
- [ ] `.AsNoTracking()` em todas as queries de leitura
- [ ] Índices nas colunas de WHERE/ORDER BY/JOIN
- [ ] EXPLAIN ANALYZE em queries críticas
- [ ] Cache Redis para dados frequentemente lidos
- [ ] Lazy loading em todas as features Angular
- [ ] Bundle size analisado e otimizado
- [ ] `computed()` para derivações no template
- [ ] `OnPush` em todos os componentes
