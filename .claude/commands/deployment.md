# Deployment e CI/CD

## Objetivo

Templates e padroes para containerizacao, CI/CD com GitHub Actions e deployment da aplicacao .NET + Angular.

## Quando usar

- Criar Dockerfiles para backend e frontend
- Configurar pipeline de CI/CD
- Preparar ambiente de producao
- Adicionar health checks para orquestracao de containers

---

## Dockerfile — Backend (.NET 9)

```dockerfile
# backend/Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0-alpine AS build
WORKDIR /src

# Copiar csproj e restaurar dependencias (cache de layers)
COPY src/{NomeProjeto}.Domain/*.csproj src/{NomeProjeto}.Domain/
COPY src/{NomeProjeto}.Application/*.csproj src/{NomeProjeto}.Application/
COPY src/{NomeProjeto}.Infrastructure/*.csproj src/{NomeProjeto}.Infrastructure/
COPY src/{NomeProjeto}.API/*.csproj src/{NomeProjeto}.API/
COPY src/{NomeProjeto}.Workers/*.csproj src/{NomeProjeto}.Workers/
RUN dotnet restore src/{NomeProjeto}.API/{NomeProjeto}.API.csproj

# Copiar todo o codigo e publicar
COPY src/ src/
RUN dotnet publish src/{NomeProjeto}.API/{NomeProjeto}.API.csproj \
    -c Release \
    -o /app/publish \
    --no-restore

# Runtime — imagem minima
FROM mcr.microsoft.com/dotnet/aspnet:9.0-alpine AS runtime
WORKDIR /app

# Seguranca: rodar como usuario nao-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=build /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health/live || exit 1

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "{NomeProjeto}.API.dll"]
```

---

## Dockerfile — Frontend (Angular + Nginx)

```dockerfile
# frontend/Dockerfile
FROM node:22-alpine AS build
WORKDIR /app

# Copiar package.json e instalar dependencias (cache de layers)
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

# Copiar codigo e fazer build
COPY . .
RUN npx ng build --configuration=production

# Runtime — Nginx
FROM nginx:alpine AS runtime

# Copiar config customizada do Nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copiar build do Angular
COPY --from=build /app/dist/{nome-projeto}/browser /usr/share/nginx/html

# Seguranca: rodar como usuario nao-root
RUN chown -R nginx:nginx /usr/share/nginx/html

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:80/ || exit 1

EXPOSE 80
```

### nginx.conf

```nginx
# frontend/nginx.conf
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1000;

    # Cache de assets estaticos
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA fallback — redireciona rotas para index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy para API (opcional — se servido no mesmo dominio)
    location /api/ {
        proxy_pass http://api:8080/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Correlation-Id $request_id;
    }
}
```

---

## GitHub Actions — CI/CD

### Workflow Completo

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '9.0.x'
  NODE_VERSION: '22'
  REGISTRY: ghcr.io
  API_IMAGE: ghcr.io/${{ github.repository }}/api
  WEB_IMAGE: ghcr.io/${{ github.repository }}/web

jobs:
  # ── Backend ──────────────────────────────
  backend-build-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restaurar dependencias
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -c Release

      - name: Testes unitarios
        run: dotnet test --no-build -c Release --logger "trx" --results-directory TestResults

      - name: Upload resultados de teste
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: backend-test-results
          path: backend/TestResults

  # ── Frontend ─────────────────────────────
  frontend-build-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Instalar dependencias
        run: npm ci

      - name: Lint
        run: npx ng lint

      - name: Testes unitarios
        run: npx ng test --watch=false --browsers=ChromeHeadless

      - name: Build producao
        run: npx ng build --configuration=production

  # ── Docker Build & Push ──────────────────
  docker-build:
    needs: [backend-build-test, frontend-build-test]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Login no GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build e push — API
        uses: docker/build-push-action@v5
        with:
          context: backend
          push: true
          tags: |
            ${{ env.API_IMAGE }}:latest
            ${{ env.API_IMAGE }}:${{ github.sha }}

      - name: Build e push — Web
        uses: docker/build-push-action@v5
        with:
          context: frontend
          push: true
          tags: |
            ${{ env.WEB_IMAGE }}:latest
            ${{ env.WEB_IMAGE }}:${{ github.sha }}

  # ── Deploy ───────────────────────────────
  deploy:
    needs: docker-build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            cd /opt/app
            docker compose -f docker-compose.prod.yml pull
            docker compose -f docker-compose.prod.yml up -d
            docker image prune -f
```

---

## docker-compose.prod.yml

```yaml
# docker-compose.prod.yml
services:
  api:
    image: ghcr.io/{org}/{repo}/api:latest
    restart: unless-stopped
    ports:
      - "8080:8080"
    env_file:
      - .env.production
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 5s
      retries: 3

  web:
    image: ghcr.io/{org}/{repo}/web:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      api:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    env_file:
      - .env.production
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3-management-alpine
    restart: unless-stopped
    env_file:
      - .env.production
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

### Diferencas entre dev e prod

| Aspecto | docker-compose.yml (dev) | docker-compose.prod.yml |
|---|---|---|
| Imagens | Oficiais (postgres, redis, etc.) | Inclui API e Web customizadas |
| Portas | Todas expostas (debug) | Apenas 80/443 e 8080 |
| Secrets | Inline no YAML | Via `.env.production` |
| Volumes | Bind mounts para hot reload | Named volumes apenas |
| Restart | `unless-stopped` | `unless-stopped` |
| Health checks | Basicos | Completos com dependencias |
| Redis | Sem senha | Com senha obrigatoria |

---

## Health Check Endpoints

Essenciais para orquestracao de containers (Docker, Kubernetes):

```csharp
// Configuracao (ver /logging-observability para detalhes)
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // apenas verifica se a app responde
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

| Endpoint | Uso | Container |
|---|---|---|
| `/health/live` | Liveness probe — app esta rodando? | `HEALTHCHECK` no Dockerfile |
| `/health/ready` | Readiness probe — app pode receber trafego? | Kubernetes readinessProbe |
| `/health` | Status completo de todas as dependencias | Monitoramento/dashboard |

---

## GitHub Actions Secrets

Secrets necessarios no repositorio (`Settings > Secrets and variables > Actions`):

| Secret | Descricao |
|---|---|
| `DEPLOY_HOST` | IP ou hostname do servidor de producao |
| `DEPLOY_USER` | Usuario SSH para deploy |
| `DEPLOY_SSH_KEY` | Chave privada SSH |
| `GITHUB_TOKEN` | Automatico — acesso ao GHCR |

> Nunca coloque secrets diretamente no workflow YAML. Use sempre `${{ secrets.NOME }}`.

---

## Checklist

- [ ] Dockerfile do backend com multi-stage build (sdk → aspnet)
- [ ] Dockerfile do frontend com multi-stage build (node → nginx)
- [ ] Ambos Dockerfiles rodam como usuario nao-root
- [ ] `nginx.conf` com SPA fallback e proxy para API
- [ ] GitHub Actions: build, test, docker push, deploy
- [ ] Testes executados antes do build Docker
- [ ] Docker push apenas na branch `main`
- [ ] `docker-compose.prod.yml` com health checks e dependencias
- [ ] `.env.production` no `.gitignore`
- [ ] Health check endpoints configurados (`/health/live`, `/health/ready`)
- [ ] Secrets configurados no GitHub Actions — nunca hardcoded
- [ ] Imagens tagueadas com `latest` e `sha` do commit
