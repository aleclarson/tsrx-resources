# Introducing TSRX: A Syntax-Level Extension for TypeScript and JSX

TSRX is a strict, compiler-agnostic superset of JSX and TypeScript with official support for **React, Preact, Solid, Vue, and Ripple** compiler targets. It introduces dedicated syntax for control flow, inline blocks, component architecture, framework-aware hooks, and native styling without altering existing JavaScript or JSX semantics.

With one explicit exception regarding standard `<style>` tags, every feature introduced by TSRX is entirely optional. Valid JSX and TypeScript code remains valid when pasted directly into a `.tsrx` file.

A core philosophical pillar of TSRX is **locality**: layout structures, styling, and their direct data or logic dependencies should live as close to each other as possible. TSRX is designed with deliberate architectural restraint to prevent this locality from turning into spaghetti code. The creators of TSRX explicitly recommend splitting deeply nested structures into dedicated child components to keep complexity manageable.

---

## Feature Summary

TSRX introduces the following syntax-level features:

* `@` templates: JavaScript statements inside JSX-producing templates
* `@if`: structured conditional rendering
* `@for`: iterable rendering with optional `index`, `key`, and `@empty`
* `@switch`: non-fallthrough branch selection
* `@try`: async and error boundary syntax
* `@{ ... }`: inline statement containers
* Function body `@{ ... }`: JSX-producing component bodies
* Conditional hooks: compile-time hook-safe extraction
* Native `<style>` blocks: scoped CSS and class-map generation
* `&` lazy destructuring: reactivity-preserving destructuring for Solid and Ripple

---

## Core Concept: The TSRX Template (`@`)

The defining feature of TSRX is the **TSRX template**, denoted by the `@` prefix.

A TSRX template allows JavaScript statements to be written directly inside JSX-producing layout boundaries. Its final statement must produce JSX.

### Syntax

```tsx
@{
  const value = computeValue();
  <Component value={value} />
}
```

### Output Shape

A TSRX template evaluates to the JSX produced by its final statement.

The final statement may be:

* a JSX element
* a JSX fragment
* a TSRX construct that itself produces JSX, such as `@if`, `@for`, `@switch`, or `@try`

### Core Rules

* A TSRX template may contain local JavaScript statements before its final JSX-producing statement.
* The final statement must produce JSX.
* Naked text strings or lone JavaScript expressions in the final position must be wrapped in a JSX fragment, such as `<>Text</>`.
* A TSRX template does not need to be wrapped in curly braces when nested inside JSX layout.

### Valid Positions

TSRX templates can be used in three main positions:

1. As a JavaScript expression
2. Inside an existing JSX template
3. As a function body declaration

---

# 1. Control Flow

TSRX replaces common JSX rendering patterns such as nested ternaries, short-circuit rendering, and `.map()` loops with structured control flow syntax.

---

## Conditionals: `@if`, `@else if`, and `@else`

The `@if` block provides structured conditional rendering inside JSX layout. Unlike JSX ternaries, each branch may contain local JavaScript statements before producing JSX.

### Syntax

```tsx
@if (condition) {
  <Result />
} @else if (otherCondition) {
  <OtherResult />
} @else {
  <Fallback />
}
```

### Output Shape

An `@if` block evaluates to the JSX output of exactly one matching branch.

### Constraints

* Each branch must produce JSX.
* Branches may contain local JavaScript statements before their final JSX-producing statement.
* `@else if` and `@else` branches are optional.

### Traditional JSX

```tsx
function UserProfile({ user, status }: Props) {
  return (
    <div>
      {status === 'loading' ? (
        <Spinner />
      ) : status === 'error' ? (
        <ErrorNotice />
      ) : user ? (
        <Card user={user} />
      ) : (
        <NoData />
      )}
    </div>
  );
}
```

### TSRX Equivalent

```tsx
function UserProfile({ user, status }: Props) {
  return (
    <div>
      @if (status === 'loading') {
        <Spinner />
      } @else if (status === 'error') {
        // Run localized logic directly inside the branch
        console.error("Profile load failed");
        const fallbackId = generateFallbackId();
        <ErrorNotice id={fallbackId} />
      } @else if (user) {
        <Card user={user} />
      } @else {
        <NoData />
      }
    </div>
  );
}
```

The error branch can perform local setup before rendering its layout without requiring a nested IIFE or extracted component.

---

## Loops: `@for...of` and `@empty`

