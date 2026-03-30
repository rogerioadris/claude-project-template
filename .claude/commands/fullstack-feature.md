# Criando uma Feature Fullstack (Backend + Frontend)

Use este guia ao criar um recurso completo que envolve backend e frontend.

---

## Ordem de Execução

**Sempre backend primeiro, frontend depois.** O contrato da API define o que o frontend consome.

### Fase 1 — Backend

Execute `/backend-new-feature` e siga os 9 passos na ordem:

1. Entity (Domain)
2. Interface do Repositório
3. Command + Handler + Validator
4. Query + Handler + DTO
5. Repositório (Infrastructure)
6. Configuração EF Core
7. Migration
8. Controller
9. Testes

**Antes de passar para o frontend**, garanta que:
- [ ] A API responde corretamente (POST cria, GET lista/detalha, DELETE remove)
- [ ] Os DTOs de response estão definidos (eles viram as interfaces do frontend)

### Fase 2 — Frontend

Execute `/frontend-new-feature` e siga os 8 passos na ordem:

1. Model (baseado nos DTOs do backend)
2. Service (HttpClient apontando para os endpoints criados)
3. Store (se necessário)
4. Routes
5. Componente Pai
6. List Component
7. Form Component
8. Registrar rota lazy

---

## Pontos de Integração

### DTO do backend → Interface do frontend

O DTO de response do backend define o contrato. Mapeie 1:1:

```
// Backend (C#)                          // Frontend (TypeScript)
ExampleDto(                              export interface Example {
  Guid Id,                     →           id: string;
  string Name,                 →           name: string;
  string Description,          →           description: string;
  DateTime CreatedAt           →           createdAt: string;
)                                        }
```

**Regras:**
- `Guid` → `string` (JSON serializa como string)
- `DateTime` → `string` (ISO 8601, parsear no frontend quando necessário)
- `long` (centavos) → `number` (converter para display no frontend: `valor / 100`)
- `camelCase` no JSON é garantido pelo backend (`JsonNamingPolicy.CamelCase`)

### Endpoints do backend → Service do frontend

```
// Backend Controller                    // Frontend Service
[HttpGet]    GetAll        →             listar(): Observable<PagedResult>
[HttpGet]    GetById       →             buscarPorId(id): Observable<Example>
[HttpPost]   Create        →             criar(dto): Observable<Example>
[HttpPut]    Update        →             atualizar(id, dto): Observable<Example>
[HttpDelete] Delete        →             excluir(id): Observable<void>
```

### Autorização

Se o backend usa `[Authorize(Policy = AppPolicies.AdminOnly)]`:
- O frontend deve ter `data: { requiredRole: 'Admin' }` + `roleGuard` na rota
- O sidebar deve esconder o menu para usuários sem a role

---

## Checklist Fullstack

- [ ] **Backend completo** — todos os 9 passos de `/backend-new-feature`
- [ ] **API testada** — endpoints respondendo corretamente
- [ ] **Frontend completo** — todos os 8 passos de `/frontend-new-feature`
- [ ] **Interfaces alinhadas** — DTOs do backend = interfaces do frontend
- [ ] **Autorização consistente** — policies do backend refletidas no roleGuard do frontend
- [ ] **Sidebar atualizada** — nova entrada de menu adicionada (se necessário)
