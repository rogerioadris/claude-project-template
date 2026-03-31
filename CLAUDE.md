# CLAUDE.md

Este arquivo fornece orientações ao Claude Code ao trabalhar neste repositório.

## Idioma

Toda documentação, comentários e mensagens de commit devem ser escritos em **português do Brasil (pt-br)**. Código-fonte, nomes de variáveis, funções, classes e identificadores devem permanecer em **inglês**.

## Visão Geral do Projeto

**{Descrição do projeto}** — {resumo do que o sistema faz}.

- **Backend**: `backend/` — veja [`backend/CLAUDE.md`](backend/CLAUDE.md)
- **Frontend**: `frontend/` — veja [`frontend/CLAUDE.md`](frontend/CLAUDE.md)

## Stack Tecnológica

| Camada | Tecnologia | Versão |
|--------|-----------|--------|
| Backend | .NET / ASP.NET Core | 8.x |
| Frontend | Angular (Standalone) | 21.x |
| Banco de Dados | PostgreSQL | 16+ |
| Cache / Lock | Redis | — |
| Message Broker | RabbitMQ | — |

## Skills Disponíveis

Ao receber uma tarefa que envolva criar novo recurso, feature ou configurar o projeto, **invoque o skill correspondente automaticamente** antes de começar a implementar. Não espere o usuário pedir.

| Comando | Quando usar |
|---------|-------------|
| `/project-bootstrap` | Setup inicial do projeto (backend + frontend) |
| `/fullstack-feature` | Criar recurso completo (backend + frontend juntos) |
| `/backend-new-feature` | Criar novo recurso/entidade no backend |
| `/backend-architecture` | Consultar arquitetura, CQRS flow, estrutura de pastas |
| `/backend-conventions` | Consultar convenções detalhadas de código backend |
| `/backend-templates` | Templates de infraestrutura (Pipeline, DI, Program.cs, NuGet) |
| `/frontend-new-feature` | Criar nova feature no frontend Angular |
| `/frontend-conventions` | Consultar convenções detalhadas de código frontend |
| `/frontend-state-routing` | Consultar padrões de estado (Signals) e roteamento |
| `/frontend-tabler` | Consultar integração e classes do Tabler.io |
| `/security-review` | Auditoria de segurança (auth, OWASP, DB, dependências) |
| `/scalability-review` | Revisar escalabilidade (API, PostgreSQL, Redis, RabbitMQ) |
| `/researcher` | Conduzir pesquisa estruturada para decisão técnica |
| `/auth-reference` | Autenticação (JWT, BCrypt) e autorização (roles, permissões, guards) |

<!-- TODO: criar skills quando estes docs forem escritos:
  docs/domain.md → /domain-reference
  docs/database.md → /database-schema
  backend/docs/infra.md → /backend-infra
  backend/docs/api.md → /api-reference
-->

## Regras Absolutas (memorize — valem sempre)

1. **Minor Units:** todo valor monetário é `long` em centavos — nunca `decimal`/`float`. `R$ 1,00 = 100`.
2. **UUID v7** para todas as chaves primárias novas (ordenação por tempo, performance no PostgreSQL).
3. **`DateTime.UtcNow`** sempre — nunca `DateTime.Now`.
4. **Nenhum setter público** nas entidades de domínio.
5. **Nunca lance exceção** para erros de negócio nos Handlers — use `ErrorOr`.
6. **Nunca use `goto`** — use flags, métodos auxiliares ou reestruture o fluxo de controle.
7. **Erro de negócio ≠ Erro de infraestrutura:** business error → falha direta sem retry; infra error → retry com backoff.
8. **RedLock obrigatório** ao atualizar saldos ou estados concorrentes — chave `{recurso}:{id}`.

## Fase Atual de Desenvolvimento

> Atualize esta seção a cada mudança de fase.

**Fase em andamento:** {fase atual}
**Referência:** [`TASKS.md`](TASKS.md)

## Atualização de Progresso

Ao concluir qualquer item do TASKS.md, atualize o arquivo imediatamente:
- Marque `[x]` no item concluído
- Atualize a tabela de visão geral da fase correspondente (⬜ → 🔄 se em andamento, ✅ se fase completa)
- Atualize a linha "Fase em andamento" neste CLAUDE.md quando uma fase for concluída

Faça isso como parte do commit de cada item — nunca deixe o TASKS.md desatualizado.

## Quando em Dúvida

- Não sabe se é Command ou Query? → Se modifica estado = Command. Se só lê = Query.
- Não sabe se precisa de `IAuthorizedRequest`? → Se é endpoint público (login, health) = não. Todo o resto = sim.
- Não sabe se precisa de RedLock? → Se modifica saldo ou estado concorrente = sim. Senão = não.
- Não sabe se cria teste unitário ou integração? → Handler = unitário. Controller/endpoint = integração.

## Anti-patterns (nunca faça isso)

- Lógica de negócio no Controller
- `throw new Exception()` para erros de negócio (use ErrorOr)
- `DateTime.Now` em qualquer lugar (use `DateTime.UtcNow`)
- Query sem `.AsNoTracking()`
- Entidade exposta na response (use DTO)
- `*ngIf` ou `*ngFor` (use @if / @for)
- `@Input()` ou `@Output()` (use `input()` / `output()`)
- `any` em TypeScript (use tipo explícito ou `unknown`)

## Commits

- **Commit automático:** ao finalizar qualquer tarefa, **sempre crie um commit**. Não espere o usuário pedir.
- **Conventional Commits** em português: `feat(examples): adiciona filtro por status`, `fix(auth): corrige validação de token expirado`.
