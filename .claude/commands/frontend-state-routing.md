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
