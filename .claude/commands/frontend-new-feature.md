# Criando uma Nova Feature — Frontend Angular

Ao adicionar qualquer nova feature, siga **sempre** esta sequência.
Não pule etapas nem inverta a ordem.

---

## Estrutura de Pastas da Feature

```
src/app/features/{feature}/
├── components/
│   ├── {feature}-list/
│   │   ├── {feature}-list.component.ts
│   │   ├── {feature}-list.component.html
│   │   ├── {feature}-list.component.scss
│   │   └── {feature}-list.component.spec.ts
│   └── {feature}-form/
│       ├── {feature}-form.component.ts
│       ├── {feature}-form.component.html
│       ├── {feature}-form.component.scss
│       └── {feature}-form.component.spec.ts
├── services/
│   └── {feature}.service.ts
├── models/
│   └── {feature}.model.ts
├── {feature}.store.ts          ← se estado compartilhado
├── {feature}.routes.ts
└── {feature}.component.ts      ← componente pai (router-outlet)
```

---

## 1 — Model

Crie `src/app/features/{feature}/models/{feature}.model.ts`

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
export enum ExampleStatus {
  ACTIVE   = 'Active',
  INACTIVE = 'Inactive',
}
```

---

## 2 — Service

Crie `src/app/features/{feature}/services/{feature}.service.ts`

```typescript
@Injectable({ providedIn: 'root' })
export class ExampleService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/examples`;

  listar(pagina = 1, itensPorPagina = 20): Observable<ExamplePaginated> {
    const params = new HttpParams().set('page', pagina).set('limit', itensPorPagina);
    return this.http.get<ExamplePaginated>(this.baseUrl, { params });
  }

  buscarPorId(id: string): Observable<Example> {
    return this.http.get<Example>(`${this.baseUrl}/${id}`);
  }

  criar(dto: CreateExampleDto): Observable<Example> {
    return this.http.post<Example>(this.baseUrl, dto);
  }

  atualizar(id: string, dto: Partial<CreateExampleDto>): Observable<Example> {
    return this.http.put<Example>(`${this.baseUrl}/${id}`, dto);
  }
}
```

- `@Injectable({ providedIn: 'root' })`
- Apenas chamadas HTTP via `HttpClient`
- Tipagem explícita em todos os retornos

---

## 3 — Store (se necessário)

Crie `src/app/features/{feature}/{feature}.store.ts`

Use quando o estado for compartilhado entre componentes da feature.

```typescript
@Injectable({ providedIn: 'root' })
export class ExampleStore {
  private readonly _items     = signal<Example[]>([]);
  private readonly _isLoading = signal(false);
  private readonly _filtro    = signal('');

  // Expor como readonly para fora
  readonly items     = this._items.asReadonly();
  readonly isLoading = this._isLoading.asReadonly();

  readonly itemsFiltrados = computed(() => {
    const filtro = this._filtro().toLowerCase();
    return this._items().filter(i =>
      i.name.toLowerCase().includes(filtro) ||
      i.description.toLowerCase().includes(filtro)
    );
  });

  readonly totalInativos = computed(() =>
    this._items().filter(i => !i.isActive).length
  );

  setItems(items: Example[]): void  { this._items.set(items); }
  setLoading(v: boolean): void      { this._isLoading.set(v); }
  setFiltro(v: string): void        { this._filtro.set(v); }

  adicionarItem(item: Example): void {
    this._items.update(lista => [...lista, item]);
  }
}
```

---

## 4 — Routes

Crie `src/app/features/{feature}/{feature}.routes.ts`

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

- Todas as rotas com `loadComponent`
- `title` definido em cada rota

---

## 5 — Componente Pai

Crie `src/app/features/{feature}/{feature}.component.ts`

Apenas `<router-outlet />` ou layout da feature.

---

## 6 — List Component (Smart)

Crie `src/app/features/{feature}/components/{feature}-list/`

```typescript
import { Component, signal, computed, inject, input, output, ChangeDetectionStrategy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-example-list',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './example-list.component.html',
  styleUrl: './example-list.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush, // SEMPRE OnPush
})
export class ExampleListComponent {
  // Injeção via inject() — nunca via construtor com @Inject
  private readonly exampleService = inject(ExampleService);

  // Inputs via input() signal — não usar @Input()
  readonly titulo = input<string>('Itens');
  readonly filtroAtivo = input<boolean>(false);

  // Outputs via output() — não usar @Output() EventEmitter
  readonly itemSelecionado = output<Example>();

  // Estado local com signals
  readonly items = signal<Example[]>([]);
  readonly isLoading = signal(false);
  readonly erro = signal<string | null>(null);

  // Computed para derivações
  readonly itemsFiltrados = computed(() =>
    this.filtroAtivo()
      ? this.items().filter(i => i.isActive)
      : this.items()
  );
  readonly totalRegistros = computed(() => this.itemsFiltrados().length);
}
```

