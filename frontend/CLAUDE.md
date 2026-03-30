# CLAUDE.md — Frontend

Orientações específicas para o frontend. Consulte o `CLAUDE.md` raiz para idioma, stack geral e domínio do negócio.

## Skills do Frontend

- `/frontend-new-feature` — ao criar nova feature (model + service + store + componentes)
- `/frontend-conventions` — convenções detalhadas (nomenclatura, models, config)
- `/frontend-state-routing` — Signals, Store pattern e roteamento
- `/frontend-tabler` — classes, layout e componentes do Tabler.io

---

## Comandos Comuns

```bash
# Instalar dependências
npm install

# Executar em modo desenvolvimento
npm start

# Build de produção
npm run build

# Executar testes unitários
npm test

# Executar testes e2e
npm run e2e

# Lint
npm run lint
```

---

## Regras Inegociáveis

- **Standalone only** — NgModules proibidos. `OnPush` em tudo. `inject()` sempre.
- **Signals API:** `input()`, `output()`, `signal()`, `computed()` — nunca decorators legados.
- **Control flow:** `@if`/`@for`/`@switch` — nunca `*ngIf`/`*ngFor`. `track` obrigatório.
- **Signals** para estado; **RxJS** apenas para HTTP e WebSockets.
- **Lazy loading** obrigatório. HTTP só em services. `any` proibido. Prefixo `app-`.
- **Tabler.io exclusivo** — nunca misturar com Angular Material, PrimeNG, etc.

> Lista completa via `/frontend-conventions`

> Checklist de início de projeto disponível via `/project-bootstrap`
