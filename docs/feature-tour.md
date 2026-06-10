# Introducing TSRX

TSRX is a compiler-agnostic TypeScript language extension for JSX-shaped UI templates with official support for **React, Preact, Solid, Vue, and Ripple** compiler targets. You can think of it as “JSX 2.0”: the same TypeScript-and-markup mental model, extended with first-class control flow, local statements, and native styling.

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
* JavaScript comments in JSX: `//` and `/* ... */` without expression containers
* JSX attribute shorthand: `{foo}` as `foo={foo}` in attribute position
* Dynamic JSX tag names: `<{expr}>...</{expr}>` for runtime-selected elements or components
* Native `<style>` blocks: scoped CSS and class-map generation
* `&` lazy destructuring: reactivity-preserving destructuring for Solid, Vue, and Ripple

---

## Compatibility with JSX and TypeScript

TSRX is designed as a strict superset of JSX and TypeScript syntax, subject to the `<style>` exception below. Existing imports, types, JavaScript statements and expressions, JSX elements, JSX fragments, JSX text, and JSX expression containers keep their standard meanings in `.tsrx` files unless TSRX-specific syntax is used.

Outside that exception, TSRX features are opt-in. Syntax such as `@if`, `@for`, `@{ ... }`, `<style>...</style>`, and `&{ ... }` adds capabilities without changing ordinary JavaScript or TypeScript semantics.

There are two important boundaries:

* Native `<style>` blocks are parsed as CSS, not JSX children. Write CSS directly inside the block:

  ```tsx
  <style>
    h1 { color: red; }
  </style>
  ```

  Standard JSX style-string children are outside TSRX’s native `<style>` form:

  ```tsx
  <style>{`h1 { color: red; }`}</style>
  ```

* The compatibility claim applies to JSX and TypeScript syntax, not necessarily to literal JSX text that contains TSRX syntax markers. For example, text containing `@if` may be parsed as TSRX control-flow syntax instead of being rendered as text. Use an explicit JSX string expression, such as `{'@if'}`, when TSRX syntax should appear as literal text.

---

## Core Model: TSRX Templates (`@`)

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
* `return` statements are allowed only in the top-level `@{ ... }` block that serves as a function body; all other TSRX templates and control-flow blocks produce their result from the final JSX-producing statement.
* The final statement must produce JSX.
* Naked text strings or lone JavaScript expressions in the final position must be wrapped in a JSX fragment, such as `<>Text</>`.
* A TSRX template does not need to be wrapped in curly braces when nested inside JSX layout.

### Valid Positions

TSRX templates can be used in three main positions:

1. As a JavaScript expression
2. Inside an existing JSX template
3. As a function body declaration

---

## JSX Syntax Conveniences

TSRX also includes a small set of JSX authoring conveniences that are independent of TSRX template control flow.

### JavaScript Comments in JSX

TSRX supports normal JavaScript comment syntax inside JSX templates. Both `//` and `/* ... */` comments can appear directly in JSX layout without being wrapped in `{...}`.

```tsx
<div>
  // Render the primary action first
  <PrimaryAction />

  /* Secondary actions stay grouped below */
  <SecondaryActions />
</div>
```

### JSX Attribute Shorthand

TSRX supports shorthand for passing a value as a JSX attribute with the same name. In JSX attribute position, `{foo}` is equivalent to `foo={foo}`.

```tsx
<Card {user} {isSelected} />
```

This is equivalent to:

```tsx
<Card user={user} isSelected={isSelected} />
```

The shorthand is recognized only in JSX attribute position. Between JSX tags, `{foo}` remains a normal JSX child expression.

### Dynamic JSX Tag Names

TSRX supports expression-based JSX tag names for elements or components selected at runtime. Put the tag expression in braces immediately after the opening `<`, and repeat the same expression in the closing tag.

```tsx
function TextBlock(props: { as: 'p' | 'blockquote'; children: JSX.Children }) @{
  <{props.as} class="text-block">
    {props.children}
  </{props.as}>
}
```

The tag expression may evaluate to an intrinsic element name, such as `'section'`, or to a component value.

```tsx
<{props.panel} title="Details">
  <p>Dynamic component content</p>
</{props.panel}>
```