The `@for...of` syntax iterates over arrays or iterables and produces one JSX child per iteration.

### Syntax

```tsx
@for (const item of items) {
  <ItemView item={item} />
}
```

With `index`, `key`, and `@empty`:

```tsx
@for (const item of items; index i; key item.id) {
  <ItemView item={item} />
} @empty {
  <EmptyState />
}
```

### Output Shape

A `@for` block evaluates to a JSX fragment containing one output per iteration.

If the iterable contains no elements and an `@empty` block is provided, the `@empty` block is rendered instead.

### Loop Configuration

The loop variable declaration supports an optional configuration suffix separated by a semicolon (`;`).

You may declare:

* an `index` variable
* a stable `key` expression
* both `index` and `key`

If both are declared, `index` must appear before `key`.

```tsx
@for (const item of items; index i; key item.id) {
  <li>{i + 1}. {item.name}</li>
}
```

The declared `index` variable may be used inside the `key` expression if the data objects do not have unique identifiers.

```tsx
@for (const item of items; index i; key i) {
  <li>{item.name}</li>
}
```

When a `key` is specified, the compiler automatically propagates it to the underlying rendered elements, even if those elements are wrapped in standard shorthand fragments (`<>...</>`).

### Constraints

* `@empty` is optional.
* `break` and `continue` are not supported inside `@for` blocks.
* If both `index` and `key` are declared, `index` must come first.
* Each iteration must produce JSX.

### Traditional JSX

```tsx
interface Item {
  name: string;
  vip: boolean;
}

function ItemList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.length === 0 ? (
        <li>No items available</li>
      ) : (
        items.map((item, i) => (
          <li key={i}>
            {i + 1}. {item.vip ? <strong>{item.name}</strong> : item.name}
          </li>
        ))
      )}
    </ul>
  );
}
```

### TSRX Equivalent

```tsx
function ItemList({ items }: { items: Item[] }) {
  return (
    <ul>
      @for (const item of items; index i; key i) {
        <li>
          <> {i + 1}. </>
          @if (item.vip) {
            <strong>{item.name}</strong>
          } @else {
            <>{item.name}</>
          }
        </li>
      } @empty {
        <li>No items available</li>
      }
    </ul>
  );
}
```

The loop, empty state, index, key, and conditional item rendering all remain inside the layout structure.

---

## Switches: `@switch`, `@case:`, and `@default:`

The `@switch` block provides structured multi-branch selection inside JSX layout.

Unlike JavaScript `switch`, TSRX switch branches do not fall through automatically. The `break` keyword is not used.

### Syntax

```tsx
@switch (value) {
  @case 'a': {
    <A />
  }
  @case 'b': {
    <B />
  }
  @default: {
    <Fallback />
  }
}
```

Multiple `@case` labels may be stacked to share a single output block.

```tsx
@switch (status) {
  @case 'warning':
  @case 'error': {
    <AlertIcon />
  }
  @default: {
    <InfoIcon />
  }
}
```

### Output Shape

A `@switch` block evaluates to the JSX output of the matching case.

If no case matches, the `@default` block is used when present.

### Constraints

* `@case` requires a trailing colon.
* `@default` requires a trailing colon.
* Branches do not fall through.
* `break` is not used.
* Multiple `@case` labels may share one output block.

### Traditional JSX

```tsx
function Notification({ type }: { type: 'success' | 'warning' | 'error' | 'info' }) {
  return (
    <div className="alert">
      {(() => {
        switch (type) {
          case 'success':
            return <SuccessIcon />;
          case 'warning':
          case 'error':
            return <AlertIcon />;
          default:
            return <InfoIcon />;
        }
      })()}
    </div>
  );
}
```

### TSRX Equivalent

```tsx
function Notification({ type }: { type: 'success' | 'warning' | 'error' | 'info' }) {
  return (
    <div className="alert">
      @switch (type) {
        @case 'success': {
          <SuccessIcon />
        }
        @case 'warning':
        @case 'error': {
          <AlertIcon />
        }
        @default: {
          <InfoIcon />
        }
      }
    </div>
  );
}
```

The branch selection stays directly inside the rendered layout without requiring an immediately invoked function expression.

---

## Error and Async Boundaries: `@try`, `@pending`, and `@catch`

The `@try` block combines asynchronous loading states and runtime error handling into a single syntactic construct.

Under the hood, `@pending` handles loading states, mapping to abstractions such as React’s `<Suspense>`, while `@catch` implements an Error Boundary component.

