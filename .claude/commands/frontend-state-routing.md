# Gerenciamento de Estado e Roteamento

## Signals vs RxJS

| Cenário                                   | Usar        |
|-------------------------------------------|-------------|
| Estado de UI (loading, filtros, seleção)  | **Signals** |
| Estado compartilhado entre componentes    | **Signals** (via service) |
| Chamadas HTTP, streams de dados           | **RxJS**    |
| WebSockets, eventos em tempo real         | **RxJS**    |
| Combinação de múltiplos observables       | **RxJS**    |

---

## Store com Signals

Padrão: signals privados + getters readonly + computed para derivações + métodos para mutação.

> Template completo do Store disponível via `/frontend-new-feature` (Passo 3)

---

## app.routes.ts

```typescript
export const appRoutes: Routes = [
  {
    path: '',
    component: MainLayoutComponent,
    canActivate: [authGuard],
    children: [
      { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
      {
        path: 'dashboard',
        loadChildren: () =>
          import('./features/dashboard/dashboard.routes').then(m => m.dashboardRoutes),
        title: 'Dashboard',
      },
      {
        path: 'examples',
        loadChildren: () =>
          import('./features/examples/examples.routes').then(m => m.examplesRoutes),
        title: 'Exemplos',
      },
    ],
  },
  {
    path: 'login',
    loadComponent: () =>
      import('./features/auth/login/login.component').then(m => m.LoginComponent),
    title: 'Login',
  },
  {
    path: 'unauthorized',
    loadComponent: () =>
      import('./features/auth/unauthorized/unauthorized.component').then(m => m.UnauthorizedComponent),
    title: 'Acesso Negado',
  },
  { path: '**', redirectTo: 'dashboard' },
];
```

## {feature}.routes.ts

```typescript
export const examplesRoutes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./examples.component').then(m => m.ExamplesComponent),
    children: [
      {
        path: '',
        loadComponent: () =>
          import('./components/example-list/example-list.component')
            .then(m => m.ExampleListComponent),
        title: 'Lista de Exemplos',
      },
      {
        path: 'novo',
        loadComponent: () =>
          import('./components/example-form/example-form.component')
            .then(m => m.ExampleFormComponent),
        title: 'Novo Exemplo',
      },
      {
        path: ':id/detalhe',
        loadComponent: () =>
          import('./components/example-detail/example-detail.component')
            .then(m => m.ExampleDetailComponent),
        title: 'Detalhe do Exemplo',
      },
    ],
  },
];
```

---

## Regras de Roteamento

- Todas as features são lazy-loaded — nunca importar componente diretamente em `appRoutes`.
- Usar `withViewTransitions()` no `provideRouter`.
- Toda rota protegida usa `canActivate: [authGuard]` no nível do layout.
- Definir `title` em todas as rotas.

### Proteção por Role

Rotas que exigem role específica combinam `authGuard` + `roleGuard` com `data`:

```typescript
{
  path: 'admin-area',
  data: { requiredRole: 'Admin' as AppRole },
  canActivate: [authGuard, roleGuard],
  loadChildren: () =>
    import('./features/admin/admin.routes').then(m => m.adminRoutes),
  title: 'Administração',
}
```

O `roleGuard` lê `route.data['requiredRole']` e aplica a hierarquia de roles. Acesso insuficiente redireciona para `/unauthorized`.

---

## Signal ↔ RxJS Interop

| Função | Direção | Uso |
|--------|---------|-----|
| `toSignal()` | Observable → Signal | Respostas HTTP, dados assíncronos |
| `toObservable()` | Signal → Observable | Raro — quando precisa de operadores RxJS |

### Exemplo: HTTP como Signal no Store

```typescript
@Injectable({ providedIn: 'root' })
export class ExampleStore {
  private readonly http = inject(HttpClient);

  private readonly _refresh = signal(0);

  readonly items = toSignal(
    toObservable(this._refresh).pipe(
      switchMap(() => this.http.get<Example[]>('/api/v1/examples')),
    ),
    { initialValue: [] },
  );

  refresh(): void {
    this._refresh.update(v => v + 1);
  }
}
```

> `toSignal()` faz auto-subscribe e auto-unsubscribe. Sempre forneça `initialValue` para evitar `undefined`.

---

## effect()

Use `effect()` para side-effects que dependem de valores de signals.

### Quando usar

- Sincronizar filtro com `localStorage`
- Registrar analytics
- Logar mudanças de estado para debug

### Exemplo

```typescript
export class ExampleListComponent {
  readonly filtro = signal('');

  constructor() {
    effect(() => {
      localStorage.setItem('example-filtro', this.filtro());
    });
  }
}
```

### Regras

- **Nunca** escreva em signals dentro de `effect()` sem `allowSignalWrites`
- Se precisar escrever, passe a opção explicitamente:

```typescript
effect(() => {
  this.total.set(this.items().length);
}, { allowSignalWrites: true });
```

- Prefira `computed()` em vez de `effect()` + `set()` sempre que possível
- `effect()` roda no mínimo 1 vez (na criação) e depois a cada mudança dos signals lidos

---

## @defer

Carregamento lazy de componentes inline — sem necessidade de rotas separadas.

### Quando usar

- Componentes pesados (gráficos, editores, mapas)
- Conteúdo abaixo do fold
- Seções que o usuário pode nunca acessar

### Condições disponíveis

| Condição | Dispara quando |
|----------|---------------|
| `@defer (on viewport)` | Elemento entra no viewport |
| `@defer (on interaction)` | Usuário interage (click, focus) |
| `@defer (on idle)` | Browser está idle |
| `@defer (on timer(5s))` | Após tempo especificado |
| `@defer (when condition)` | Expressão booleana é `true` |

### Exemplo: gráfico pesado carregado ao entrar no viewport

```html
@defer (on viewport) {
  <app-revenue-chart [data]="chartData()" />
} @placeholder {
  <div class="card placeholder-glow" style="height: 300px">
    <div class="card-body">
      <span class="placeholder col-12 h-100"></span>
    </div>
  </div>
} @loading (minimum 300ms) {
  <div class="d-flex justify-content-center py-5">
    <div class="spinner-border text-primary"></div>
  </div>
} @error {
  <div class="alert alert-danger">Erro ao carregar gráfico.</div>
}
```

> Use `@placeholder` com skeleton Tabler para evitar layout shift. Use `@loading (minimum Xms)` para evitar flash de spinner.
