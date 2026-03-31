# Docker e Infraestrutura Local

## Objetivo

Configurar e gerenciar a infraestrutura local (PostgreSQL, Redis, RabbitMQ) via Docker Compose.

## Quando usar

- Setup inicial do ambiente de desenvolvimento
- Problemas com containers ou volumes
- Adicionar novo serviço à infra

---

## Docker Compose

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    container_name: app-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app_dev_password
      POSTGRES_DB: app_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: app-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: app-rabbitmq
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: app
      RABBITMQ_DEFAULT_PASS: app_dev_password
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
```

---

## appsettings por Ambiente

### appsettings.json (commitado — valores genéricos)

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "",
    "Redis": ""
  },
  "Jwt": {
    "Issuer": "",
    "Audience": "",
    "ExpirationMinutes": 60
  },
  "RabbitMq": {
    "Host": "",
    "Username": "",
    "Password": ""
  }
}
```

### appsettings.Development.json (NUNCA commitado)

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=app_db;Username=app;Password=app_dev_password",
    "Redis": "localhost:6379"
  },
  "Jwt": {
    "Issuer": "app-dev",
    "Audience": "app-dev",
    "SecretKey": "dev-secret-key-min-32-chars-long!!"
  },
  "RabbitMq": {
    "Host": "localhost",
    "Username": "app",
    "Password": "app_dev_password"
  }
}
```

### appsettings.Development.example.json (commitado — template)

Copiar para `appsettings.Development.json` e preencher valores reais.

---

## Comandos Comuns

```bash
# Subir tudo
docker compose up -d

# Ver status
docker compose ps

# Ver logs de um serviço
docker compose logs -f postgres
docker compose logs -f rabbitmq

# Parar tudo
docker compose down

# Parar e remover volumes (CUIDADO: perde dados)
docker compose down -v

# Restart de um serviço
docker compose restart redis
```

---

## Manutenção PostgreSQL

```bash
# Backup
docker compose exec postgres pg_dump -U app app_db > backup_$(date +%Y%m%d).sql

# Restore
docker compose exec -T postgres psql -U app app_db < backup_20260330.sql

# Acessar psql
docker compose exec postgres psql -U app app_db

# Ver tamanho do banco
docker compose exec postgres psql -U app app_db -c "SELECT pg_size_pretty(pg_database_size('app_db'));"
```

## Manutenção Redis

```bash
# Acessar redis-cli
docker compose exec redis redis-cli

# Ver chaves
docker compose exec redis redis-cli KEYS '*'

# Limpar cache (CUIDADO)
docker compose exec redis redis-cli FLUSHDB

# Monitorar comandos em tempo real
docker compose exec redis redis-cli MONITOR
```

## Manutenção RabbitMQ

```bash
# Acessar Management UI
open http://localhost:15672  # user: app / pass: app_dev_password

# Listar filas
docker compose exec rabbitmq rabbitmqctl list_queues

# Purgar uma fila (CUIDADO)
docker compose exec rabbitmq rabbitmqctl purge_queue nome_da_fila
```

---

## .gitignore (adicionar)

```gitignore
# Nunca commitar
appsettings.Development.json
appsettings.*.local.json
```

---

## Checklist

- [ ] `docker-compose.yml` na raiz do projeto
- [ ] Health checks em todos os serviços
- [ ] Volumes nomeados para persistência
- [ ] `appsettings.Development.example.json` commitado como template
- [ ] `appsettings.Development.json` no `.gitignore`
- [ ] Senhas diferentes entre dev e produção