### Syntax

```tsx
@try {
  <Content />
} @pending {
  <Loading />
} @catch (error, reset) {
  <ErrorState error={error} onRetry={reset} />
}
```

### Output Shape

A `@try` block evaluates to one of three outputs:

* the main `@try` content
* the `@pending` fallback while asynchronous work is pending
* the `@catch` fallback if an error is thrown

### Catch Arguments

The `@catch` block receives the thrown error as its first argument.

It may also receive a `reset` function that triggers a retry of the `@try` block when invoked.

```tsx
@catch (error, reset) {
  <ErrorWidget message={error.message} onRetry={reset} />
}
```

### Constraints

* `@pending` handles pending asynchronous rendering states.
* `@catch` handles runtime errors from the protected render tree.
* The main `@try`, `@pending`, and `@catch` blocks must each produce JSX.
* The exact emitted boundary implementation depends on the selected compiler target.

### Traditional JSX

```tsx
import { Suspense } from 'react';
import { CustomErrorBoundary } from './ErrorBoundary';

function Dashboard() {
  return (
    <CustomErrorBoundary fallback={(error, reset) => <ErrorWidget message={error.message} onRetry={reset} />}>
      <Suspense fallback={<Skeleton />}>
        <DataGrid />
      </Suspense>
    </CustomErrorBoundary>
  );
}
```

### TSRX Equivalent

```tsx
function Dashboard() {
  return (
    @try {
      <DataGrid />
    } @pending {
      <Skeleton />
    } @catch (error, reset) {
      <ErrorWidget message={error.message} onRetry={reset} />
    }
  );
}
```

The loading and error states are declared beside the render tree they protect.

---

# 2. Statement Containers: `@{ ... }`

TSRX statement containers are inline JavaScript scopes that produce JSX.

They replace common JSX patterns such as defining and immediately invoking an IIFE to compute intermediate values inside layout.

### Syntax

```tsx
@{
  const value = computeValue();
  <span>{value}</span>
}
```

### Output Shape

A statement container evaluates to the JSX produced by its final statement.

### Valid Positions

Statement containers are valid inside JSX layout.

### Constraints

* A statement container creates a local JavaScript scope.
* It may contain local declarations and other JavaScript statements.
* Its final statement must produce JSX.
* Naked text strings or lone expressions in the final position must be wrapped in a fragment.

### Traditional JSX

```tsx
function PriceDisplay({ rawPrice, taxRate }: { rawPrice: number; taxRate: number }) {
  return (
    <div className="price-tag">
      {(() => {
        const total = rawPrice * (1 + taxRate);
        const formatted = total.toFixed(2);
        return <span>${formatted}</span>;
      })()}
    </div>
  );
}
```

### TSRX Equivalent

```tsx
function PriceDisplay({ rawPrice, taxRate }: { rawPrice: number; taxRate: number }) {
  return (
    <div className="price-tag">
      @{
        const total = rawPrice * (1 + taxRate);
        const formatted = total.toFixed(2);
        <span>${formatted}</span>
      }
    </div>
  );
}
```

The local calculation remains close to the layout that consumes it, without introducing an IIFE.

---

# 3. Function Bodies

TSRX extends its template block syntax directly to component declarations.

Function declarations and arrow functions may use the `@{ ... }` body form, allowing the final JSX-producing statement to serve as the component output without an explicit base-level `return`.

Early exit `return` statements remain valid when needed.

### Syntax

```tsx
function Component(props) @{
  const value = computeValue(props);
  <View value={value} />
}
```

```tsx
const Component = (props) => @{
  const value = computeValue(props);
  <View value={value} />
}
```

### Output Shape

A TSRX function body evaluates to the JSX produced by its final statement, unless an earlier `return` statement exits first.

### Valid Positions

Function body templates are valid as:

* function declaration bodies
* arrow function bodies

### Constraints

* The final statement must produce JSX.
* Early `return` statements may still be used.
* Local JavaScript statements may appear before the final JSX-producing statement.

### Traditional JSX

```tsx
// Traditional Function Declaration
function UserDeclaration({ user }: { user: User | null }) {
  if (!user) return <p>No profile data available.</p>;
  return <div className="card">{user.name}</div>;
}

// Traditional Arrow Function
const UserArrow = ({ user }: { user: User | null }) => {
  if (!user) return <p>No profile data available.</p>;
  return <div className="card">{user.name}</div>;
};
```

### TSRX Equivalent

