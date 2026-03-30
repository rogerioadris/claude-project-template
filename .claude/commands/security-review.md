# Revisão de Segurança

## Objetivo

Identificar e corrigir vulnerabilidades de segurança na aplicação. Foco em autenticação, OWASP Top 10, banco de dados e dependências.

## Quando usar

- Revisão de segurança antes de lançamento
- Investigação de incidente de segurança
- Auditoria de configurações de produção
- Implementação de autenticação e autorização

## Como executar

1. Identifique a superfície de ataque: API, frontend, banco, infra
2. Aplique os checklists abaixo por área
3. Documente cada vulnerabilidade com: severidade, impacto e remediação
4. Priorize por risco real (probabilidade x impacto)

## Severidade

- **Crítica:** exposição de dados de usuários, acesso root, vazamento de dados de pagamento
- **Alta:** bypass de autenticação, SQL injection, XSS armazenado
- **Média:** CSRF, rate limiting ausente, logs com dados sensíveis
- **Baixa:** headers de segurança faltando, mensagens de erro verbosas

---

## Autenticação e Secrets

### Senhas (BCrypt)

- Hash com BCrypt (custo >= 12) — nunca MD5/SHA1
- Mínimo 8 caracteres, sem restrição de caracteres especiais
- Bloqueio após N tentativas (lockout ou CAPTCHA)
- 2FA via TOTP (Google Authenticator) como padrão mínimo

### JWT

- Assine com RS256 (assimétrico) em produção — nunca HS256 com secret fraco
- Access token com exp curto (15 min a 1h)
- Refresh token com rotação: ao usar, invalide o anterior e emita novo
- Nunca coloque dados sensíveis no payload (é base64, não criptografia)
- Blacklist de tokens invalidados: Redis com TTL igual ao exp do token

### Secrets e Credenciais

**Nunca:**
- Commitar `.env` no git (use `.gitignore` + `git-secrets`)
- Hardcodar API keys no código-fonte
- Logar credenciais mesmo em debug
- Usar mesmas credenciais em dev e produção

**Onde guardar:**
- Produção: HashiCorp Vault, AWS Secrets Manager, Doppler
- CI/CD: variáveis de ambiente criptografadas (GitHub Actions Secrets)
- Dev local: `appsettings.Development.json` nunca commitado

**Rotação:**
- Rotacione secrets a cada 90 dias ou após saída de membro do time
- API keys de produção: uma por serviço/ambiente, nunca compartilhadas

---

## Web Security (OWASP Top 10)

### Broken Access Control

- Verifique permissões no servidor (`AuthorizationBehavior`), nunca apenas no frontend
- IDOR: valide se o usuário tem acesso ao recurso pelo ID no Handler
- Princípio do menor privilégio via `AppPolicies` e `AppPermissions`

### Cryptographic Failures

- HTTPS em tudo, sem exceção
- HSTS header com `includeSubDomains`
- Não exponha dados sensíveis em URLs (query strings ficam em logs)

### Injection

- SQL: EF Core com queries parametrizadas sempre — cuidado com `.FromSqlRaw()`
- XSS: Angular sanitiza por padrão, mas cuidado com `[innerHTML]` e `bypassSecurityTrust*`
- Command injection: nunca execute input do usuário como comando shell

### XSS

- Angular sanitiza HTML por padrão — nunca desabilite
- Content-Security-Policy: bloqueie inline scripts
- HttpOnly e Secure flags nos cookies de sessão

### CSRF

- SameSite=Strict ou Lax nos cookies
- Angular HttpClient envia X-XSRF-TOKEN automaticamente se configurado
- Verifique Origin/Referer em requests sensíveis

### Headers de Segurança Obrigatórios

```
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

### Rate Limiting

- Login: máximo 5 tentativas por minuto por IP
- APIs públicas: limite por IP e por token
- Endpoints de reset de senha: especialmente restritivos
- Implemente via middleware ASP.NET Core + Redis

---

## Banco de Dados e Dependências

### PostgreSQL

- Use sempre EF Core com queries parametrizadas — nunca concatene input
- Raw queries (`FromSqlRaw`) precisam de parâmetros explícitos
- Usuário da aplicação: apenas SELECT/INSERT/UPDATE/DELETE
- Usuário de migrations: separado, com ALTER/CREATE/DROP
- Read replicas: usuário somente leitura

### Dados Sensíveis (Pagamentos)

- **Nunca armazene dados de cartão** — use tokenização via gateway de pagamento
- Criptografe PII (CPF, dados pessoais) em repouso com AES-256
- Audit trail: registre quem acessou/modificou dados sensíveis (`IAuditService`)
- Backups criptografados

### Dependências

- `dotnet list package --vulnerable` e `npm audit`: rode em CI em cada PR
- Dependabot ou Renovate: automatize atualizações de segurança
- Lock files (`packages.lock.json`, `package-lock.json`): sempre commite
- Dependências de segurança: atualize imediatamente

---

## Checklist de Revisão Rápida

- [ ] Todos os inputs validados via FluentValidation?
- [ ] Autenticação verificada via `IAuthorizedRequest` em todas as rotas protegidas?
- [ ] Permissões granulares via `IPermissionRequiredRequest` onde necessário?
- [ ] HTTPS forçado com redirect de HTTP?
- [ ] Headers de segurança configurados no middleware?
- [ ] Logs não contêm dados sensíveis (senhas, tokens, cartões)?
- [ ] Dependências auditadas (`dotnet list package --vulnerable`, `npm audit`)?
- [ ] Dados de cartão NUNCA armazenados (tokenização via gateway)?
- [ ] Rate limiting implementado em login e endpoints públicos?
- [ ] Refresh token com rotação e blacklist via Redis?
- [ ] Secrets em `appsettings.Development.json` nunca commitados?
- [ ] RedLock usado em operações concorrentes de saldo?
