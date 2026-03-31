# Formularios Angular

## Objetivo

Padroes para formularios reativos tipados no Angular 21: validacao, mascaras, componentes reutilizaveis com ControlValueAccessor, integracao com Tabler.io e Signals.

## Quando usar

- Criar formularios de cadastro, edicao ou filtros
- Implementar validacoes customizadas (sync e async)
- Criar componentes de formulario reutilizaveis
- Aplicar mascaras (CPF, CNPJ, telefone, moeda)

---

## Reactive Forms Tipados

### Setup basico com NonNullableFormBuilder

```typescript
// src/app/features/customers/components/customer-form/customer-form.component.ts
import { Component, inject, ChangeDetectionStrategy, effect, output } from '@angular/core';
import { NonNullableFormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { cpfValidator } from '../../../../shared/validators/cpf.validator';
import { uniqueEmailValidator } from '../../validators/unique-email.validator';
import { CustomerCreate } from '../../models/customer.model';

@Component({
  selector: 'app-customer-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './customer-form.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class CustomerFormComponent {
  private readonly fb = inject(NonNullableFormBuilder);
  private readonly uniqueEmail = inject(uniqueEmailValidator);

  readonly submitted = output<CustomerCreate>();

  readonly form = this.fb.group({
    name: ['', [Validators.required, Validators.minLength(3), Validators.maxLength(200)]],
    email: ['', [Validators.required, Validators.email], [this.uniqueEmail.validate]],
    document: ['', [Validators.required, cpfValidator]],
    phone: ['', [Validators.required, Validators.pattern(/^\(\d{2}\)\s\d{4,5}-\d{4}$/)]],
    address: this.fb.group({
      street: ['', Validators.required],
      number: ['', Validators.required],
      city: ['', Validators.required],
      state: ['', [Validators.required, Validators.minLength(2), Validators.maxLength(2)]],
      zipCode: ['', [Validators.required, Validators.pattern(/^\d{5}-\d{3}$/)]],
    }),
  });

  onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.submitted.emit(this.form.getRawValue());
  }
}
```

> **Nunca use `any`** — o `NonNullableFormBuilder` garante tipagem completa. Todos os controles sao `FormControl<string>`, `FormControl<number>`, etc.

---

## Validacao no Template com Tabler.io

### Classes CSS do Tabler

| Classe | Uso |
|---|---|
| `is-invalid` | Aplicar no input quando invalido e tocado |
| `invalid-feedback` | Mensagem de erro abaixo do input |
| `is-valid` | Aplicar no input quando valido (opcional) |

### Template

```html
<!-- customer-form.component.html -->
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <div class="card">
    <div class="card-body">
      <div class="row g-3">

        <!-- Nome -->
        <div class="col-md-6">
          <label class="form-label required" for="name">Nome</label>
          <input
            id="name"
            type="text"
            class="form-control"
            formControlName="name"
            [class.is-invalid]="form.controls.name.invalid && form.controls.name.touched"
            placeholder="Nome completo"
          />
          @if (form.controls.name.errors?.['required'] && form.controls.name.touched) {
            <div class="invalid-feedback">Nome e obrigatorio.</div>
          }
          @if (form.controls.name.errors?.['minlength'] && form.controls.name.touched) {
            <div class="invalid-feedback">Nome deve ter no minimo 3 caracteres.</div>
          }
        </div>

        <!-- Email -->
        <div class="col-md-6">
          <label class="form-label required" for="email">E-mail</label>
          <input
            id="email"
            type="email"
            class="form-control"
            formControlName="email"
            [class.is-invalid]="form.controls.email.invalid && form.controls.email.touched"
            placeholder="email@exemplo.com"
          />
          @if (form.controls.email.errors?.['required'] && form.controls.email.touched) {
            <div class="invalid-feedback">E-mail e obrigatorio.</div>
          }
          @if (form.controls.email.errors?.['email'] && form.controls.email.touched) {
            <div class="invalid-feedback">E-mail invalido.</div>
          }
          @if (form.controls.email.errors?.['uniqueEmail'] && form.controls.email.touched) {
            <div class="invalid-feedback">Este e-mail ja esta em uso.</div>
          }
          @if (form.controls.email.pending) {
            <div class="form-text text-info">Verificando disponibilidade...</div>
          }
        </div>

        <!-- CPF com mascara -->
        <div class="col-md-6">
          <label class="form-label required" for="document">CPF</label>
          <input
            id="document"
            type="text"
            class="form-control"
            formControlName="document"
            [class.is-invalid]="form.controls.document.invalid && form.controls.document.touched"
            placeholder="000.000.000-00"
            appCpfMask
          />
          @if (form.controls.document.errors?.['required'] && form.controls.document.touched) {
            <div class="invalid-feedback">CPF e obrigatorio.</div>
          }
          @if (form.controls.document.errors?.['invalidCpf'] && form.controls.document.touched) {
            <div class="invalid-feedback">CPF invalido.</div>
          }
        </div>

      </div>
    </div>

    <div class="card-footer text-end">
      <button type="submit" class="btn btn-primary" [disabled]="form.invalid">
        Salvar
      </button>
    </div>
  </div>
</form>
```