```tsx
// TSRX Function Declaration
function UserDeclaration({ user }: { user: User | null }) @{
  if (!user) return <p>No profile data available.</p>;
  <div className="card">{user.name}</div>
}

// TSRX Arrow Function
const UserArrow = ({ user }: { user: User | null }) => @{
  if (!user) return <p>No profile data available.</p>;
  <div className="card">{user.name}</div>
}
```

The function body remains statement-oriented while eliminating the final explicit `return`.

---

# 4. Conditional Hooks

In standard JSX architecture, hooks are bound by strict framework rules. In React, for example, hooks must be invoked unconditionally at the root level of a component.

TSRX relaxes the authoring limitation while preserving hook safety through static compilation.

When a hook appears inside a conditional branch, loop branch, switch branch, or after an early return, TSRX extracts the hook-containing render path into an internal component so the generated target code preserves the hook rules of the target framework.

### Supported Positions

Hooks may be authored inside:

* conditional branches
* loop bodies
* switch branches
* render paths after early returns
* TSRX templates that the compiler can safely isolate

### Output Shape

Conditional hook usage compiles into generated component boundaries that preserve hook ordering and scoping rules for the target framework.

### Constraints

* Conditional hook support depends on static compiler analysis.
* Hook-containing branches must be extractable by the compiler.
* The compiler preserves access to downstream JSX dependencies by handling captured scope variables during extraction.
* The emitted implementation depends on the selected compiler target.

### Traditional JSX

```tsx
// Hand-crafted separation required to satisfy hook constraints
function StatusWrapper({ streamId }: { streamId: string | null }) {
  if (!streamId) {
    return <p>Disconnected</p>;
  }
  return <ActiveStream streamId={streamId} />;
}

function ActiveStream({ streamId }: { streamId: string }) {
  const data = useSubscription(streamId);
  return <div>Live: {data}</div>;
}
```

### TSRX Equivalent

```tsx
function StatusWrapper({ streamId }: { streamId: string | null }) @{
  if (!streamId) {
    return <p>Disconnected</p>;
  }

  // Hook run conditionally after an early return
  const data = useSubscription(streamId);
  <div>Live: {data}</div>
}
```

The hook can be authored in the same local render path where its data is consumed, while the compiler emits hook-safe target code.

---

# 5. Native Scoped CSS: `<style>`

TSRX natively parses standard CSS syntax inside `<style>` tags without requiring styles to be wrapped in JavaScript strings.

At compile time, CSS is extracted into an external asset layout that modern bundlers can process.

This is the one explicit exception to TSRX’s strict superset rule: standard JSX syntax such as the following is invalid in a `.tsrx` file:

```tsx
<style>{`color: red`}</style>
```

TSRX supports three distinct compilation modes for `<style>` blocks.

---

## A. `<style>` as a JSX Child

When declared as a direct child within component layout, a `<style>` block may contain standard CSS syntax, including semantic selectors such as `h1`, `p`, and `div`.

The compiler scopes the CSS to the component or local scope where the style block appears.

### Position

Direct child inside JSX layout.

### Selectors Allowed

* Semantic tag selectors
* Class selectors
* Other valid scoped CSS selectors

### Output Shape

The compiler:

* extracts the CSS into an external asset
* applies generated scoping class names to matching JSX elements
* replaces the rendered `<style>` element with `null`

### Example

```tsx
function Box() @{
  <div>
    <h1>Scoped Title</h1>
    <style>
      h1 { color: coral; }
      div { padding: 20px; }
    </style>
  </div>
}
```

The `h1` and `div` selectors are scoped to the component layout where the style block appears.

---

## B. `<style>` as a JSX Expression

When a `<style>` block is assigned directly to a JavaScript variable as an expression, it compiles to a type-safe `Record<string, string>` containing mappings to generated, scoped class names.

In this mode, only class selectors are permitted.

### Position

Assigned directly to a JavaScript variable.

### Selectors Allowed

* Class selectors only

### Output Shape

The compiler produces a class-name mapping object.

```tsx
const classes = <style>
  .cardContainer { border: 1px solid #ccc; }
  .titleText { font-weight: bold; }
</style>;
```

The resulting `classes` value behaves like:

```ts
Record<string, string>
```

### Constraints

* Semantic tag selectors are not permitted in expression style blocks.
* Class references are type-safe.
* Accessing an undefined class name is invalid.

### Example

