# React Mental Model — A Rails Developer's Guide

## The Big Picture: What Problem Does React Solve?

In Rails, the server owns the HTML. Every user action triggers a request → controller → view → full HTML response. Even with Turbo/Hotwire, the server is the source of truth for what the page looks like.

**React flips this.** The browser owns the UI. Your JavaScript code describes *what the screen should look like right now*, and React figures out how to make the DOM match. The server becomes a pure data API (like your Rails API mode).

---

## 1. Declarative vs Imperative — The jQuery → React Shift

This is the single most important mental shift.

### jQuery (imperative) — you describe the *steps*
```javascript
// "Find the element, then change it, then add this, then remove that"
$('#counter').text(count);
$('#message').show();
$('#old-message').hide();
```

### React (declarative) — you describe the *result*
```jsx
// "Here's what the screen should look like given this data"
function Counter({ count }) {
  return (
    <div>
      <p>{count}</p>
      {count > 0 && <p>You've clicked!</p>}
    </div>
  );
}
```

> [!IMPORTANT]
> **The core contract:** You tell React *what* UI corresponds to *what* data. React handles *when* and *how* to update the DOM. You never touch `document.getElementById()` or `innerHTML`.

### Why Was React Born? The Pain of Imperative DOM at Scale

In 2011-2012, Facebook's web app was a mess. Here's a simplified version of the problem.

Imagine you're building Facebook's notification system with jQuery:

```javascript
// User gets a new message — update the UI
function onNewMessage(message) {
  // Update the message counter in the header
  var count = parseInt($('#msg-count').text()) + 1;
  $('#msg-count').text(count);

  // Show the badge if it was hidden
  if (count > 0) {
    $('#msg-badge').show();
  }

  // Add message to the list if the panel is open
  if ($('#msg-panel').is(':visible')) {
    $('#msg-list').prepend('<div class="msg">' + message.text + '</div>');
  }

  // Update the page title
  document.title = '(' + count + ') Facebook';

  // Also update the chat sidebar count
  var chatUser = $('#chat-' + message.senderId);
  var chatCount = parseInt(chatUser.find('.unread').text() || '0') + 1;
  chatUser.find('.unread').text(chatCount);

  // Oh wait, also update the mobile nav if screen is small
  if (window.innerWidth < 768) {
    $('#mobile-msg-count').text(count);
  }
}
```

**Now imagine dozens of events like this.** Messages, friend requests, likes, comments, live updates — each one modifying multiple pieces of the DOM. Every event handler has to know about every other piece of UI it might affect.

The result? **The notification count bug.** Facebook's notification would show "1" when you had no unread notifications. Or show nothing when you had 3. The UI state and the actual data would drift apart because some code path forgot to update some element.

### The Insight: UI as a Function of State

React's creators (Jordan Walke at Facebook) had a key insight:

> **What if we just re-rendered everything from scratch whenever data changes?**

Instead of carefully mutating individual DOM nodes, what if the entire UI was simply a **function of the current data**?

```
UI = f(state)
```

```jsx
// React approach: just describe what the UI looks like for ANY state
function NotificationHeader({ messages, friendRequests }) {
  const unreadCount = messages.filter(m => !m.read).length;

  return (
    <header>
      <span className="badge">
        {unreadCount > 0 && `(${unreadCount})`}
      </span>
      <h1>{unreadCount > 0 ? `(${unreadCount}) Facebook` : 'Facebook'}</h1>
    </header>
  );
}
```

**No bugs possible in the counting logic.** The count is *derived from the data* every single time. You can't have a stale count because there's no separate counter being manually incremented/decremented — it's always computed fresh from the source of truth.

(But wait — isn't re-rendering everything slow? Yes! That's where the **Virtual DOM** comes in — covered in Section 8 when we look at how re-rendering actually works.)

### The Tradeoff

Nothing is free. Here's what you give up:

| You Gain | You Pay With |
|----------|-------------|
| Predictable UI | Build tooling complexity (Vite, Webpack, etc.) |
| Developer productivity at scale | Larger bundle size sent to browser (~40KB+ for React) |
| Component reusability | Learning curve (new mental model) |
| Ecosystem & community | JavaScript fatigue (new tools every year) |