Constraints:

* The opening and closing dynamic tag expressions must match syntactically.
* Self-closing dynamic tags use the same opening form: `<{props.icon} />`.
* Dynamic tag names are JSX tag names, not JSX child expressions; use ordinary `{expr}` between tags when rendering a value as a child.

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
function UserProfile(props: Props) {
  return (
    <div>
      {props.status === 'loading' ? (
        <Spinner />
      ) : props.status === 'error' ? (
        <ErrorNotice />
      ) : props.user ? (
        <Card user={props.user} />
      ) : (
        <NoData />
      )}
    </div>
  );
}
```

### TSRX Equivalent

```tsx
function UserProfile(props: Props) @{
  <div>
    @if (props.status === 'loading') {
      <Spinner />
    } @else if (props.status === 'error') {
      // Run localized logic directly inside the branch
      console.error("Profile load failed");
      const fallbackId = generateFallbackId();
      <ErrorNotice id={fallbackId} />
    } @else if (props.user) {
      <Card user={props.user} />
    } @else {
      <NoData />
    }
  </div>
}
```

In the error branch, setup code can live right next to the fallback UI that depends on it. There is no need to wrap the branch in a nested IIFE or move the fallback into a separate component just to declare a local value.

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

function ItemList(props: { items: Item[] }) {
  return (
    <ul>
      {props.items.length === 0 ? (
        <li>No items available</li>
      ) : (
        props.items.map((item, i) => (
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
function ItemList(props: { items: Item[] }) @{
  <ul>
    @for (const item of props.items; index i; key i) {
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
function Notification(props: { type: 'success' | 'warning' | 'error' | 'info' }) {
  return (
    <div className="alert">
      {(() => {
        switch (props.type) {
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
function Notification(props: { type: 'success' | 'warning' | 'error' | 'info' }) @{
  <div className="alert">
    @switch (props.type) {
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

`@pending` and `@catch` may also be declared as empty blocks. An empty block renders `null` automatically, and `@catch` may omit its parameter list when the error and reset values are not needed.

```tsx
@try {
  <Content />
} @pending {
} @catch {
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
* The main `@try`, `@pending`, and `@catch` blocks must each produce JSX, except that empty `@pending {}` and `@catch {}` blocks render `null` automatically.
* `@catch` may omit its parameter list when the thrown error and reset function are not needed.
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
function Dashboard() @{
  @try {
    <DataGrid />
  } @pending {
    <Skeleton />
  } @catch (error, reset) {
    <ErrorWidget message={error.message} onRetry={reset} />
  }
}
```

The loading and error states are declared beside the render tree they protect.

---

# 2. Statement Containers: `@{ ... }`

TSRX statement containers are inline JavaScript scopes that produce JSX.

They replace common JSX patterns such as defining and immediately invoking an IIFE to compute intermediate values inside layout.

### Syntax

```tsx
<div>@{
  const value = computeValue();
  <span>{value}</span>
}</div>
```

### Output Shape

A statement container evaluates to the JSX produced by its final statement.

### Valid Positions

Statement containers are valid inside JSX layout.

### Constraints

* A statement container creates a local JavaScript scope.
* It may contain local declarations and JavaScript statements.
* `return` statements are not allowed; use the final JSX-producing statement as the container result.
* Its final statement must produce JSX.
* Naked text strings or lone expressions in the final position must be wrapped in a fragment.

### Traditional JSX

```tsx
function PriceDisplay(props: { rawPrice: number; taxRate: number }) {
  return (
    <div className="price-tag">
      {(() => {
        const total = props.rawPrice * (1 + props.taxRate);
        const formatted = total.toFixed(2);
        return <span>${formatted}</span>;
      })()}
    </div>
  );
}
```

### TSRX Equivalent

```tsx
function PriceDisplay(props: { rawPrice: number; taxRate: number }) @{
  <div className="price-tag">@{
    const total = props.rawPrice * (1 + props.taxRate);
    const formatted = total.toFixed(2);
    <span>${formatted}</span>
  }</div>
}
```

The local calculation remains close to the layout that consumes it, without introducing an IIFE.

---

# 3. Function Bodies

TSRX extends its template block syntax directly to component declarations.

Function declarations and arrow functions may use the `@{ ... }` body form, allowing the final JSX-producing statement to serve as the component output without an explicit base-level `return`.

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
* Early exit `return` statements may still be used in top-level TSRX function bodies.
* Function body templates are the only TSRX `@{ ... }` form where `return` statements are allowed.
* Local JavaScript statements may appear before the final JSX-producing statement.

### Traditional JSX

```tsx
function ProjectSummary(props: { projects: Project[] }) {
  return (
    <div className="projects">
      {props.projects.length === 0 ? (
        <p>No projects available.</p>
      ) : (
        props.projects.map((project) => {
          const openTasks = project.tasks.filter((task) => !task.done);

          return (
            <section key={project.id}>
              <h2>{project.name}</h2>

              {openTasks.length > 0 ? (
                <ul>
                  {openTasks.map((task, i) => {
                    const label = `${i + 1}. ${task.title}`;
                    return <li key={task.id}>{label}</li>;
                  })}
                </ul>
              ) : (
                <p>All tasks complete.</p>
              )}
            </section>
          );
        })
      )}
    </div>
  );
}
```

### TSRX Equivalent

```tsx
function ProjectSummary(props: { projects: Project[] }) @{
  <div className="projects">
    @for (const project of props.projects; key project.id) {
      const openTasks = project.tasks.filter((task) => !task.done);

      <section>
        <h2>{project.name}</h2>

        @if (openTasks.length > 0) {
          <ul>
            @for (const task of openTasks; index i; key task.id) {
              const label = `${i + 1}. ${task.title}`;
              <li>{label}</li>
            }
          </ul>
        } @else {
          <p>All tasks complete.</p>
        }
      </section>
    } @empty {
      <p>No projects available.</p>
    }
  </div>
}
```

Using `@{ ... }` for the component body keeps the whole component in a single template-oriented flow: setup can appear before the returned markup, and nested loops or conditionals can declare local values immediately before the JSX that consumes them. In this example, `openTasks` stays inside the project loop and `label` stays inside the task loop, without falling back to nested `.map()` callbacks, ternaries, or IIFEs.

### Reactive Guard Clauses in Fine-Grained Targets

In fine-grained reactive targets such as Solid and Ripple, ordinary JavaScript guard clauses inside components can be non-reactive because the component function may execute only once.

```tsx
function Component(props) {
  if (props.disabled) return null;
  return <div />;
}
```

A TSRX function body can preserve the guard-clause authoring style while compiling to target-native reactive control flow.

```tsx
function Component(props) @{
  if (props.disabled) return null;
  <div />
}
```

For a Solid target, the generated code may use a reactive control-flow primitive such as `<Show>`:

```tsx
function Component(props) {
  return (
    <Show when={props.disabled} fallback={<div />}>
      {null}
    </Show>
  );
}
```

Changes to `props.disabled` can therefore continue to update the rendered output even though the source uses an ordinary-looking guard clause.

---

# 4. Native Scoped CSS: `<style>`

TSRX natively parses standard CSS syntax inside `<style>` tags without requiring styles to be wrapped in JavaScript strings.

At compile time, CSS is extracted into external assets that modern bundlers can process. The compiler can scope selectors to a template or generate class maps for use from TypeScript.

TSRX supports two `<style>` forms: scoped blocks in JSX layout and variable-declared class maps.

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

## B. `<style>` as a Variable-Declared Class Map

When a `<style>` block is used as the initializer of a JavaScript variable declaration, it compiles to a class-map object whose values are generated, scoped class names. The available properties correspond to the class selectors declared in the block.

The variable declaration may appear inside a component or at module scope. Module-scope declarations use the same class-map behavior and can be referenced by multiple components in the file.

In this mode, class selectors define the properties available on the generated class map.

### Position

Initializer of a JavaScript variable declaration, either in local scope or module root scope.

### Selectors Allowed

* Class selectors that should become class-map properties

### Output Shape

The compiler extracts the CSS and initializes the variable with a class-name mapping object.

```tsx
const styles = <style>
  .cardContainer { border: 1px solid #ccc; }
  .titleText { font-weight: bold; }
</style>;
```

At runtime, the resulting `styles` value behaves like a plain object mapping declared class names to generated class-name strings.

### Constraints

* Use a JSX-child `<style>` block when semantic tag selectors such as `h1`, `p`, or `div` are needed.
* The class-map variable is type-safe: declared classes autocomplete, and unknown class properties are reported by TypeScript.

### Local Example

```tsx
function CustomCard() @{
  const styles = <style>
    .cardContainer { border: 1px solid #ccc; }
    .titleText { font-weight: bold; }
  </style>;

  <div className={styles.cardContainer}>
    <span className={styles.titleText}>Card Header</span>
  </div>
}
```

The style block defines a scoped class map that can be consumed directly from TypeScript.

### Module-Scope Example

```tsx
const styles = <style>
  .paragraph { line-height: 1.5; }
</style>;

export function ComponentA() @{ <p className={styles.paragraph}>Paragraph A</p> }
export function ComponentB() @{ <p className={styles.paragraph}>Paragraph B</p> }
```

The `.paragraph` class is available through the module-scope `styles` map. To style semantic elements without manually applying classes, place the `<style>` block inside JSX layout instead.

---

## Constraints and Escape Hatches

* TSRX’s native CSS parsing is the one explicit exception to its strict superset rule: a `<style>` block contains CSS source directly.
* The variable-declared class-map form is the module-scope style form.
* Use `:global(...)` when a selector should intentionally target global or external markup instead of receiving TSRX’s generated scoping.
* `<style>` contents are static CSS. For runtime-dependent values, set CSS custom properties on JSX elements and read them from CSS:

  ```tsx
  <div className="box" style={{ '--box-color': color }}>
    <style>
      .box { color: var(--box-color); }
    </style>
  </div>
  ```

---

# 5. Lazy Destructuring (`&`)

For fine-grained reactive framework targets such as **Solid**, **Vue**, and **Ripple**, destructuring properties immediately can break the tracking graph by reading primitive accessor values too early.

TSRX introduces **lazy destructuring**, denoted by the `&` modifier prefix.

Lazy destructuring uses standard object and array destructuring syntax with an added `&` marker:

```tsx
const &{ property } = object;
const &[first, second] = tuple;
```

### Syntax

```tsx
const &{ user, theme } = state;
```

Lazy destructuring can be used at the top of a component body when a target benefits from preserving reactive property access:

```tsx
function ReactiveComponent(props) @{
  const state = getReactiveState(props.id);
  const &{ user, theme } = state;
  <div className={theme}>{user.name}</div>
}
```

### Valid Positions

Lazy destructuring is permitted anywhere standard object or array destructuring is allowed, including:

* local variable declarations
* nested block scopes
* object destructuring
* array destructuring

### Output Shape

Lazy destructuring compiles to target-specific reactive property or index access that preserves tracking behavior. The compiler defers access tracking under the hood so object and array destructuring can remain ergonomic without eagerly reading reactive values.

Object patterns may destructure getter-backed properties. Object rest preserves property descriptors, so getter-backed properties that flow into a `...rest` object remain getter-backed instead of becoming eager value copies.

### Constraints

* Lazy destructuring is relevant to fine-grained reactive targets such as Solid, Vue, and Ripple.
* It is not required for targets whose reactivity model is not affected by eager destructuring.
* The emitted implementation depends on the selected compiler target.

### Object Destructuring Example

```tsx
// Destructuring reactive state lazily in Solid, Vue, or Ripple without losing reactivity
function ReactiveComponent(props) @{
  const state = getReactiveState(props.id);
  const &{ user, theme } = state;

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

### Getter and Rest Example

Getter-backed properties and object rest work with the same syntax. In this example, `displayName` may be a getter, and any getter properties collected into `details` keep their descriptors.

```tsx
function UserCard(source) @{
  const &{ displayName, ...details } = source;

  <article title={details.title}>{displayName}</article>
}
```

### Array Destructuring Example

Lazy destructuring also works with array patterns.

```tsx
function Counter(props) @{
  const &[count, setCount] = props.counter;

  <button onClick={() => setCount(count + 1)}>Count: {count}</button>
}
```

The destructured values remain local and ergonomic while preserving the reactive tracking semantics of the target framework.
