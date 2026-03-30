# Convenções Obrigatórias — Frontend Angular

## Regras de Componentes

- **Standalone components** — NgModules são proibidos.
- `ChangeDetectionStrategy.OnPush` em **todos** os componentes, sem exceção.
- Injeção via `inject()` — nunca via construtor com `@Inject`.
- Inputs via `input()` signal, outputs via `output()` — não usar `@Input()` / `@Output()`.
- **Signals** para estado local e compartilhado; RxJS apenas para streams HTTP e WebSockets.
- **Lazy loading** em todos os feature modules via `loadChildren` / `loadComponent`.
- Chamadas HTTP apenas em **services**, nunca em componentes.
- Variáveis de ambiente sempre via `environment.ts`, nunca hardcoded.

## Regras de Template

- Usar `@if`, `@for`, `@switch` — nunca `*ngIf`, `*ngFor`.
- `track` obrigatório no `@for`.
- Nunca usar `any` — preferir `unknown` quando o tipo não é conhecido.
- `strict: true` no `tsconfig.json`.

## Prefixo de Seletor

- Prefixo `app-` em todos os seletores (configurar em `angular.json`).
- Exemplo: `app-example-list`, `app-login-form`.

## Commits

- Padrão **Conventional Commits** em português: `feat(examples): adiciona filtro por status`.

## Cobertura de Testes

- Mínimo de **80%** de cobertura para serviços.

---

## Nomenclatura de Arquivos

| Tipo        | Padrão                      | Exemplo                          |
|-------------|-----------------------------|----------------------------------|
| Componente  | `kebab-case.component.ts`   | `example-list.component.ts`      |
| Serviço     | `kebab-case.service.ts`     | `example.service.ts`             |
| Guard       | `kebab-case.guard.ts`       | `auth.guard.ts`                  |
| Interceptor | `kebab-case.interceptor.ts` | `auth.interceptor.ts`            |
| Model       | `kebab-case.model.ts`       | `example.model.ts`               |
| Rotas       | `kebab-case.routes.ts`      | `examples.routes.ts`             |
| Store       | `kebab-case.store.ts`       | `example.store.ts`               |
| Spec        | `kebab-case.spec.ts`        | `example.service.spec.ts`        |

---

## Models

```typescript
// Interfaces sem prefixo "I" — nunca IExample
export interface Example {
  id: string;
  name: string;
  description: string;
  isActive: boolean;
  createdAt: string;
}

export interface ExamplePaginated {
  data: Example[];
  total: number;
  page: number;
  lastPage: number;
}

export interface CreateExampleDto {
  name: string;
  description: string;
}

// Enums com valores SCREAMING_SNAKE_CASE
export enum AppRole {
  ADMIN     = 'Admin',
  EMPLOYEE  = 'Employee',
}
```

---

## Configuração de Infraestrutura

### environment.ts

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:5000/api'
};
```

### app.config.ts

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(appRoutes, withViewTransitions()),
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor])
    ),
  ],
};
```

### auth.interceptor.ts

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  if (token) req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  return next(req);
};
```

---

## Estrutura de Pastas

```
src/
├── app/
│   ├── core/                        # Singleton: guards, interceptors, serviços globais
│   │   ├── guards/
│   │   ├── interceptors/
│   │   ├── services/
│   │   ├── models/
│   │   └── core.providers.ts
│   │
│   ├── shared/                      # Componentes "burros", pipes, diretivas
│   │   ├── components/
│   │   ├── directives/
│   │   ├── pipes/
│   │   └── utils/
│   │
│   ├── layout/                      # Shell da aplicação
│   │   ├── main-layout/
│   │   ├── sidebar/
│   │   ├── navbar/
│   │   └── footer/
│   │
│   ├── features/                    # Lazy-loaded por padrão
│   │   └── {feature}/
│   │
│   ├── app.component.ts
│   ├── app.config.ts
│   └── app.routes.ts
│
├── assets/
├── environments/
└── styles/
```

### Regras da Estrutura

- **`core/`** — importado apenas uma vez em `app.config.ts`. Nunca importar em features.
- **`shared/`** — sem lógica de negócio. Apenas componentes "burros" e utilitários.
- **`features/`** — cada feature é lazy-loaded. Nunca importar diretamente em `appRoutes`.
- Cada componente vive em **sua própria pasta** com `.ts`, `.html`, `.scss`, `.spec.ts`.
- Stores ficam dentro da feature (se local) ou em `core/` (se global).