> [!NOTE]
> **Was it worth it?** For small sites (a blog, simple forms), jQuery is arguably still fine. But for anything with complex, interactive, state-driven UI (dashboards, chat apps, collaborative tools, SPAs) — the declarative model scales dramatically better. That's why React won. Facebook literally couldn't maintain their app without it.

---

## 2. React vs Angular — A Quick Orientation

Angular is a full, opinionated **framework** — like Rails. It ships with routing, forms, an HTTP client, dependency injection, and a CLI, and it has strong opinions about how you structure things. React is just the **view layer** — like picking Sinatra and then choosing your own gems for everything else (routing, data fetching, and forms are all separate libraries you bolt on).

### A Few Conceptual Differences That Matter

| | Angular | React |
|---|---|---|
| Language | TypeScript-first, classes + decorators (`@Component`) | Plain JS/TS, just functions |
| Templates | Its own template syntax (`*ngFor`, `*ngIf`) compiled ahead of time | JSX — literally JavaScript, no separate template language |
| Dependency injection | Built into the framework — services get injected via constructor | No built-in equivalent — you reach for Context or a library if you need it |
| Data binding | Two-way binding available (`[(ngModel)]`) | One-way only — you always explicitly set state, never implicit sync |
| Change detection | Historically dirty-checks the whole tree (via Zone.js); newer versions use Signals for finer-grained updates | Virtual DOM diffing, triggered explicitly by `setState` |

### Where Angular Tends to Win

- **Large teams/enterprises** where you want everyone building things the same way — less bikeshedding, similar to why Rails' "convention over configuration" works well for teams.
- **Complex async/event-driven UIs** — Angular leans on RxJS, which is powerful once learned but has a steep curve.
- **Long-term stability** — Google backs it with predictable LTS versions, which matters for big companies that don't want to rewrite things every 2 years.

### Where React Tends to Win

- **Flexibility** — good for everything from a tiny embedded widget to a huge app, since you're not locked into one router/form/state approach.
- **Ecosystem size and hiring pool** — more jobs, more libraries, more Stack Overflow answers.
- **Faster to pick up and prototype with** — less ceremony, fewer concepts to learn before you ship something.
- **React Native** gives you a shared mental model if you ever touch mobile.
- **Recent stability** — the core API (Hooks) hasn't fundamentally changed since 2019, and React 19 (Dec 2024) was a gentle upgrade rather than a rewrite. Tooling has settled too, with Vite as the default, and Next.js has become the closest thing React has to an opinionated, batteries-included framework if you want that Angular-like experience.

---

## 3. Components — The Building Block

Everything in React is a **component**. A component is just a JavaScript function that returns UI (HTML-like syntax called JSX).

```jsx
function Greeting() {
  return <h1>Hello, world!</h1>;
}
```

### Rails analogy

| React                | Rails Equivalent                           |
|----------------------|--------------------------------------------|
| Component            | A partial + its helper logic, bundled together |
| Component file       | `_greeting.html.erb` + related helper methods |
| Nesting components   | `render partial: 'greeting'` inside another partial |
| Component tree       | Your layout → page template → partials hierarchy |

**Key difference from Rails partials:** A React component bundles **markup + behavior + styling** together. In Rails, these live in separate files (partial, helper, CSS). In React, co-location is the philosophy.

```jsx
// This one file contains structure, logic, AND can import its own styles
function UserCard({ user }) {
  const fullName = `${user.firstName} ${user.lastName}`;

  return (
    <div className="user-card">
      <h2>{fullName}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

---

## 4. JSX — HTML in Your JavaScript

JSX looks like HTML but it's actually JavaScript. It gets compiled to function calls.

```jsx
// What you write:
<h1 className="title">Hello</h1>

