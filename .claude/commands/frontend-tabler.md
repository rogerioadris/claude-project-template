# Integração Tabler.io

**Documentação Oficial (UI):** [https://docs.tabler.io/ui](https://docs.tabler.io/ui) - Consulte para saber como utilizar os componentes e o template do Tabler.

## Instalação

```bash
npm install @tabler/core @tabler/icons
```

---

## Configuração de Estilos

```scss
// styles/styles.scss — ordem importa
@import 'variables';                        // 1. Variáveis customizadas ANTES do Tabler
@import '@tabler/core/src/scss/tabler';     // 2. Tabler Core
@import 'typography';                       // 3. Customizações locais
@import 'mixins';
```

```scss
// styles/_variables.scss
$primary:   #1a56db;
$secondary: #6c757d;
$success:   #2fb344;
$warning:   #f59f00;
$danger:    #d63939;

$font-family-sans-serif: 'Inter', system-ui, -apple-system, sans-serif;
$font-size-base: 0.875rem;

$sidebar-width:    15rem;
$header-height:    3.5rem;
$border-radius:    0.375rem;
$border-radius-lg: 0.5rem;
```

---

## Uso de Ícones

```html
<!-- SVG inline (recomendado para ícones críticos) -->
<svg xmlns="http://www.w3.org/2000/svg" class="icon icon-tabler icon-tabler-home"
     width="24" height="24" viewBox="0 0 24 24"
     stroke-width="2" stroke="currentColor" fill="none"
     stroke-linecap="round" stroke-linejoin="round"
     aria-hidden="true">
  <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
  <path d="M5 12l-2 0l9 -9l9 9l-2 0"/>
  <path d="M5 12v7a2 2 0 0 0 2 2h10a2 2 0 0 0 2 -2v-7"/>
  <path d="M9 21v-6a2 2 0 0 1 2 -2h2a2 2 0 0 1 2 2v6"/>
</svg>

<!-- Via classe CSS -->
<i class="ti ti-home" aria-hidden="true"></i>
```

---

## Classes Tabler Essenciais

```html
<!-- Layout base (Navbar Horizontal Dark) -->
<div class="page">
  <header class="navbar navbar-expand-sm navbar-dark d-print-none">
    <div class="container-xl">
       <!-- logo, toggler, menu de navegação -->
    </div>
  </header>
  <div class="page-wrapper">
    <div class="page-header d-print-none">
      <div class="container-xl">...</div>
    </div>
    <div class="page-body">
      <div class="container-xl">...</div>
    </div>
  </div>
</div>

<!-- Card de KPI -->
<div class="card card-sm">
  <div class="card-body">
    <div class="row align-items-center">
      <div class="col-auto">
        <span class="bg-green text-white avatar"><i class="ti ti-chart-bar"></i></span>
      </div>
      <div class="col">
        <div class="font-weight-medium">1.248 Registros</div>
        <div class="text-muted">Total no sistema</div>
      </div>
    </div>
  </div>
</div>

<!-- Badges de status -->
<span class="badge bg-success-lt">Ativo</span>
<span class="badge bg-danger-lt">Inativo</span>
<span class="badge bg-warning-lt">Pendente</span>
<span class="badge bg-secondary-lt">Arquivado</span>

<!-- Botões -->
<button class="btn btn-primary">Salvar</button>
<button class="btn btn-secondary">Cancelar</button>
<button class="btn btn-ghost-danger">Excluir</button>
<button class="btn btn-sm btn-outline-primary">Detalhar</button>
```

---

## Layout Principal (componente)

```typescript
@Component({
  selector: 'app-main-layout',
  standalone: true,
  imports: [RouterOutlet, NavbarComponent],
  template: `
    <div class="page">
      <app-navbar />
      <div class="page-wrapper">
        <div class="page-body">
          <router-outlet />
        </div>
        <footer class="footer footer-transparent d-print-none">
          <div class="container-xl">
            <ul class="list-inline list-inline-dots mb-0">
              <li class="list-inline-item">&copy; {{ anoAtual() }} {NomeProjeto}</li>
            </ul>
          </div>
        </footer>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class MainLayoutComponent {
  readonly anoAtual = signal(new Date().getFullYear());
}
```

---

## Acessibilidade

- Ícones decorativos: `aria-hidden="true"` obrigatório.
- Botões sem texto visível: `aria-label` obrigatório.
- Tabelas: `<caption>` ou `aria-label` descritivo.
- Inputs sempre associados ao label via `for/id` ou `aria-labelledby`.
