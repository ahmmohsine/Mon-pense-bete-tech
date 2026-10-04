# Angular Forms

Les Forms permettent de gérer les saisies utilisateur et de valider les données dans Angular.

Deux approches principales existent :

```text
Template-driven forms
Reactive forms
```

## Template-driven forms

La logique du formulaire est principalement définie dans le template.

Exemple :

```html
<input
  name="name"
  [(ngModel)]="name">
```

Le component possède :

```typescript
name = '';
```

Le template et le component sont liés.

## Two-way binding

Le mécanisme :

```html
[(ngModel)]="name"
```

permet une synchronisation dans les deux directions :

```text
Component
    ↕
Input
```

## Reactive Forms

Les Reactive Forms définissent davantage la structure du formulaire dans le code TypeScript.

Exemple :

```typescript
form = new FormGroup({
  name: new FormControl(''),
  email: new FormControl('')
});
```

Le template peut utiliser :

```html
<form [formGroup]="form">
  <input formControlName="name">
  <input formControlName="email">
</form>
```

## Validation

On peut ajouter des validateurs :

```typescript
name: new FormControl(
  '',
  Validators.required
)
```

Mental model :

```text
User input
    ↓
FormControl
    ↓
Validation
    ↓
FormGroup
    ↓
Submit
```

## Template-driven vs Reactive

```text
Template-driven
    → logique davantage dans le template

Reactive Forms
    → structure et logique davantage dans TypeScript
```

Les Reactive Forms sont particulièrement adaptées lorsque le formulaire devient complexe ou nécessite des validations et comportements dynamiques.

## À retenir

> Un formulaire Angular collecte, représente et valide les données saisies par l'utilisateur.