// What the compiler turns it into:
React.createElement('h1', { className: 'title' }, 'Hello')
```

### Key JSX gotchas for a Rails dev

| HTML / ERB               | JSX                        | Why                          |
|--------------------------|----------------------------|------------------------------|
| `<%= ruby_expr %>`       | `{jsExpression}`           | Curly braces = "escape to JS" |
| `style="color: red"`     | `style={{ color: 'red' }}` | Style is an object, not a string |
| `<!-- comment -->`        | `{/* comment */}`          | JS comment syntax inside JSX |

#### Why `className` and `htmlFor`?

In HTML, you write `class="title"` and `for="email"`. In JSX, you write `className="title"` and `htmlFor="email"`. There are two reasons.

1. **React props follow DOM property names, not HTML attribute names.** In the DOM, you write `element.className = "title"`. You do not write `element.class`. React uses the same names for the same reason, so `for` becomes `htmlFor`.
2. **The DOM uses these names because `class` and `for` are reserved words.** The DOM API is from the late 1990s. Old JavaScript (ES3) did not let you use a reserved word as a property name. `class` and `for` were reserved. The DOM designers chose `className` and `htmlFor` to avoid that problem.

> [!TIP]
> Think of `{ }` in JSX like `<%= %>` in ERB — it's how you embed dynamic expressions.

### Fragments — Returning Multiple Elements

A component must return one root element. This code fails:

```jsx
function Profile() {
  return (
    <h1>Nitish</h1>
    <p>Developer</p>   // ❌ Error: adjacent JSX elements must be wrapped
  );
}
```

You could wrap the elements in a `<div>`. But that adds an extra node to the DOM, which can break layouts (for example, inside a `<table>` or a flex container). A **Fragment** groups elements without adding a DOM node.

```jsx
// Short syntax (use this most of the time)
function Profile() {
  return (
    <>
      <h1>Nitish</h1>
      <p>Developer</p>
    </>
  );
}
```

```jsx
// Long syntax: needed when you must pass a `key`
import { Fragment } from 'react';