---

## Validadores Customizados

### Validador Sincrono — CPF

```typescript
// src/app/shared/validators/cpf.validator.ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export const cpfValidator: ValidatorFn = (control: AbstractControl): ValidationErrors | null => {
  const value = control.value?.replace(/\D/g, '');
  if (!value || value.length !== 11) return { invalidCpf: true };

  // Verificar digitos iguais
  if (/^(\d)\1+$/.test(value)) return { invalidCpf: true };

  // Validar digitos verificadores
  const calcDigit = (slice: string, factor: number): number => {
    let sum = 0;
    for (const char of slice) {
      sum += parseInt(char, 10) * factor--;
    }
    const remainder = sum % 11;
    return remainder < 2 ? 0 : 11 - remainder;
  };

  const digit1 = calcDigit(value.slice(0, 9), 10);
  const digit2 = calcDigit(value.slice(0, 10), 11);

  if (digit1 !== parseInt(value[9], 10) || digit2 !== parseInt(value[10], 10)) {
    return { invalidCpf: true };
  }

  return null;
};
```

### Validador Assincrono — Email unico

```typescript
// src/app/features/customers/validators/unique-email.validator.ts
import { Injectable, inject } from '@angular/core';
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable, debounceTime, distinctUntilChanged, map, switchMap, of, catchError } from 'rxjs';
import { CustomerService } from '../services/customer.service';

@Injectable({ providedIn: 'root' })
export class uniqueEmailValidator {
  private readonly customerService = inject(CustomerService);

  readonly validate: AsyncValidatorFn = (
    control: AbstractControl
  ): Observable<ValidationErrors | null> => {
    if (!control.value) return of(null);

    return of(control.value).pipe(
      debounceTime(400),
      distinctUntilChanged(),
      switchMap(email => this.customerService.checkEmailAvailability(email)),
      map(isAvailable => (isAvailable ? null : { uniqueEmail: true })),
      catchError(() => of(null)) // em caso de erro de rede, nao bloquear
    );
  };
}
```

---

## FormArray — Campos Dinamicos

### Exemplo: lista de telefones

```typescript
// Dentro do componente
readonly form = this.fb.group({
  name: ['', Validators.required],
  phones: this.fb.array([this.createPhoneControl()]),
});

get phonesArray() {
  return this.form.controls.phones;
}

createPhoneControl() {
  return this.fb.group({
    type: ['mobile' as 'mobile' | 'home' | 'work'],
    number: ['', [Validators.required, Validators.pattern(/^\(\d{2}\)\s\d{4,5}-\d{4}$/)]],
  });
}

addPhone(): void {
  this.phonesArray.push(this.createPhoneControl());
}

removePhone(index: number): void {
  if (this.phonesArray.length > 1) {
    this.phonesArray.removeAt(index);
  }
}
```

### Template

```html
<div formArrayName="phones">
  @for (phone of phonesArray.controls; track $index) {
    <div class="row g-2 mb-2" [formGroupName]="$index">
      <div class="col-md-4">
        <select class="form-select" formControlName="type">
          <option value="mobile">Celular</option>
          <option value="home">Residencial</option>
          <option value="work">Comercial</option>
        </select>
      </div>
      <div class="col-md-6">
        <input
          type="text"
          class="form-control"
          formControlName="number"
          placeholder="(00) 00000-0000"
          appPhoneMask
        />
      </div>
      <div class="col-md-2">
        <button
          type="button"
          class="btn btn-outline-danger w-100"
          (click)="removePhone($index)"
          [disabled]="phonesArray.length === 1"
        >
          <i class="ti ti-trash"></i>
        </button>
      </div>
    </div>
  }
</div>

<button type="button" class="btn btn-outline-primary btn-sm" (click)="addPhone()">
  <i class="ti ti-plus"></i> Adicionar telefone
</button>
```

