## Resumo

<!-- Descreva brevemente o que esta PR faz e por quê -->

## Tipo de mudança

- [ ] Nova feature
- [ ] Correção de bug
- [ ] Refatoração
- [ ] Documentação
- [ ] Infraestrutura / CI

## Checklist

- [ ] Código segue as convenções do projeto (CLAUDE.md)
- [ ] Testes unitários adicionados/atualizados
- [ ] Testes de integração adicionados/atualizados (se aplicável)
- [ ] Sem lógica de negócio no Controller
- [ ] Handlers usam ErrorOr (sem throw para erros de negócio)
- [ ] Queries usam `.AsNoTracking()`
- [ ] Entidades não expostas na response (usar DTO)
- [ ] Migration criada (se alterou o banco)

## Como testar

<!-- Passos para validar as mudanças -->
