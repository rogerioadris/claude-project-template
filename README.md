# Project Template — Fullstack .NET + Angular

Template pré-configurado para projetos fullstack com **Claude Code**. Inclui convenções, arquitetura, skills e automações prontas para uso.

## Stack Tecnológica

| Camada | Tecnologia | Versão |
|--------|-----------|--------|
| Backend | .NET / ASP.NET Core | 8.x |
| Frontend | Angular (Standalone) | 21.x |
| Banco de Dados | PostgreSQL | 16+ |
| Cache / Lock | Redis | — |
| Message Broker | RabbitMQ | — |

## Estrutura do Projeto

```
.
├── backend/
│   └── CLAUDE.md              # Convenções e comandos do backend
├── frontend/
│   └── CLAUDE.md              # Convenções e comandos do frontend
├── .claude/
│   ├── commands/              # 13 skills para Claude Code
│   └── settings.json          # Hooks de automação
├── CLAUDE.md                  # Regras absolutas e skills disponíveis
└── README.md
```

## Pré-requisitos

- [.NET SDK 8+](https://dotnet.microsoft.com/download)
- [Node.js 22 LTS](https://nodejs.org/)
- [Docker](https://www.docker.com/) (PostgreSQL, Redis, RabbitMQ)
- [Claude Code](https://claude.ai/code) (CLI ou extensão IDE)

## Como Começar

```bash
# 1. Clone o template
git clone <url-do-repo> meu-projeto
cd meu-projeto

# 2. Abra o Claude Code e execute o skill de bootstrap
claude
# Dentro do Claude Code:
/project-bootstrap
```

O skill `/project-bootstrap` guia a criação da solution .NET, projeto Angular, configuração de Docker Compose e setup inicial completo.

## Skills Disponíveis

Skills são documentação sob demanda invocada via `/nome-do-skill` no Claude Code.

| Comando | Quando usar |
|---------|-------------|
| `/project-bootstrap` | Setup inicial do projeto (backend + frontend) |
| `/fullstack-feature` | Criar recurso completo (backend + frontend juntos) |
| `/backend-new-feature` | Criar novo recurso/entidade no backend |
| `/backend-architecture` | Consultar arquitetura, CQRS flow, estrutura de pastas |
| `/backend-conventions` | Consultar convenções detalhadas de código backend |
| `/backend-templates` | Templates de infraestrutura (Pipeline, DI, Program.cs) |
| `/frontend-new-feature` | Criar nova feature no frontend Angular |
| `/frontend-conventions` | Consultar convenções detalhadas de código frontend |
| `/frontend-state-routing` | Consultar padrões de estado (Signals) e roteamento |
| `/frontend-tabler` | Consultar integração e classes do Tabler.io |
| `/security-review` | Auditoria de segurança (auth, OWASP, DB) |
| `/scalability-review` | Revisar escalabilidade (API, PostgreSQL, Redis, RabbitMQ) |
| `/researcher` | Conduzir pesquisa estruturada para decisão técnica |

## Regras Absolutas

Estas regras valem **sempre**, em qualquer parte do código:

1. **Minor Units** — todo valor monetário é `long` em centavos (`R$ 1,00 = 100`)
2. **UUID v7** — todas as chaves primárias novas
3. **`DateTime.UtcNow`** — nunca `DateTime.Now`
4. **Sem setters públicos** — entidades de domínio usam métodos para mutação
5. **ErrorOr** — nunca lance exceção para erros de negócio
6. **Sem `goto`** — use flags, métodos auxiliares ou reestruture o fluxo
7. **Business vs Infra** — erro de negócio falha direto; erro de infra faz retry com backoff
8. **RedLock** — obrigatório ao atualizar saldos ou estados concorrentes

## Comandos Comuns

### Backend

```bash
dotnet restore && dotnet build
dotnet run --project src/{NomeProjeto}.API
dotnet watch run --project src/{NomeProjeto}.API
dotnet test
dotnet ef migrations add <Nome> --project src/{NomeProjeto}.Infrastructure --startup-project src/{NomeProjeto}.API
docker compose up -d
```

### Frontend

```bash
npm install
npm start
npm run build
npm test
npm run lint
```

## Convenções Resumidas

### Backend (Clean Architecture + CQRS)

- `Domain` → `Application` → `Infrastructure` → `API`
- Toda operação via MediatR `Command` ou `Query`
- Controllers delegam 100% para `IMediator`
- Handlers retornam `ErrorOr<T>` — sem exceções para erros de negócio
- Query Handlers usam `.AsNoTracking()` + projeção para DTO
- Auditoria (`IAuditService`) após `SaveChangesAsync`

### Frontend (Angular 21 Standalone)

- Standalone only — NgModules proibidos
- `ChangeDetectionStrategy.OnPush` em tudo
- Signals API: `input()`, `output()`, `signal()`, `computed()`
- Control flow: `@if` / `@for` / `@switch`
- Signals para estado; RxJS apenas para HTTP e WebSockets
- Tabler.io exclusivo — sem misturar com outras libs de UI

## Idioma

- **Documentação, comentários e commits** — português do Brasil (pt-br)
- **Código-fonte** — inglês
- **Commits** — Conventional Commits: `feat(users): adiciona listagem paginada`

## Automações (Hooks)

O template inclui hooks pré-configurados em `.claude/settings.json`:

- **Auto-lint** — ESLint roda automaticamente após edição de `.ts` no frontend
- **Build check** — `dotnet build` roda automaticamente após edição de `.cs` no backend
- **Aviso de arquivo grande** — alerta ao ler arquivos com mais de 500 linhas