---

## Mascaras de Input

### Diretiva genérica de mascara

```typescript
// src/app/shared/directives/mask.directive.ts
import { Directive, ElementRef, HostListener, inject, input } from '@angular/core';
import { NgControl } from '@angular/forms';

@Directive({
  selector: '[appMask]',
  standalone: true,
})
export class MaskDirective {
  readonly appMask = input.required<string>(); // ex: '000.000.000-00'

  private readonly el = inject(ElementRef);
  private readonly control = inject(NgControl);

  @HostListener('input', ['$event.target.value'])
  onInput(value: string): void {
    const digits = value.replace(/\D/g, '');
    const mask = this.appMask();
    let result = '';
    let digitIndex = 0;

    for (const char of mask) {
      if (digitIndex >= digits.length) break;
      if (char === '0') {
        result += digits[digitIndex++];
      } else {
        result += char;
      }
    }

    this.el.nativeElement.value = result;
    this.control.control?.setValue(result, { emitEvent: false });
  }
}
```

### Mascaras especificas

```typescript
// src/app/shared/directives/cpf-mask.directive.ts
import { Directive, ElementRef, HostListener, inject } from '@angular/core';
import { NgControl } from '@angular/forms';

@Directive({ selector: '[appCpfMask]', standalone: true })
export class CpfMaskDirective {
  private readonly el = inject(ElementRef);
  private readonly control = inject(NgControl);

  @HostListener('input', ['$event.target.value'])
  onInput(value: string): void {
    const digits = value.replace(/\D/g, '').slice(0, 11);
    let formatted = digits;

    if (digits.length > 9) formatted = `${digits.slice(0,3)}.${digits.slice(3,6)}.${digits.slice(6,9)}-${digits.slice(9)}`;
    else if (digits.length > 6) formatted = `${digits.slice(0,3)}.${digits.slice(3,6)}.${digits.slice(6)}`;
    else if (digits.length > 3) formatted = `${digits.slice(0,3)}.${digits.slice(3)}`;

    this.el.nativeElement.value = formatted;
    this.control.control?.setValue(formatted, { emitEvent: false });
  }
}
```

### Mascara de moeda (centavos)

```typescript
// src/app/shared/directives/currency-mask.directive.ts
import { Directive, ElementRef, HostListener, inject } from '@angular/core';
import { NgControl } from '@angular/forms';

@Directive({ selector: '[appCurrencyMask]', standalone: true })
export class CurrencyMaskDirective {
  private readonly el = inject(ElementRef);
  private readonly control = inject(NgControl);

  @HostListener('input', ['$event.target.value'])
  onInput(value: string): void {
    const digits = value.replace(/\D/g, '');
    const cents = parseInt(digits || '0', 10);

    // Formatar como moeda brasileira
    const formatted = (cents / 100).toLocaleString('pt-BR', {
      style: 'currency',
      currency: 'BRL',
    });

    this.el.nativeElement.value = formatted;

    // O valor do controle e em CENTAVOS (long no backend)
    this.control.control?.setValue(cents, { emitEvent: false });
  }
}
```

> **Regra absoluta:** valores monetarios sempre em centavos (`long`). `R$ 1,00 = 100`. A mascara exibe formatado, mas o controle armazena centavos.

---

## ControlValueAccessor — Componente Reutilizavel

### Exemplo: Input com label e validacao integrada

