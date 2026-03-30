# CLAUDE.md — Frontend

Orientações específicas para o frontend. Consulte o `CLAUDE.md` raiz para idioma, stack geral e domínio do negócio.

## Skills do Frontend

- `/frontend-new-feature` — ao criar nova feature (model + service + store + componentes)
- `/frontend-conventions` — convenções detalhadas (nomenclatura, models, config)
- `/frontend-state-routing` — Signals, Store pattern e roteamento
- `/frontend-tabler` — classes, layout e componentes do Tabler.io

---

## Stack

| Tecnologia      | Versão   | Observação                        |
|-----------------|----------|-----------------------------------|
| Angular         | 21.x     | Standalone Components obrigatório |
| TypeScript      | 5.7+     | `strict: true` ativado            |
| Node.js         | 22.x LTS | Versão mínima requerida           |
| RxJS            | 7.x      | Bundled com Angular 21            |
| Angular Signals | nativo   | Preferido sobre BehaviorSubject   |
| Tabler.io       | 1.x      | Via npm                           |
| Tabler Icons    | 3.x      | SVG sprite ou font                |

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

- **Standalone components** — NgModules são proibidos.
- `ChangeDetectionStrategy.OnPush` em **todos** os componentes, sem exceção.
- Injeção via `inject()` — nunca via construtor com `@Inject`.
- Inputs via `input()` signal, outputs via `output()` — não usar `@Input()` / `@Output()`.
- `@if`, `@for`, `@switch` — nunca `*ngIf`, `*ngFor`. `track` obrigatório no `@for`.
- **Signals** para estado local e compartilhado; RxJS apenas para HTTP e WebSockets.
- **Lazy loading** em todos os feature modules via `loadChildren` / `loadComponent`.
- Chamadas HTTP apenas em **services**, nunca em componentes.
- Nunca usar `any` — preferir `unknown` quando o tipo não é conhecido.
- Não misturar outras bibliotecas de UI (Angular Material, PrimeNG) com Tabler.
- Prefixo `app-` em todos os seletores de componentes.
- Interfaces para contratos de API; types para uniões e utilitários.
- `readonly` em propriedades de objetos de valor.

---

## Checklist de Início de Projeto

Execute uma única vez ao iniciar um novo projeto baseado neste template:

- [ ] Projeto Angular criado (`ng new {nome} --routing --style=scss`)
- [ ] Tabler.io instalado (`npm install @tabler/core @tabler/icons`)
- [ ] `styles.scss` configurado importando Tabler após `_variables.scss`
- [ ] `_variables.scss` com cores e variáveis do projeto
- [ ] `environment.ts` com `apiUrl` apontando para o backend
- [ ] `app.config.ts` com `provideRouter(appRoutes, withViewTransitions())`
- [ ] Auth interceptor + error interceptor registrados em `app.config.ts`
- [ ] `authGuard` implementado e aplicado no layout principal
- [ ] Layout principal (`main-layout`) com sidebar e navbar criados
- [ ] Prefixo `app-` configurado em `angular.json`
- [ ] `tsconfig.json` com `strict: true`
- [ ] `.gitignore` configurado (node_modules, dist, .angular)
