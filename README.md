# YaxScript

YaxScript is an exploration into the next generation of web frameworks.
When React introduced JSX to the world, they showed the potential power of extending the JS language.
Its beauty was in its flexibility.
The usefulness of the abstraction the React team created is proved by the number of non-React frameworks that use it.
By making HTML-in-JS a semantic part of the language, it gave a compiler target which has allowed for a huge amount of experimentation and optimization.
Since then, however, we have avoided extending the JS language any further.
There's a high barrier to entry; once you cross that line of native JS semantics, you've now committed to writing your own language parser, syntax highlighting, language server, etc.
But the potential gains in developer experience, compiler optimizations, and interoperability are enormous.
I believe this is the next frontier for web frameworks.
YaxScript is my own exploration into the language I would want to use to write code for the web.

Solid js is the most standard signal-driven framework I know of, and I defined YaxScript with an eye towards how it would compile to Solid-specific primitives.
Therefore I will use Solid throughout this explainer to show one possible compiled output of the YaxScript syntax.

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
  It should be possible for the developer to know, at the level of syntax highlighting, whether they are reading from/writing to a regular variable or a signal.
  * Ideally, there should be *no surprise proxies* (I'm looking at you, Vue and Svelte).
  Deep reactivity should be achievable without resorting to proxies.

YaxScript introduces the `state` keyword to declare a signal:

```ts
state myState = 'foo';

// compiles to:
const [myState, setMyState] = createState('foo');
```

When a variable is declared using `state`, it's what we call a "live" signal.
That means any reference to the variable is a (possibly tracked) reference to the signal's current value.
Similarly, any updates to the variable are writes are to the value of the signal.

```ts
postData({
  myState,
});

// compiles to:
postData({
  myState: myState(),
})
```

```ts
myState = 'bar';

// compiles to:
setMyState('bar');
```

So far, this looks a lot like Svelte 5 syntax.
But sometimes we actually want to pass around the signal wrapper itself, not its value.
It's quite normal to create the state and write its business logic in one place, and use that state in other places.
Additionally, we often want to define a strict API that guards how the signal can be written to, while allowing arbitrary reads.
To support this, YaxScript introduces two new unary operators that operate *only* on live signals: `readonly` and `readwrite`.
`readonly` operates on a live signal and resolves to an "inert" readonly signal.
`readwrite` operates on a live signal and resolves to an "inert" signal that can be written to as well as read.
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

// compiles to:
function createCounter() {
  const [count, setCount] = createState(0);
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

// compiles to:
const {
  count,
  increment,
} = createCounter();
```

In the above example, `count` is now a live signal.
Note that the binding pattern *must* match the operator used to make the signal inert.
It's an error to bind a `readonly` signal to a `readwrite` variable and vice versa.
This is as much for the developer as for the compiler; there should never be any doubt as to whether the variable you are working with is mutable or not.

So now the following statement should make sense: the `state` keyword is just syntactic sugar for creating a signal and binding to it using `readwrite`.
In other words, the following two statements are equivalent:

```ts
state myState = 'foo';

// equivalent to:
const readwrite myState = createState('foo');

// compiles to:
const [myState, setMyState] = createState('foo');
```

It is an error to assign an inert signal (readonly or readwrite) as the value of another signal.
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

Components in most frameworks are often functions (or close to it), but they normally have some special rules.
React popularized the idea that components are just a function of the passed props.
In YaxScript, components are somewhat similar to functions, but they're not quite the same.
Components are declared using the `component` keyword, similar to the `function` keyword.
They take a list of props, which are similar to function arguments except that they are *unordered* and *named*.
Let's say you have a component which takes two props: `name` and `class`.
You would declare it like this:

```ts
component MyComponent(name, class as classList) {
  // We'll get to this in a minute.
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

The body of the component is not quite the same as a function body either.
It's an enhanced expression.
It's similar to the body of the proposed `do-expression` syntax, but with a couple extra features.
Basically:
* 0 or more statements are allowed.
* The last statement must be a classic JS(X) expression, an `if` block, or a `for` or `while` loop.
  * If the last statement is an `if` block or loop, the block's body is enhanced in the same way.
* This last statement is what the whole component body resolves to (or `undefined` if there are no statements).
* The `return` keyword is *disallowed*.
You can't early return out of a component.

JSX-like templating is allowed and works how you would expect, except for one major difference: the expressions enclosed within `{}` in the JSX are also enhanced.

It's easiest to see examples.
Here's an example of a Login/Logout button:

```tsx
component SessionToggle(user?: User) {
  console.log('This will log once per instance of this component');

  if (user) {
    console.log('I log once every time the `user` becomes truthy');

    <>
      <span>Hello, {
        console.log('I log once every time `user.name` changes');
        user.name
      }!</span>
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