function Glossary({ items }) {
  return (
    <dl>
      {items.map(item => (
        <Fragment key={item.id}>
          <dt>{item.term}</dt>
          <dd>{item.description}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
```

> [!NOTE]
> The short `<>...</>` syntax cannot take props. If a Fragment in a list needs a `key` (see Section 7), use `<Fragment key={...}>`.

### JSX is Optional — What Are the Alternatives?

JSX is syntactic sugar. It compiles down to plain JavaScript function calls. You can write React without JSX — it's just verbose and painful.

#### Alternative A: `React.createElement()` — the raw API

Every JSX tag becomes a `React.createElement()` call. You can write this directly:

```jsx
// ✅ With JSX (what everyone writes)
function Greeting({ name }) {
  return (
    <div className="card">
      <h1>Hello, {name}!</h1>
      <p>Welcome to React</p>
    </div>
  );
}
```

```javascript
// ✅ Without JSX (what the compiler actually produces)
function Greeting({ name }) {
  return React.createElement(
    'div',
    { className: 'card' },
    React.createElement('h1', null, 'Hello, ', name, '!'),
    React.createElement('p', null, 'Welcome to React')
  );
}
```

**Both are 100% equivalent.** JSX is just a developer convenience. The browser never sees JSX — it always runs the `createElement` version.

#### Alternative B: `htm` — tagged template literals

A middle ground that uses JavaScript template literals (no build step needed):

```javascript
import htm from 'htm';
import { createElement } from 'react';

const html = htm.bind(createElement);

function Greeting({ name }) {
  return html`
    <div className="card">
      <h1>Hello, ${name}!</h1>
      <p>Welcome to React</p>
    </div>
  `;
}
```

#### Why does everyone use JSX then?

| Approach               | Readability | Build Step Required? | Used in Practice |
|------------------------|-------------|----------------------|------------------|
| JSX                    | ⭐⭐⭐⭐⭐      | Yes                  | ~99% of projects |
| `React.createElement`  | ⭐⭐          | No                   | Almost never     |
| `htm` (template literals) | ⭐⭐⭐⭐     | No                   | Rare, prototypes |

**Rails analogy:** JSX is to `React.createElement` what ERB is to manual string concatenation. You *could* build HTML with `"<div class=\"card\">" + name + "</div>"` in Ruby — but why would you?

> [!NOTE]
> The fact that JSX requires a build step (a compiler to transform it to `createElement` calls) is exactly why tools like Vite exist. Which brings us to...

---

## 5. Vite & Build Tools — Why They Exist

### The Problem

Browsers don't understand JSX. They don't understand modern import/export syntax well. They don't know how to bundle 200 JavaScript files into something efficient. **You need a tool to transform and bundle your code.**

### The jQuery Era — No Build Tools

In the jQuery days, you didn't need any of this:

```html
<!-- jQuery era: just drop script tags and go -->
<script src="jquery.min.js"></script>
<script src="my-app.js"></script>
<link rel="stylesheet" href="styles.css">
```

That was it. No compilation, no bundling, no transformation. This is analogous to how Rails still works — you write Ruby, the server interprets it. No "build" step for your backend code. The asset pipeline (`sprockets` or `propshaft`) is the closest Rails equivalent to what Vite does.

### The Evolution of JavaScript Build Tools

```
2010-2013   No tools / just <script> tags
     ↓
2013-2015   Grunt / Gulp — task runners (concat files, minify)
     ↓        ↗ Rails equivalent: Sprockets asset pipeline
2015-2019   Webpack — the monster bundler (powerful but complex, slow)
     ↓        ↗ Create React App (CRA) was a wrapper around Webpack
2020+       Vite — the modern, fast alternative
```

### What Does Vite Actually Do?

Vite (French for "fast", pronounced "veet") does several things:

| What | Why | Rails Equivalent |
|------|-----|------------------|
| **Compiles JSX** → JavaScript | Browsers don't understand JSX | ERB → HTML compilation |
| **Bundles modules** | Combines 200 files into a few optimized bundles | Asset pipeline bundling |
| **Dev server with HMR** | Instant updates in browser when you save a file | Like `rails server` + LiveReload, but faster |
| **TypeScript support** | Compiles TS to JS | N/A (Ruby is dynamic) |
| **CSS processing** | Handles imports, PostCSS, Sass | Sass preprocessing in Rails |
| **Production builds** | Minification, tree-shaking, code-splitting | `rails assets:precompile` |

### Why Vite Over Webpack?

**Speed.** Webpack bundles your *entire app* before the dev server starts. For a large app, this could take 30-60 seconds. Vite uses native ES modules in the browser during development — it only transforms files *on demand* as the browser requests them.

```
Webpack dev server:
  Start → Bundle ALL files → Serve → (wait 30s) → Ready

Vite dev server:
  Start → Ready (instant) → Transform files on-demand as browser requests them
```

> [!TIP]
> **Think of Vite as the `rails server` + asset pipeline of the React world.** It's the thing that makes your development experience work. You run `npm run dev`, it starts a server, you edit code, the browser updates instantly.

### Before Vite: Create React App (CRA)

You might see old tutorials mention `create-react-app` (CRA). It was the official React scaffolding tool (2016-2023) built on Webpack. It's now **deprecated**. Vite is the recommended replacement. If you see CRA in a tutorial, mentally substitute it with Vite.

---

## 6. Props — Data Flowing Down

**Props** are how you pass data from a parent component to a child component. They are **read-only**.

```jsx
// Parent passes data down
function App() {
  return <UserCard name="Nitish" role="Developer" />;
}

// Child receives it as a props object
function UserCard({ name, role }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>{role}</p>
    </div>
  );
}
```

### Rails analogy

```ruby
# Rails — passing locals to a partial:
render partial: 'user_card', locals: { name: "Nitish", role: "Developer" }
```

```erb
<!-- _user_card.html.erb -->
<div>
  <h2><%= name %></h2>
  <p><%= role %></p>
</div>
```

**It's the same concept.** Props = locals. Parent decides what data the child gets. The child cannot modify its own props, just like a partial can't change the locals passed to it.

> [!IMPORTANT]
> **Props flow ONE way: parent → child.** A child can never send props back up. This is called **unidirectional data flow** and is the most important architectural rule in React. (We'll cover how children communicate back up shortly.)

### Can I Pass Any Prop to a Native DOM Element?

You can pass any prop to **your own components**. You cannot do the same with **native DOM elements** (`<div>`, `<input>`, and so on).

```jsx
<UserCard blahblahblah={true} />   // ✅ Fine: UserCard decides what to do with the prop
<div blahblahblah={true}>Hi</div>  // ❌ React warns and does not render the attribute
```

**Why?** JSX tags like `<div>` become real DOM elements. A real DOM element only understands the attributes and properties in the HTML specification (`id`, `className`, `href`, `disabled`, and so on). `blahblahblah` is not in that list. React does not know what to do with it, so it ignores it and prints a warning in the console.

> [!TIP]
> To attach custom data to a DOM element, use a `data-*` attribute: `<div data-user-id={42}>`. The HTML specification allows these, and React passes them through to the DOM.

> [!NOTE]
> **`onClick` on a custom component is just a prop.** React gives `onClick` special meaning only on native DOM elements. In `<Star onClick={toggleFavorite} />`, React passes `onClick` into `Star` like any other prop. Nothing happens unless `Star` uses it:
>
> ```jsx
> function Star({ isFilled, onClick }) {
>   return <button onClick={onClick}>{isFilled ? '★' : '☆'}</button>;
> }
> ```
>
> The name `onClick` is only a convention. You can use any name, for example `onToggle`.

---

## 7. Rendering Lists — `.map()` and the `key` Prop

In ERB you loop with `each`. In JSX you use `.map()`, because JSX embeds *expressions* — you need something that **returns** an array of elements, not a block that iterates.

```erb
<!-- Rails -->
<ul>
  <% @todos.each do |todo| %>
    <li><%= todo.text %></li>
  <% end %>
</ul>
```

```jsx
// React
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

### The `key` Requirement

Notice the `key={todo.id}`. React **requires** a `key` on every element you render in a loop — it warns in the console if you forget.

**Why?** Remember the Virtual DOM diff from Section 1: when the list re-renders, React compares the new list of elements against the old one. Without a stable identity per item, React can't tell the difference between:

- "item 2 changed its text", and
- "item 2 was deleted and a different item was inserted in its place"

The `key` gives each element a stable identity across renders, so the diff can match old items to new items and make the minimal DOM change.

> [!IMPORTANT]
> **`key` is a primary key for the Virtual DOM diff.** Use your database ID (`todo.id`) — the same value you'd use as a primary key in Rails. It must be stable (same item = same key on every render) and unique among its siblings.

### Why Not the Array Index?

You'll be tempted to write `key={index}`. Don't — it works only until the list reorders or you delete from the middle. Then the indexes shift, React matches the *wrong* old items to new items, and you get bugs like input text jumping to a different row. Index-as-key is like using a row's position in a result set as its primary key: fine until the data moves.

---

## 8. State — Data That Changes Over Time

**State** is data that a component *owns* and can *change*. When state changes, React **automatically re-renders** the component (re-calls the function and updates the DOM).

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  //      ↑          ↑              ↑
  //   current    function to    initial
  //    value     update it       value

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

> [!WARNING]
> **State updates are batched — `count` is a snapshot.** Inside an event handler, `count` holds the value from the current render, so this only increments once:
>
> ```jsx
> setCount(count + 1);  // "set count to 0 + 1"
> setCount(count + 1);  // "set count to 0 + 1" — same thing again!
> ```
>
> When the next value depends on the previous one, pass a function instead — React calls it with the latest value:
>
> ```jsx
> setCount(c => c + 1);  // 0 → 1
> setCount(c => c + 1);  // 1 → 2 ✓
> ```

> [!TIP]
> **Best practice:** When the new state depends on the old state, always use the callback form. It is safe with batching, timers, and async code. It also keeps state out of `useEffect` dependency arrays. You do not need it when the new value is independent of the old one, such as `setName('Nitish')` or `setText(e.target.value)`.

### The Render Cycle — How React "reacts"

```
User clicks button
       ↓
setCount(count + 1) is called
       ↓
React sees state changed
       ↓
React re-calls your component function (with new state)
       ↓
Gets new JSX output
       ↓
Compares with previous output (Virtual DOM diffing)
       ↓
Updates ONLY the parts of the real DOM that changed
```

### The Virtual DOM — Making Declarative Fast

Remember the insight from Section 1: `UI = f(state)`, re-render everything on every change. But nuking the entire DOM and rebuilding it on every state change would be catastrophically slow. DOM operations are expensive (the browser has to recalculate layouts, repaint pixels, etc.).

React's solution: **diff in memory, apply minimally.**

```
Step 1: State changes (e.g., new message arrives)
         ↓
Step 2: React calls your component function again
         ↓
Step 3: Gets new JSX → converts to a new Virtual DOM tree
        (this is just a lightweight JavaScript object, very fast)
         ↓
Step 4: React DIFFS the new tree against the previous tree
        (like running `git diff` between two commits)
         ↓
Step 5: Computes the minimal set of actual DOM changes needed
        (e.g., "change text of this one <span> from '2' to '3'")
         ↓
Step 6: Applies ONLY those changes to the real DOM
        (one surgical DOM update instead of rebuilding everything)
```

> [!NOTE]
> **Virtual DOM:** React keeps a lightweight copy of the DOM in memory. When state changes, it computes the new UI, diffs it against the old one, and applies the minimum set of changes. Think of it like `git diff` for the DOM — it only applies the delta.

---

## 9. Props vs State — The Critical Distinction

|                   | Props                              | State                               |
|-------------------|------------------------------------|--------------------------------------|
| **Who owns it?** | Parent component                   | The component itself                 |
| **Mutable?**     | No (read-only)                    | Yes (via setter function)           |
| **Rails analogy** | `locals:` passed to a partial      | `@instance_variable` on a model     |
| **Triggers re-render?** | Yes (if parent re-renders with new props) | Yes (when setter is called) |
| **Purpose**      | Configure a component from outside | Track data that changes over time    |

---

## 10. Events & Child → Parent Communication

Since props only flow down, how does a child tell the parent something happened?

**The parent passes a callback function as a prop.** The child calls it.

```jsx
function Parent() {
  const [items, setItems] = useState([]);

  const handleAddItem = (item) => {
    setItems([...items, item]);
  };

  return <AddItemForm onAdd={handleAddItem} />;
}

function AddItemForm({ onAdd }) {
  const [text, setText] = useState('');

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      onAdd(text);     // ← calling the parent's function
      setText('');
    }}>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button type="submit">Add</button>
    </form>
  );
}
```

### Rails analogy

This is like a Rails **callback / hook pattern:**
```ruby
# Parent decides what happens
child_service.on_complete = ->(result) { update_records(result) }
child_service.run
```

The child doesn't know or care what the parent does — it just calls the function it was given.

---

## 11. Hooks — The Modern React API

Hooks are special functions (all prefixed with `use`) that let you "hook into" React features. You've already seen `useState`. Here are the essential ones:

| Hook            | Purpose                                     | Rails-ish Analogy                              |
|-----------------|---------------------------------------------|-------------------------------------------------|
| `useState`      | Manage component-local state                 | Instance variable with auto-refresh             |
| `useEffect`     | Side effects (API calls, subscriptions, etc) | `after_action` / `after_commit` callbacks       |
| `useContext`    | Share data without prop-drilling             | `Current.user` / thread-local globals           |
| `useRef`        | Hold a mutable value that doesn't trigger re-render | A utility variable you don't want to watch |
| `useMemo`       | Cache expensive computations                 | Memoization / `Rails.cache.fetch`               |

### useRef — Why `.current`?

`useRef` returns a plain object with one property: `{ current: initialValue }`. You read and write the value through `.current`.

React returns the **same object** on every render. Your component function runs again on each render, so local variables reset. The ref object survives. You change `.current` on that one object, and every render sees the new value.

A plain value cannot do this. JavaScript passes numbers and strings by value. A change to a plain variable would only change your local copy. The wrapper object fixes this.

```jsx
function Timer() {
  const idRef = useRef(null);   // { current: null }, same object every render

  function start() {
    idRef.current = setInterval(tick, 1000);  // changes the property, not the object
  }
  function stop() {
    clearInterval(idRef.current);             // reads the latest value
  }
  // ...
}
```

Changing `.current` does **not** cause a re-render. React does not watch the object. React re-renders only when you call a state setter.

React also uses a ref to give you a DOM element. In `<input ref={inputRef} />`, React sets `inputRef.current` to the real `<input>` node after the render. Then you can call `inputRef.current.focus()`.

> [!TIP]
> Use **state** for values that change what the screen shows. Use a **ref** for values that React does not need to show, such as a timer ID or a DOM node.

### useEffect — The Side Effect Hook

`useEffect` runs *after* your component renders. It's where you do things like fetch data from APIs.

```jsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // This runs after the component mounts (and when userId changes)
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data));
  }, [userId]);  // ← dependency array: re-run only when userId changes
  //    ↑
  //  This is the key — it controls WHEN the effect runs

  if (!user) return <p>Loading...</p>;
  return <h1>{user.name}</h1>;
}
```

> [!WARNING]
> `useEffect` is the most common source of bugs for React beginners. The dependency array (`[userId]`) tells React when to re-run the effect. Missing dependencies = stale data. Extra dependencies = infinite loops. We'll practice this.

> [!NOTE]
> **With several dependencies, any one change re-runs the effect.** React runs the effect again when any one of the dependencies changes. All of them do not need to change.

> [!NOTE]
> **Your effect will run twice in development — that's not a bug.** A fresh Vite scaffold wraps your app in `<StrictMode>`, which deliberately mounts every component twice in development to expose effects that don't clean up after themselves. So you *will* see your `fetch` fire twice in the network tab. This only happens in dev — production builds run effects once. Don't "fix" it by removing StrictMode; treat it as a lint for effect hygiene.
>
> **How it helps:** React runs your effect, then its cleanup, then the effect again. An effect with no cleanup shows its bug at once:
>
> ```jsx
> // ❌ No cleanup: StrictMode starts two intervals, so the counter goes up by 2 each second
> useEffect(() => {
>   const id = setInterval(() => setCount(c => c + 1), 1000);
> }, []);
>
> // ✅ Cleanup stops the first interval before the second one starts
> useEffect(() => {
>   const id = setInterval(() => setCount(c => c + 1), 1000);
>   return () => clearInterval(id);
> }, []);
> ```
>
> In production, the same leak happens when the component unmounts, for example when the user goes to another page. The interval keeps running with no component.

---

## 12. The Component Tree — Thinking in React

A React app is a **tree of components**. Data flows down through props. Events flow up through callbacks.

```
        App                    ← owns top-level state
       /   \
    Header  TodoList           ← receive state as props
              |
          TodoItem (×N)        ← receive individual items as props
              |                   call onDelete/onToggle callbacks up
          [Delete Button]
```

### The Rails comparison

```
ApplicationLayout              ← application.html.erb
       |
   PageTemplate                ← show.html.erb
      / \
  _header  _todo_list          ← partials
              |
          _todo_item (×N)      ← partial in a loop
```

**The shape is identical.** The difference is that in React, this tree is *alive in the browser* and updates in real-time without round-tripping to the server.

### Lifting State Up — Where Does State Live?

The React answer to "where do I put this?": **state lives in the closest common parent of every component that needs it.**

Say a search box and a results list both need the current query. Neither can own it — siblings can't see each other's state. So the state moves *up* to their shared parent:

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');   // ← state lives here

  return (
    <>
      <SearchBox query={query} onQueryChange={setQuery} />
      <ResultsList query={query} />
    </>
  );
}
```

`SearchBox` gets the value *and* a callback to change it. `ResultsList` just gets the value. Neither sibling owns anything — the parent does, and the data flows down.

This is called **lifting state up**, and it's the standard refactoring move in React: start with state in the component that uses it; the moment a second component needs the same data, lift the state to their closest common ancestor.

**Rails analogy:** It's like moving shared logic out of two views and into the controller action that renders both. The data moves up to the layer that "sees" both consumers, then flows down.

---

## 13. Mental Model Summary

Here's the complete picture in one sentence:

> **A React app is a tree of functions (components) that take data in (props), manage their own changing data (state), and return UI (JSX). When state changes, React re-runs the affected functions and efficiently updates the DOM.**

### How All the Pieces Fit Together

```
                    The React Philosophy
                    ═══════════════════

jQuery era:   "The DOM is the source of truth. Query it, mutate it."
                     ↓ (doesn't scale)
React:        "Data is the source of truth. The DOM is a projection of data."
                     ↓ (needs to be fast)
Virtual DOM:  "Re-render the projection in memory, diff it, apply minimal changes."
                     ↓ (needs tooling to compile JSX)
Vite:         "I'll compile JSX, bundle modules, and give you instant dev feedback."
```

This is why these pieces exist and how they fit together.

---

## What's Next?

Now that you have the mental model, a good path forward:

1. **Set up a Vite + React project** in this repo
2. **Build a small app** (maybe a task list?) — applying each concept above
3. **Layer in complexity** — API calls, routing, forms