```typescript
// src/app/shared/components/form-input/form-input.component.ts
import {
  Component, ChangeDetectionStrategy, inject, input, forwardRef
} from '@angular/core';
import {
  ControlValueAccessor, NG_VALUE_ACCESSOR, NgControl, ReactiveFormsModule
} from '@angular/forms';

@Component({
  selector: 'app-form-input',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <div class="mb-3">
      <label class="form-label" [class.required]="required()" [for]="inputId()">
        {{ label() }}
      </label>
      <input
        [id]="inputId()"
        [type]="type()"
        class="form-control"
        [class.is-invalid]="showError"
        [placeholder]="placeholder()"
        [value]="value"
        (input)="onInputChange($event)"
        (blur)="onTouched()"
      />
      @if (showError && errorMessage) {
        <div class="invalid-feedback">{{ errorMessage }}</div>
      }
    </div>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => FormInputComponent),
      multi: true,
    },
  ],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FormInputComponent implements ControlValueAccessor {
  private readonly ngControl = inject(NgControl, { optional: true, self: true });

  readonly label = input.required<string>();
  readonly inputId = input.required<string>();
  readonly type = input<string>('text');
  readonly placeholder = input<string>('');
  readonly required = input<boolean>(false);
  readonly errorMessages = input<Record<string, string>>({});

  value = '';
  onChange: (value: string) => void = () => {};
  onTouched: () => void = () => {};

  constructor() {
    if (this.ngControl) {
      this.ngControl.valueAccessor = this;
    }
  }

  get showError(): boolean {
    const control = this.ngControl?.control;
    return !!control && control.invalid && control.touched;
  }

  get errorMessage(): string {
    const errors = this.ngControl?.control?.errors;
    if (!errors) return '';

    const messages = this.errorMessages();
    const firstKey = Object.keys(errors)[0];
    return messages[firstKey] ?? `Erro: ${firstKey}`;
  }

  writeValue(value: string): void {
    this.value = value ?? '';
  }

  registerOnChange(fn: (value: string) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  onInputChange(event: Event): void {
    const value = (event.target as HTMLInputElement).value;
    this.value = value;
    this.onChange(value);
  }
}
```

### Uso

```html
<app-form-input
  label="Nome"
  inputId="name"
  formControlName="name"
  [required]="true"
  placeholder="Nome completo"
  [errorMessages]="{ required: 'Nome e obrigatorio', minlength: 'Minimo 3 caracteres' }"
/>
```

---

## Signals e Formularios

### effect() — Persistir formulario no localStorage

```typescript
import { effect } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';

export class CustomerFormComponent {
  private readonly storageKey = 'customer-form-draft';

  readonly formValues = toSignal(this.form.valueChanges, { initialValue: this.form.value });

  constructor() {
    // Restaurar rascunho
    const draft = localStorage.getItem(this.storageKey);
    if (draft) {
      this.form.patchValue(JSON.parse(draft));
    }

    // Salvar rascunho automaticamente
    effect(() => {
      const values = this.formValues();
      localStorage.setItem(this.storageKey, JSON.stringify(values));
    });
  }

  clearDraft(): void {
    localStorage.removeItem(this.storageKey);
  }
}
```

### toSignal() / toObservable() — Interop

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { signal, computed } from '@angular/core';

// Observable → Signal
readonly searchResults = toSignal(
  this.form.controls.search.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => this.service.search(term))
  ),
  { initialValue: [] }
);

// Signal → Observable (para usar em pipes RxJS)
readonly selectedFilter = signal<string>('active');

readonly filteredItems$ = toObservable(this.selectedFilter).pipe(
  switchMap(filter => this.service.getByStatus(filter))
);
```

---

## @defer para Formularios Pesados

Carregar formularios complexos sob demanda:

```html
<!-- No template do componente pai -->
@defer (on interaction) {
  <app-customer-form (submitted)="onCustomerCreated($event)" />
} @placeholder {
  <button class="btn btn-primary">
    <i class="ti ti-plus"></i> Novo Cliente
  </button>
} @loading (minimum 200ms) {
  <div class="d-flex align-items-center gap-2">
    <div class="spinner-border spinner-border-sm"></div>
    Carregando formulario...
  </div>
}
```

> Use `@defer` para formularios que nao sao exibidos imediatamente (modais, abas secundarias, acordeoes).

---

## Checklist

- [ ] Formularios usam `NonNullableFormBuilder` — nunca `FormBuilder`
- [ ] Todos os controles sao tipados — nenhum `any`
- [ ] Validacoes exibidas com classes Tabler (`is-invalid`, `invalid-feedback`)
- [ ] `markAllAsTouched()` chamado no submit quando formulario invalido
- [ ] Validadores customizados criados como funcoes puras (sync) ou services (async)
- [ ] Validadores async usam `debounceTime` para evitar chamadas excessivas
- [ ] `FormArray` com `track $index` no `@for`
- [ ] Mascaras implementadas via diretivas standalone
- [ ] Valores monetarios em centavos no controle — formatados apenas na exibicao
- [ ] `ControlValueAccessor` para componentes de formulario reutilizaveis
- [ ] `effect()` para side-effects (localStorage, analytics)
- [ ] `@defer` para formularios pesados nao exibidos imediatamente
- [ ] `ChangeDetectionStrategy.OnPush` em todos os componentes de formulario