### Template HTML com Tabler

```html
<!-- Usar @if, @for, @switch — NUNCA *ngIf, *ngFor -->
<div class="container-xl">
  <div class="page-header d-print-none">
    <div class="row align-items-center">
      <div class="col">
        <h2 class="page-title">{{ titulo() }}</h2>
        <div class="text-muted mt-1">{{ totalRegistros() }} registros</div>
      </div>
    </div>
  </div>

  @if (isLoading()) {
    <div class="d-flex justify-content-center py-5">
      <div class="spinner-border text-primary" role="status">
        <span class="visually-hidden">Carregando...</span>
      </div>
    </div>
  }

  @if (erro()) {
    <div class="alert alert-danger" role="alert">{{ erro() }}</div>
  }

  @if (!isLoading() && !erro()) {
    <div class="card">
      <div class="table-responsive">
        <table class="table table-vcenter card-table">
          <thead>
            <tr><th>Nome</th><th>Descrição</th><th>Status</th><th class="w-1"></th></tr>
          </thead>
          <tbody>
            @for (item of itemsFiltrados(); track item.id) {
              <tr>
                <td>{{ item.name }}</td>
                <td>{{ item.description }}</td>
                <td>
                  <span class="badge" [class]="item.isActive ? 'bg-success-lt' : 'bg-danger-lt'">
                    {{ item.isActive ? 'Ativo' : 'Inativo' }}
                  </span>
                </td>
                <td>
                  <button class="btn btn-sm btn-ghost-secondary"
                          (click)="itemSelecionado.emit(item)">Detalhar</button>
                </td>
              </tr>
            } @empty {
              <tr>
                <td colspan="4" class="text-center text-muted py-5">Nenhum registro encontrado.</td>
              </tr>
            }
          </tbody>
        </table>
      </div>
    </div>
  }
</div>
```

### Smart vs Dumb Components

| Tipo                      | Responsabilidade                                                    |
|---------------------------|---------------------------------------------------------------------|
| **Smart (Container)**     | Orquestra dados, chama serviços, gerencia estado                    |
| **Dumb (Presentational)** | Recebe via `input()`, emite via `output()`, sem injeção de serviços |

---

## 7 — Form Component

Crie `src/app/features/{feature}/components/{feature}-form/`

```typescript
@Component({ standalone: true, imports: [ReactiveFormsModule], ... })
export class ExampleFormComponent {
  private readonly fb = inject(FormBuilder);
  readonly isSubmitting = signal(false);

  readonly form = this.fb.group({
    name:        ['', Validators.required],
    description: ['', [Validators.required, Validators.maxLength(300)]],
  });

  get name()        { return this.form.controls.name; }
  get description() { return this.form.controls.description; }

  onSubmit(): void {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    this.isSubmitting.set(true);
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <div class="mb-3">
    <label class="form-label required">Nome</label>
    <input type="text" class="form-control"
           [class.is-invalid]="name.invalid && name.touched"
           formControlName="name">
    @if (name.invalid && name.touched) {
      <div class="invalid-feedback">
        @if (name.errors?.['required']) { Campo obrigatório. }
      </div>
    }
  </div>
  <div class="form-footer">
    <button type="submit" class="btn btn-primary w-100" [disabled]="isSubmitting()">
      @if (isSubmitting()) {
        <span class="spinner-border spinner-border-sm me-2"></span> Salvando...
      } @else { Salvar }
    </button>
  </div>
</form>
```

---

## 8 — Registrar Rota Lazy

Adicione em `src/app/app.routes.ts`:

```typescript
{
  path: '{feature}',
  loadChildren: () =>
    import('./features/{feature}/{feature}.routes').then(m => m.{feature}Routes),
  title: '{Feature}',
}
```

Se a feature exige role específica:

```typescript
{
  path: '{feature}',
  data: { requiredRole: 'Admin' as AppRole },
  canActivate: [authGuard, roleGuard],
  loadChildren: () =>
    import('./features/{feature}/{feature}.routes').then(m => m.{feature}Routes),
  title: '{Feature}',
}
```

---

## Checklist de Nova Feature

- [ ] Model criado com interfaces e enums tipados
- [ ] Service criado com todos os métodos HTTP necessários
- [ ] Store criado (se estado compartilhado entre componentes da feature)
- [ ] Routes com `loadComponent` e `title` em todas as rotas
- [ ] List component criado com `OnPush`, signals, tabela Tabler
- [ ] Form component criado com Reactive Forms e validação visual
- [ ] Rota lazy registrada em `app.routes.ts`
- [ ] Testes unitários do service com cobertura mínima de 80%
- [ ] Se a feature exige role específica: `data: { requiredRole: '...' }` + `roleGuard` na rota
- [ ] Navegação condicional no `SidebarComponent` atualizada (se nova entrada de menu)
- [ ] Acesso negado tratado: rota `/unauthorized` já disponível globalmente