```tsx
function CustomCard() @{
  const classes = <style>
    .cardContainer { border: 1px solid #ccc; }
    .titleText { font-weight: bold; }
  </style>;

  <div className={classes.cardContainer}>
    <span className={classes.titleText}>Card Header</span>
  </div>
}
```

The style block defines a scoped class map that can be consumed directly from TypeScript.

---

## C. `<style>` at Module Scope

When written directly at the module root scope without variable assignment, a `<style>` block applies globally to JSX elements throughout that module.

### Position

Module root scope.

### Selectors Allowed

* Global CSS selectors

### Output Shape

The compiler emits CSS that applies across the module.

### Example

```tsx
// Applies to all components within this module file
<style>
  p { line-height: 1.5; }
</style>

export function ComponentA() @{ <p>Paragraph A</p> }
export function ComponentB() @{ <p>Paragraph B</p> }
```

The `p` selector applies to paragraph elements defined across the module.

---

# 6. Fine-Grained Reactive Framework Feature: Lazy Destructuring (`&`)

For fine-grained reactive framework targets such as **Solid** and **Ripple**, destructuring properties immediately can break the tracking graph by reading primitive accessor values too early.

TSRX introduces **lazy destructuring**, denoted by the `&` modifier prefix.

Lazy destructuring uses standard destructuring syntax with an added `&` marker:

```tsx
const &{ property } = object;
```

The compiler defers property access tracking under the hood so destructuring can preserve fine-grained reactivity.

### Syntax

```tsx
const &{ user, theme } = props;
```

Lazy destructuring is also valid in component parameter position:

```tsx
function ReactiveComponent(&{ user, theme }) @{
  <div>{user.name}</div>
}
```

### Valid Positions

Lazy destructuring is permitted anywhere standard object or array destructuring is allowed, including:

* component parameters
* local variable declarations
* nested block scopes
* object destructuring
* array destructuring

### Output Shape

Lazy destructuring compiles to target-specific reactive property access that preserves tracking behavior.

### Constraints

* Lazy destructuring is relevant to fine-grained reactive targets such as Solid and Ripple.
* It is not required for targets whose reactivity model is not affected by eager destructuring.
* The emitted implementation depends on the selected compiler target.

### Example

```tsx
// Destructuring props lazily in Solid or Ripple without losing reactivity
function ReactiveComponent(&{ user, theme }) @{
  <div>
    <h1 className={theme}>{user.name}</h1>
  </div>
}
```

Lazy destructuring may also be used inside a standard block scope.

```tsx
function TaskManager(props) @{
  const state = useComplexState();

  // Localized lazy destructuring in variable assignment positions
  const &{ currentTask, priority } = state.todo;

  <div>Task: {currentTask} ({priority})</div>
}
```

The destructured values remain local and ergonomic while preserving the reactive tracking semantics of the target framework.

---

# 7. Feature Reference

## TSRX Template

```tsx
@{
  const value = computeValue();
  <Component value={value} />
}
```

Used for JSX-producing blocks that contain local JavaScript statements.

## Conditional Rendering

```tsx
@if (condition) {
  <A />
} @else {
  <B />
}
```

Used for structured conditional layout.

## Iterable Rendering

```tsx
@for (const item of items; index i; key item.id) {
  <Item item={item} />
} @empty {
  <EmptyState />
}
```

Used for rendering arrays or iterables with optional empty-state handling.

## Switch Rendering

```tsx
@switch (value) {
  @case 'a': {
    <A />
  }
  @default: {
    <Fallback />
  }
}
```

Used for non-fallthrough multi-branch rendering.

## Async and Error Boundary Rendering

```tsx
@try {
  <Content />
} @pending {
  <Loading />
} @catch (error, reset) {
  <ErrorState error={error} onRetry={reset} />
}
```

Used for colocated async loading and error fallback layout.

## Statement Container

```tsx
@{
  const formatted = formatValue(value);
  <span>{formatted}</span>
}
```

Used for inline local computation inside JSX layout.

## Function Body Template

```tsx
function Component(props) @{
  const value = computeValue(props);
  <View value={value} />
}
```

Used for JSX-producing component bodies without a final explicit `return`.

## Expression Style Block

```tsx
const classes = <style>
  .container { padding: 20px; }
</style>;
```

Used for scoped, type-safe CSS class maps.

## Lazy Destructuring

```tsx
const &{ currentTask, priority } = state.todo;
```

Used to preserve fine-grained reactive tracking during destructuring.

