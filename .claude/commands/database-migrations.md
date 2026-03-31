# Database e Migrations

## Objetivo

Guia para criar migrations, modelagem EF Core, índices e particionamento no PostgreSQL.

## Quando usar

- Criar nova entidade e migration
- Adicionar índice ou otimizar query
- Configurar seed data
- Modelagem avançada (value objects, owned types)

---

## Migrations — Fluxo

### Criar

```bash
dotnet ef migrations add {NomeDaMigration} \
  --project src/{NomeProjeto}.Infrastructure \
  --startup-project src/{NomeProjeto}.API
```

### Aplicar

```bash
dotnet ef database update \
  --project src/{NomeProjeto}.Infrastructure \
  --startup-project src/{NomeProjeto}.API
```

### Reverter última

```bash
dotnet ef database update {MigrationAnterior} \
  --project src/{NomeProjeto}.Infrastructure \
  --startup-project src/{NomeProjeto}.API
```

### Remover última (se não aplicada)

```bash
dotnet ef migrations remove \
  --project src/{NomeProjeto}.Infrastructure \
  --startup-project src/{NomeProjeto}.API
```

### Gerar script SQL (para produção)

```bash
dotnet ef migrations script {From} {To} \
  --project src/{NomeProjeto}.Infrastructure \
  --startup-project src/{NomeProjeto}.API \
  --output migration.sql
```

---

## Naming de Migrations

| Ação | Nome | Exemplo |
|------|------|---------|
| Nova entidade | `Add{Entidade}` | `AddTransaction` |
| Novo campo | `Add{Campo}To{Entidade}` | `AddStatusToTransaction` |
| Índice | `AddIndex{Campo}On{Entidade}` | `AddIndexStatusOnTransaction` |
| Remover campo | `Remove{Campo}From{Entidade}` | `RemoveOldFieldFromUser` |
| Seed data | `Seed{Entidade}` | `SeedRoles` |

---

## Entity Configuration (IEntityTypeConfiguration)

```csharp
public sealed class TransactionConfiguration : IEntityTypeConfiguration<Transaction>
{
    public void Configure(EntityTypeBuilder<Transaction> builder)
    {
        builder.ToTable("Transactions");

        builder.HasKey(t => t.Id);

        builder.Property(t => t.Id)
            .ValueGeneratedNever(); // UUID v7 gerado pelo domínio

        builder.Property(t => t.AmountInCents)
            .IsRequired();

        builder.Property(t => t.Description)
            .HasMaxLength(500);

        builder.Property(t => t.Status)
            .HasConversion<string>() // enum como string no banco
            .HasMaxLength(50);

        builder.Property(t => t.CreatedAt)
            .IsRequired();

        // Índices
        builder.HasIndex(t => t.Status);
        builder.HasIndex(t => t.CreatedAt);
        builder.HasIndex(t => new { t.CustomerId, t.CreatedAt }); // composto
    }
}
```

---

## Índices

### Quando criar

- Colunas em WHERE, ORDER BY, JOIN
- Foreign keys (EF Core não cria automaticamente em todos os casos)
- Combinações de colunas usadas juntas em filtros

### Quando NÃO criar

- Tabelas com < 10k linhas
- Colunas com baixa cardinalidade (ex: bool, status com 3 valores)
- Se a tabela tem muitas escritas e poucas leituras

### Tipos de Índice (PostgreSQL)

```csharp
// B-tree (padrão) — igualdade e range
builder.HasIndex(t => t.CreatedAt);

// Composto — filtros combinados
builder.HasIndex(t => new { t.CustomerId, t.Status });

// Unique — constraint de unicidade
builder.HasIndex(t => t.Email).IsUnique();

// Partial (via SQL) — índice apenas para um subconjunto
builder.HasIndex(t => t.Status)
    .HasFilter("\"Status\" = 'Pending'");

// GIN (via SQL) — full-text search
migrationBuilder.Sql(
    "CREATE INDEX IX_Products_SearchVector ON \"Products\" USING gin(to_tsvector('portuguese', \"Name\" || ' ' || \"Description\"));"
);
```

---

## Seed Data

```csharp
// Na migration ou no DatabaseSeeder
public static class DatabaseSeeder
{
    public static async Task SeedAsync(AppDbContext context)
    {
        if (await context.Roles.AnyAsync()) return; // idempotente

        var roles = new[]
        {
            new Role(SeedIds.AdminRoleId, "Admin"),
            new Role(SeedIds.EmployeeRoleId, "Employee"),
        };

        context.Roles.AddRange(roles);
        await context.SaveChangesAsync();
    }
}
```

- Use `SeedIds` (constantes em Domain) para IDs fixos do seed
- Sempre idempotente: verifique se já existe antes de inserir
- Execute no `InitializeDatabaseAsync()` do Program.cs

---

## Value Converters

```csharp
// Enum → string
builder.Property(t => t.Status)
    .HasConversion<string>();

// Money (long centavos) — sem conversão necessária, já é long

// DateOnly → DateTime (se precisar)
builder.Property(t => t.BirthDate)
    .HasConversion(
        v => v.ToDateTime(TimeOnly.MinValue),
        v => DateOnly.FromDateTime(v)
    );
```

## Owned Types (Value Objects)

```csharp
// Entidade com value object
public sealed class Customer
{
    public Address Address { get; private set; }
}

// Configuração
builder.OwnsOne(c => c.Address, a =>
{
    a.Property(x => x.Street).HasMaxLength(200).IsRequired();
    a.Property(x => x.City).HasMaxLength(100).IsRequired();
    a.Property(x => x.State).HasMaxLength(2).IsRequired();
    a.Property(x => x.ZipCode).HasMaxLength(9).IsRequired();
});
```

---

## Particionamento (tabelas grandes)

Para tabelas com milhões de registros (transações, logs, audit trail):

```sql
-- Criar tabela particionada por range de data
CREATE TABLE "AuditLogs" (
    "Id" uuid NOT NULL,
    "Action" varchar(50) NOT NULL,
    "CreatedAt" timestamp NOT NULL,
    -- demais colunas
) PARTITION BY RANGE ("CreatedAt");

-- Criar partições por mês
CREATE TABLE "AuditLogs_2026_01" PARTITION OF "AuditLogs"
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE "AuditLogs_2026_02" PARTITION OF "AuditLogs"
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

- Facilita DROP de dados antigos (drop partition, não DELETE)
- Queries com filtro de data são mais rápidas

---

## Checklist

- [ ] Migration nomeada seguindo convenção (`Add{Entidade}`, `AddIndex...`)
- [ ] `ValueGeneratedNever()` em IDs (UUID v7 gerado pelo domínio)
- [ ] Enums convertidos para string no banco
- [ ] Índices nas colunas usadas em WHERE/ORDER BY/JOIN
- [ ] Foreign keys com índice quando necessário
- [ ] Seed data idempotente (verifica antes de inserir)
- [ ] Script SQL gerado para migrations de produção
- [ ] Tabelas grandes particionadas por data
