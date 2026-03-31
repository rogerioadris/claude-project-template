# Git Workflow

## Objetivo

Convenções de branching, commits e pull requests.

## Quando usar

- Criar branch para nova feature ou fix
- Escrever mensagem de commit
- Abrir pull request
- Definir estratégia de release

---

## Branching

### Nomes de Branch

```
feature/{ticket}-{descricao-curta}    → Nova funcionalidade
fix/{ticket}-{descricao-curta}        → Correção de bug
hotfix/{descricao-curta}              → Fix urgente em produção
chore/{descricao-curta}               → Manutenção, deps, CI/CD
refactor/{descricao-curta}            → Refatoração sem mudança funcional
```

### Exemplos

```
feature/MP-42-crud-usuarios
fix/MP-78-login-token-expirado
hotfix/corrige-calculo-saldo
chore/atualiza-efcore-8.1
refactor/extrai-transfer-service
```

### Regras

- Sempre a partir de `main` (ou `develop` se usar Gitflow)
- Nunca commitar diretamente na `main`
- Uma branch por feature/fix — não misture escopos
- Delete a branch após merge

---

## Commits — Conventional Commits

### Formato

```
tipo(escopo): descrição curta em português

Corpo opcional com mais detalhes.

Refs: #42
```

### Tipos

| Tipo | Quando usar |
|------|-------------|
| `feat` | Nova funcionalidade |
| `fix` | Correção de bug |
| `refactor` | Refatoração sem mudança funcional |
| `chore` | Manutenção, deps, CI/CD, configs |
| `docs` | Documentação |
| `test` | Adição ou correção de testes |
| `style` | Formatação, lint (sem mudança de lógica) |
| `perf` | Otimização de performance |

### Exemplos

```
feat(users): adiciona listagem paginada de usuários
fix(auth): corrige validação de token expirado
refactor(transactions): extrai lógica de transferência para domain service
chore(deps): atualiza MediatR para 12.4
test(users): adiciona testes de integração do controller
perf(queries): adiciona índice composto em transactions
```

### Regras

- Mensagem em português
- Imperativo: "adiciona", não "adicionado" ou "adicionando"
- Primeira linha max 72 caracteres
- Escopo = feature ou módulo afetado
- Referenciar ticket quando existir

---

## Pull Request

### Template

```markdown
## Resumo
<!-- 1-3 bullets do que foi feito e por quê -->

## Tipo de Mudança
- [ ] Nova feature
- [ ] Bug fix
- [ ] Refatoração
- [ ] Manutenção/Chore

## Checklist
- [ ] Código segue as convenções do projeto
- [ ] Testes adicionados/atualizados
- [ ] Build passa sem erros
- [ ] Sem dados sensíveis no código
- [ ] Migration criada (se aplicável)
- [ ] Documentação atualizada (se aplicável)

## Como Testar
<!-- Passos para testar as mudanças -->

## Screenshots
<!-- Se houver mudanças visuais -->
```

### Regras

- Título curto (< 70 caracteres)
- Descrição com contexto — por que, não apenas o quê
- PR pequeno (< 400 linhas de diff) — se maior, divida
- Sem commits de merge — use rebase ou squash
- Ao menos 1 reviewer antes de merge

---

## Estratégia de Merge

### Squash and Merge (recomendado)

- Cada PR vira 1 commit na main
- Histórico limpo e linear
- Mensagem do squash = título do PR

### Quando usar Merge Commit

- PRs grandes com histórico importante de decisões
- Release branches

### Quando usar Rebase

- Atualizar branch com mudanças da main antes de abrir PR

```bash
git checkout feature/minha-feature
git rebase main
git push --force-with-lease  # nunca --force
```

---

## Release Workflow

### Versioning (SemVer)

```
MAJOR.MINOR.PATCH
1.0.0 → 1.1.0 (nova feature)
1.1.0 → 1.1.1 (bug fix)
1.1.1 → 2.0.0 (breaking change)
```

### Tags

```bash
git tag -a v1.2.0 -m "feat: adiciona módulo de relatórios"
git push origin v1.2.0
```

### Hotfix Flow

```bash
# 1. Branch a partir da tag de produção
git checkout -b hotfix/corrige-saldo v1.2.0

# 2. Fix + commit
git commit -m "fix(balance): corrige cálculo de saldo em transferências"

# 3. Tag + merge em main
git tag -a v1.2.1 -m "fix: corrige cálculo de saldo"
git checkout main
git merge hotfix/corrige-saldo
git push origin main --tags

# 4. Cleanup
git branch -d hotfix/corrige-saldo
```

---

## .gitignore Essencial

```gitignore
# .NET
bin/
obj/
*.user
appsettings.Development.json
appsettings.*.local.json

# Angular
node_modules/
dist/
.angular/

# IDE
.vs/
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Secrets
*.env
*.pem
*.key
```

---

## Checklist

- [ ] Branch nomeada seguindo convenção
- [ ] Commits em Conventional Commits (português)
- [ ] PR com descrição, tipo e checklist preenchidos
- [ ] PR < 400 linhas de diff (dividir se necessário)
- [ ] Build passa antes de abrir PR
- [ ] Testes passam antes de merge
- [ ] Branch deletada após merge
- [ ] `.gitignore` cobre arquivos sensíveis e build artifacts
