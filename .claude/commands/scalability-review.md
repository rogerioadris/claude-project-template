# Revisão de Escalabilidade

## Objetivo

Avaliar e implementar estratégias de escalabilidade. Cobre APIs, PostgreSQL, Redis, RabbitMQ e infraestrutura.

## Quando usar

- Sistema apresentando lentidão sob carga
- Preparar arquitetura para crescimento de 10x
- Revisar gargalos antes de campanha ou lançamento
- Planejar capacidade de processamento de pagamentos

## Como executar

1. Identifique o gargalo atual (CPU? DB? I/O? rede?)
2. Aplique a seção correspondente abaixo
3. Proponha solução com menor impacto de implementação primeiro
4. Documente limites atuais e projeção após otimização
5. Defina métricas de sucesso antes de implementar

---

## API e Serviços (.NET)

### Princípios

- **Stateless:** serviços não guardam estado entre requisições
- **Idempotência:** requisições repetidas têm o mesmo efeito (crítico para pagamentos)
- **Contratos claros:** versionamento de API via `/v1/`, `/v2/`

### Rate Limiting

- Por IP e por usuário autenticado
- Sliding window (mais preciso) em vez de fixed window
- Retorne 429 com header `Retry-After`
- Implemente via middleware ASP.NET Core + Redis (lua script)

### Circuit Breaker

- Evita cascata de falhas quando serviço downstream cai
- Estados: Closed → Open → Half-Open
- Use **Polly** (.NET): retry, circuit breaker, timeout, bulkhead
- Aplique em chamadas a gateways de pagamento e APIs externas

### Load Balancing

- Round robin: cargas uniformes
- Least connections: requisições de duração variável
- Health checks: remova instâncias doentes automaticamente

---

## PostgreSQL

### Identificando o Gargalo

- Queries lentas: ative slow query log (> 100ms = alerta)
- Connections esgotadas: use PgBouncer (transaction mode)
- CPU alta: índices faltando ou queries ineficientes
- I/O alto: volume de dados crescendo mais rápido que o hardware

### Índices

**Quando criar:**
- Colunas usadas em WHERE, ORDER BY, JOIN
- Foreign keys sem índice automático
- Combinações de colunas usadas juntas em filtros

**Quando não criar:**
- Tabelas pequenas (< 10k linhas)
- Colunas com poucos valores distintos (ex: status com 3 opções)
- Índices demais degradam escrita (todo INSERT/UPDATE atualiza índices)

### Read Replicas

- Separe queries de leitura (relatórios, listagens) para réplica
- Escreva sempre no primário
- Atenção ao replication lag em dados críticos (saldos!)

### Particionamento

- Particione tabelas grandes por data (logs, transações, audit trail)
- Facilita DELETE de dados antigos (drop partition)
- Melhora performance de queries com filtro de data

### Conexões

- PgBouncer em transaction mode: centenas de clientes, dezenas de conexões reais
- Nunca abra conexão sem fechar (connection leak mata o banco)
- Configure `MaxPoolSize` no connection string do EF Core

---

## Redis (Cache e Lock)

### Estratégias de Cache

**Cache-aside (Lazy Loading):**
1. App busca no cache → miss → busca no banco → salva no cache
2. Vantagem: cache só tem o que foi acessado
3. Desvantagem: primeiro acesso lento

**Write-through:**
1. App escreve no cache e no banco simultaneamente
2. Vantagem: cache sempre atualizado
3. Desvantagem: overhead em escrita

### TTL Recomendado

| Dado | TTL |
|------|-----|
| Sessão / Token blacklist | Igual ao exp do JWT |
| Resposta de API externa | Conforme frequência de mudança |
| Contadores e rankings | 1 a 5 minutos |
| Configurações do sistema | 5 a 60 minutos |
| Cache de listagens | 1 a 5 minutos |

### Rate Limiting via Redis

- Chave: `rate:{user_id}` ou `rate:{ip}`
- Estrutura: INCR + EXPIRE (sliding window com lua script)

### RedLock (já obrigatório no projeto)

- Chave: `{recurso}:{id}` — ex: `balance:550e8400-...`
- TTL do lock: tempo máximo da operação + margem
- Retry com backoff exponencial se lock não adquirido

---

## RabbitMQ (Filas)

### Quando usar fila

- Processamento de pagamentos assíncrono
- Envio de emails e notificações
- Integração com gateways externos lentos
- Qualquer operação > 500ms que não precisa de resposta imediata
- Reconciliação e liquidação em lote

### Boas Práticas

- **Dead letter queue:** defina max retries e DLQ para mensagens que falham
- **Idempotência:** consumers devem ser idempotentes (reprocessamento seguro)
- **Monitoramento:** backlog crescendo = consumer lento ou travado
- **Prefetch count:** ajuste para evitar que um consumer monopolize mensagens
- **Retry com backoff:** erro de infra → retry com backoff; erro de negócio → falha direto (sem retry)

---

## Infraestrutura

### Auto-scaling Horizontal

- Melhor para aplicações stateless (.NET API)
- Métricas de trigger: CPU > 70%, req/s, latência p99
- Cooldown: 2-3 min entre escalas para evitar flapping

### Kubernetes (se aplicável)

- HPA: escala pods por CPU/memória
- Resource requests/limits: defina sempre
- PodDisruptionBudget: garante disponibilidade em updates

### CDN

- Assets estáticos Angular (JS, CSS): sempre via CDN
- Cache-Control correto para assets com hash no nome

### Monitoramento

- SLOs: ex. p99 latência < 500ms, disponibilidade > 99.9%
- Alertas proativos: avise quando tendência indica estouro em 24h
- Load testing: k6 ou Locust antes de lançamentos
- Capacity planning: projete 3 meses e provisione com antecedência

---

## Checklist de Revisão

- [ ] Queries críticas têm índices apropriados?
- [ ] PgBouncer configurado para connection pooling?
- [ ] Cache Redis implementado para dados frequentemente lidos?
- [ ] Rate limiting ativo em endpoints públicos e de login?
- [ ] Circuit breaker (Polly) em chamadas a serviços externos?
- [ ] Filas RabbitMQ para operações assíncronas pesadas?
- [ ] Consumers idempotentes com DLQ configurada?
- [ ] Assets Angular servidos via CDN?
- [ ] Monitoramento de latência e throughput ativo?
- [ ] Particionamento em tabelas de alto volume (transações, logs)?
