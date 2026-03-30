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

Use estes comandos para carregar documentação detalhada sob demanda:

| Comando | Quando usar |
|---------|-------------|
| `/backend-new-feature` | Criar novo recurso/entidade no backend |
| `/backend-architecture` | Consultar arquitetura, CQRS flow, estrutura de pastas |
| `/backend-conventions` | Consultar convenções detalhadas de código backend |
| `/backend-templates` | Templates de infraestrutura (Pipeline, DI, Program.cs) |
| `/frontend-new-feature` | Criar nova feature no frontend Angular |
| `/frontend-conventions` | Consultar convenções detalhadas de código frontend |
| `/frontend-state-routing` | Consultar padrões de estado (Signals) e roteamento |
| `/frontend-tabler` | Consultar integração e classes do Tabler.io |

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

## Commit Automático

Ao finalizar qualquer tarefa solicitada pelo usuário, **sempre crie um commit** com as alterações realizadas. Não espere o usuário pedir — commitar faz parte da conclusão da tarefa.
