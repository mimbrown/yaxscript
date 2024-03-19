# YaxScript

YaxScript (Yet Another X-script) is my exploration into a possible future of web frameworks.
When React introduced JSX to the world, they showed the potential power of extending the JS language.
Its beauty was in its flexibility.
The usefulness of the abstraction the React team created is proved by the number of non-React frameworks that use it.
By making HTML-in-JS a semantic part of the language, it gave a compiler target which has allowed for a huge amount of experimentation and optimization.
Since then, however, we have avoided extending the JS language any further.
There's a high barrier to entry; once you cross that line of native JS semantics, you've now committed to writing your own language parser, syntax highlighting, language server, etc.
But the potential gains in developer experience, compiler optimizations, and interoperability are enormous.
I believe this is the next frontier for web frameworks.
YaxScript is a description of the language I would want to use to write code for the web.
It does not exist, and let's be honest, probably never will.
But here's dreaming.

Solid js is the most standard signal-driven framework I know of, and I've iterated on YaxScript with an eye towards how it would compile to Solid-specific primitives.
Therefore I will use Solid throughout this explainer to show one possible compiled output of the YaxScript syntax.

## Types

There is no "JS" vs "TS" flavor of YaxScript.
Typescript types are always allowed (albeit with the same restrictions imposed in TSX).
It's up to the developer if/how they want type errors to be surfaced.
If you don't want to use Typescript, then don't write it.
If you're consuming a library written in Yaxscript that was written with type information, it will only help you.

## Reactivity

Most frameworks have coalesced around Signals and Effects as the main units of reactivity.
Signals are basically a wrapper around a value with the ability to track reads and writes.
Effects track which signals are read in a given run, and rerun when any of those dependent signals are changed.
Each framework that uses signals implements them differently.
YaxScript seeks to strike the balance between ergonomics and explicitness.

Goals:

* Ergonomic: it should be a delight to work with.
* Explicit: devs should never be surprised about what they are dealing with.
  * To this end, signals should be statically analyzable.
  This is required for the compiler, but is also helpful for the developer.
  It should be possible for the developer to know, even at the level of *syntax highlighting*, whether they are reading from/writing to a regular variable or a signal.
  * Ideally, there should be *no surprise proxies* (I'm looking at you, Vue and Svelte).
  Deep reactivity should be achievable without resorting to proxies.

YaxScript introduces the `state` keyword to declare a signal:

```ts
state myState = 'foo';

// in Solid js, compiles to:
const [myState, setMyState] = createSignal('foo');
```

When a variable is declared using `state`, it's what we call a "live" signal.
That means any reference to the variable is a (possibly tracked, if we're in a tracking context) reference to the signal's current value.
Similarly, any updates to the variable are writes are to the value of the signal.

```ts
postData({
  myState,
});

// in Solid js, compiles to:
postData({
  myState: myState(),
})
```

```ts
myState = 'bar';

// in Solid js, compiles to:
setMyState('bar');
```

So far, this looks a lot like Svelte 5 syntax.
But sometimes we actually want to pass around the signal wrapper itself, not its value.
It's quite normal to create the state and write its business logic in one place, and use that state in other places.
Additionally, we often want to define a strict API that guards how the signal can be written to, while allowing arbitrary reads.
To support this, YaxScript introduces two new unary operators that operate *only* on live signals: `readonly` and `readwrite`.

* `readonly` operates on a live signal and resolves to an "inert" readonly signal.
* `readwrite` operates on a live signal and resolves to an "inert" signal that can be written to as well as read.

Inert signals can be passed around like normal variables quite freely until they need to become live again.
They are typed as opaque symbols with information about their value's type; you can't do anything with them except pass them around until you want them to be live again.

For example, let's say you wanted to create a `counter` that can only be incremented.
In YaxScript, you can write this:

```ts
function createCounter() {
  state count = 0;
  return {
    count: readonly count,
    increment() {
      count += 1;
    },
  };
}

// equivalent using object shorthand:
function createCounter() {
  state count = 0;
  return {
    readonly count,
    increment() {
      count += 1;
    },
  };
}

// in Solid js, compiles to:
function createCounter() {
  const [count, setCount] = createSignal(0);
  return {
    count,
    increment() {
      setCount(count() + 1);
    },
  };
}
```

How does an inert signal become live again?
YaxScript introduces the equivalent `readonly` and `readwrite` binding patterns.
To consume `createCounter`, you would write this in YaxScript:

```ts
const {
  count: readonly count,
  increment,
} = createCounter();

// equivalent using object binding shorthand:
const {
  readonly count,
  increment,
} = createCounter();

// in Solid js, compiles to:
const {
  count,
  increment,
} = createCounter();
```

In the above example, `count` is a live signal.
Note that the binding pattern *must* match the operator used to make the signal inert.
It's an error to bind a `readonly` signal to a `readwrite` variable and vice versa.
This is as much for the developer as for the compiler; there should never be any doubt as to whether the variable you are working with is mutable or not.

So now the following statement should make sense: the `state` keyword is just syntactic sugar for creating a signal and binding to it using `readwrite`.
In other words, the following two statements are equivalent:

```ts
state myState = 'foo';

// equivalent to:
const readwrite myState = createSignal('foo');

// in Solid js, compiles to:
const [myState, setMyState] = createSignal('foo');
```

It is an error[^1] to assign an inert signal (readonly or readwrite) as the value of another signal.
The following are both disallowed:

```ts
state a = 'foo';
state b = readwrite a; // or `readonly a`
```

```ts
state a;
a = readwrite a; // or `readonly a`
```

## Components

### Component Props

Components in most frameworks are often functions (or close to it), but they normally have some special rules.
In YaxScript, components are somewhat similar to functions, but they're not quite the same.
Components are declared using the `component` keyword, similar to the `function` keyword.
They take a list of props, which are similar to function arguments except that they are *unordered* and *named*.
Let's say you have a component which takes two props: `name` and `class`.
You would declare it like this:

```ts
component MyComponent(name, class as classList) {
  // We'll get to the body in a minute.
  // But here we have access to the `name` and `classList`
  // variables, which are functionally equivalent to live
  // readonly signals.
}
```

And use it like this:

```jsx
<MyComponent name="Some Name" class="foo bar" />
```

Spread/rest props are allowed:

```ts
component MyComponent(prop1, ...rest) {}
```

Some frameworks have the concept of two-way binding (e.g. Vue's `v-model` and Svelte's `bind:` syntax).
This is useful when the parent and child need to share state, and both need to be able to write to it.
This is perhaps most common in form field-like components.
YaxScript enables and encourages ergonomic signal-sharing.
This forces explicitness, yet is consistent with the rest of the language, is non-verbose, and eliminates the overhead of keeping two signals in sync with one another because there is only one signal!
If a component wants to expose one or more props as "two way bindable", it simply expects an inert readwrite signal and binds to it:

```tsx
// provide a default signal to make this prop optional
component MyTextField(readwrite value = createSignal('')) {
  // `value` is a live readwrite signal
}

// usage:
state value = 'initial value';
<MyTextField value={readwrite value} />

// if the parent wants to provide a default value, but doesn't ever need to write to it:
<MyTextField value={createSignal('initial value')} />
```

### Component Body

The body of the component is not quite the same as a function body either.
It's an enhanced expression.
It's similar to the body of the proposed `do-expression` syntax, but with a couple extra features.
Basically:
* 0 or more statements are allowed.
* The last statement must be a classic JS(X) expression, an `if` block, or a `for-of` loop[^2].
  * If the last statement is an `if` block or loop, the block's body is enhanced in the same way.
* This last statement is what the whole component body resolves to (or `undefined` if there are no statements).
* The `return` keyword is *disallowed*.
You can't early return out of a component.

JSX-like templating is allowed and works how you would expect, except for one major difference: the expressions enclosed within `{}` in the JSX are also enhanced.

It's easiest to see examples.
Here's an example of a Login/Logout Component:

```tsx
component SessionToggle(user?: User) {
  console.log('This will log once per instance of this component');

  // open question: is this right here a tracking context?
  // in other words: if we reference the value of a signal here,
  // and that signal changes, would the whole component rerender?

  if (user) {
    console.log('I log once every time the `user` becomes truthy');

    // same open question: is this right here a tracking context?

    <>
      <span>
        Hello, {
          console.log('I log once every time `user.name` changes');
          user.name
        }!
      </span>
      <button on:click={logout}>
        Sign out
      </button>
    </>
  } else {
    console.log('I log once every time the `user` becomes falsy');

    <button>
      Sign in
    </button>
  }
}
```

Rendering a list:

```tsx
const always = () => true;

component ItemsList<T>(
  items: T[],
  Item: Component<{ item: T }>,
  filter: (item: T) => boolean = always,
) {
  <ul>
    {
      for (state item of items) {
        if (filter(item)) {
          <Item {item} />
        }
      }
    }
  </ul>
}

// usage:
<ItemsList
  items={[1, 2, 3, 4]}
  Item={component (item) {
    // this is an anonymous component (like an anonymous function)
    <li>
      Item #{item}
    </li>
  }}
  filter={number => number % 2 === 0}
/>
```

## Styling

Unlike reactivity, there's not much consensus on the best practices for styling.
Some people love frameworks like Tailwind, others hate it.
Most frameworks support some kind of scoped styling, but there's doubts about the ability of scoped styles to scale in large projects.
YaxScript does not aim (or pretend to be able) to solve this debate, but it's too important to just punt.
The way forward, I believe, is to provide a YaxScript-native way to write styles for your components, just like JSX provided a native way to author HTML, while leaving the specific output and application of those styles to the compiler.

Principles:
* It would be a mistake to adopt a bespoke styling language like Tailwind.
The authoring experience should be based on CSS.
StyleX should provide some good ideas.
* It should follow the same goals of ergonomic and explicit.
Enabling locality of thinking is a must.
* There is no reason we can't give CSS the JSX treatment and allow mixing of JS within CSS.

Pure spitballing with very little thought:

```
styleblock myFirstBlock(hoverColor: <color> = #000, hoverBackground: <color> = #c0ffee) {
  padding: .5em;
  border-radius: .25em;
  &:hover {
    background-color: var(hoverBackground);
    color: var(hoverColor);
  }
}

styleblock mySecondBlock {
  border-radius: .5em;
}

component MyComp() {
  <div (myFirstBlock(hoverBackground: #fff), mySecondBlock, styleblock {
    font: 'Comic Sans'
  })>

  </div>
}
```

[^1]: Relevant question: is this a compiler or runtime error? The answer would probably be, compile time error to the extant that it is analyzable, otherwise a runtime error in dev mode, otherwise silently ignored in production mode.

[^2]: Who knows, maybe other kinds of loops are supportable as well.
