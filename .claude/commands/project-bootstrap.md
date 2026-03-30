# Bootstrap do Projeto

Checklists para inicialização do projeto. Execute uma única vez ao criar um novo projeto baseado neste template.

---

## Backend — Setup Inicial

- [ ] Solution criada (`dotnet new sln -n {NomeProjeto}`)
- [ ] Projetos criados conforme nomenclatura:
  ```
  {NomeProjeto}.Domain
  {NomeProjeto}.Application
  {NomeProjeto}.Infrastructure
  {NomeProjeto}.API
  {NomeProjeto}.Workers
  {NomeProjeto}.Tests
  ```
- [ ] Referências entre projetos configuradas (Domain ← Application ← Infrastructure ← API)
- [ ] Pacotes NuGet instalados (ver `/backend-templates`)
- [ ] `<Nullable>enable</Nullable>` em todos os `.csproj`
- [ ] `AppDbContext` criado com `ApplyConfigurationsFromAssembly`
- [ ] `DependencyInjection.cs` criado em Application e Infrastructure
- [ ] Pipeline behaviors registrados (Logging → Authorization → Permission → Validation)
- [ ] `Program.cs` configurado com chamadas de alto nível (ver `/backend-templates`)
- [ ] `appsettings.Development.example.json` criado (sem segredos)
- [ ] Docker Compose com PostgreSQL + Redis + RabbitMQ
- [ ] `.gitignore` configurado (bin, obj, appsettings.Development.json)

---

## Frontend — Setup Inicial

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
