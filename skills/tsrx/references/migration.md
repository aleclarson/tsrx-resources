# Migrating from legacy `tsrx-preact`

Legacy `tsrx-preact` was a Preact-specific statement-template language. Current TSRX is TypeScript + JSX with opt-in `@` templates and multiple compiler targets.

## Quick map

| Legacy syntax | Current syntax |
| --- | --- |
| `component Name(props) { ... }` | `function Name(props) @{ ... }` |
| `const Name = component (...) => { ... }` | `const Name = (...) => @{ ... }` |
| JSX statements throughout component bodies | Standard JSX plus `@{ ... }` statement containers |
| `"Text"` text nodes | Standard JSX text: `Text` |
| `<tsrx>...</tsrx>` expression wrapper | Ordinary JSX, or `@{ ... }` for local statements/control flow |
| `if (...) { ... } else { ... }` in templates | `@if (...) { ... } @else { ... }` |
| `for (... of ...; index i; key expr)` | `@for (... of ...; index i; key expr)` |
| Empty-list handling around loops | `@for (...) { ... } @empty { ... }` |
| `switch` / `case` / `default` / `break` | `@switch` / `@case:` / `@default:` with selected-branch output |
| `try` / `pending` / `catch` | `@try` / `@pending` / `@catch` |
| Bare guard `return;` after rendered fallback | `return <Fallback />` or `return null` in top-level `@{ ... }` function bodies |

## Component shape

Legacy components used the `component` keyword and statement-position JSX:

```tsrx
export component Button({ label }: { label: string }) {
  <button>"Save: "{label}</button>
}
```

Current components use normal function or arrow declarations with an `@{ ... }` body:

```tsx
export function Button({ label }: { label: string }) @{
  <button>Save: {label}</button>
}

const Button = ({ label }: { label: string }) => @{
  <button>Save: {label}</button>
}
```

The top-level function body may use early `return`; otherwise its final JSX-producing statement is the component output.

## Text and local statements

Legacy static text used quoted text nodes:

```tsrx
<p>"Hello, "{name}"!"</p>
```

Current TSRX keeps JSX text rules:

```tsx
<p>Hello, {name}!</p>
```

Legacy element bodies could contain local declarations directly:

```tsrx
<div>
  const greeting = `Hello, ${name}`;
  <p>{greeting}</p>
</div>
```

Current JSX child positions use statement containers or `@` control blocks:

```tsx
<div>
  @{
    const greeting = `Hello, ${name}`;
    <p>{greeting}</p>
  }
</div>
```

## Expression-position markup

Legacy used `<tsrx>` when markup appeared in assignment, helper returns, prop values, or render props:

```tsrx
const title = <tsrx><span>"Settings"</span></tsrx>;
```

Current code can use ordinary JSX for plain markup:

```tsx
const title = <span>Settings</span>;
```

Use `@{ ... }` in expression position when the value needs local statements or TSRX control flow:

```tsx
const title = @{
  const label = section.name.toUpperCase();
  <span>{label}</span>
};
```

## Control flow

Current template control flow is explicitly prefixed with `@`.

```tsx
@if (status === 'loading') {
  <Spinner />
} @else {
  <Content />
}
```

```tsx
@for (const item of items; index i; key item.id) {
  <Item item={item} index={i} />
} @empty {
  <Empty />
}
```

```tsx
@switch (type) {
  @case 'warning':
  @case 'error': {
    <Alert />
  }
  @default: {
    <Info />
  }
}
```

```tsx
@try {
  <Panel />
} @pending {
  <Skeleton />
} @catch (error, reset) {
  <ErrorView error={error} onRetry={reset} />
}
```

Each branch/body may run local JavaScript statements, then finishes with JSX. `@for` bodies exclude `break` and `continue`; `@switch` branches use selected output with `break` omitted.

## Guards and hooks

Legacy guard clauses rendered fallback JSX, then used bare `return;`:

```tsrx
component Dashboard({ user }: { user: User | null }) {
  if (!user) {
    <p>"Please sign in."</p>
    return;
  }

  <h1>"Welcome, "{user.name}</h1>
}
```

Current top-level TSRX function bodies can return fallback output directly:

```tsx
function Dashboard({ user }: { user: User | null }) @{
  if (!user) return <p>Please sign in.</p>;
  <h1>Welcome, {user.name}</h1>
}
```

Current TSRX also supports compiler-extracted conditional hooks in conditionals, loops, switches, and paths after early returns when the branch is statically extractable.

## Styling and reactivity additions

Current TSRX adds native CSS and lazy destructuring:

```tsx
const styles = <style>
  .card { border: 1px solid #ccc; }
</style>;

<div className={styles.card} />
```

```tsx
function UserCard(&{ user, theme }) @{
  <article className={theme}>{user.name}</article>
}
```

JSX-child `<style>` blocks are scoped CSS. Variable-initializer `<style>` blocks create typed class maps. `&` destructuring preserves fine-grained reactivity for Solid, Vue, and Ripple targets.

## Mostly unchanged

- TypeScript imports, types, generics, and expressions keep TypeScript semantics.
- JSX elements, fragments, expression containers, and children keep JSX semantics.
- Attribute shorthand remains available: `<Input {value} {onInput} />`.
- Preact remains a supported target; TSRX is now target-agnostic across React, Preact, Solid, Vue, and Ripple.
