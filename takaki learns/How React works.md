---
type: concept
---
## React Overview:
1: My JSX
2: Compiled to JS function calls (Because each jsx is a function)
3: Creates React elements objects
4: React builds/update fiber tree
5: React compares old tree vs new tress
6: React commits neccessary changes to real DOM
7: Browser piants pixels

## 0: VITE and launching local server
npm run dev (vite) => starts http server like http://localhost:5173

then broswer opens and visit "http://localhost:5173", it gives it back "index.html" which might contains:
```html
<div id="root"></div>
<script type="module" src="/src/main.jsx"></script>
```

but browser doesn't understand main.jsx
```
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";

ReactDOM.createRoot(document.getElementById("root")).render(
  <App />
);
```



Vite transforms it into jsx(App, {})

```
Your source <App />
│ 
▼ 
Vite transformation 
│ 
▼ 
JavaScript jsx(App, {}) 
│ 
▼
Browser
```

then React is downloaded in the browser and ready to do its job in the browser...


## 1 JSX
**jsx becomes js obejects that describes UIs**
- First it becomes function calls
EX)
```html
<div className="card">
  <h1>Counter</h1>
  <p>Count: {count}</p>
</div>
```
||
 V
```javascript
jsx("div, {
	className: "card",
	childres: [
	  jsx("h1", { children: "Counter"}), 
	  jsx("p", {children: `Count: ${count}`,})
	]
})
```

JSX is a convenient way to create js objects

**jsx create objects**
```
<h1 className="title">Hello</h1>
```

==> 
{ type: "h1", props: { className: "title", children: "Hello" } }

If you have a custom component:
```
<UserCard name="takaki" />
```

It becomes something like:
{ type: UserCard, props: {name: "takaki"} }
=> This UserCard is a function jsx needs to call

```
function UserCard(name) {
return <div><p>{name}</p></div>
}
```

{ type: "div", props: { children : [ type: "p", children: name] }}


## React Rendering

Real html has:
```
<div id="root"></div> 
```

React receives <Counter /> component

then, calls Counter(), returns ```
```
<div></div>
``` 
create DOM nodes and inserts div...

If Update happens, react diff new vs old and only update the newly updated part

```
Render phase - Call components - Create new React elements - Compare with previous tree - Decide what needs to change - Should be pure Commit phase - Actually change the real DOM - Attach refs - Run layout effects - Browser can then paint
```


## React Virtual Dom: Fiber
Fiber is like a React's internal work unit for a component or dom node that remembers work, state, effects, DOM link 

**React can preserv states between renders thanks to fiber

Fiber remembers like:
- This component has useState value 0 
- This DOM node already exists 
- This child moved 
- This text changed 
- This effect should run after commit

## Key matters in list
It identifies the element and React won't be confused

Keys shouldn't be a list keys
EX)
```
{users.map((user, index) => ( <UserRow key={index} user={user} /> ))}
```

At first:
key=0 : "Aki"
key=1 : "Bob"

After Aki is removed
key=0 : "Bob"

React might give Aki's previous component/Fiber state for Bob

## State Update Big Visual:
```
function App() {
  const [count, setCount] = React.useState(0);
  const [theme, setTheme] = React.useState("light");

  return (
    <main className={theme}>
      <h1>Dashboard</h1>

      <section>
        <p>Count: {count}</p>
        <button onClick={() => setCount(count + 1)}>
          Increment
        </button>
      </section>

      <button onClick={() => setTheme("dark")}>
        Dark mode
      </button>
    </main>
  );
}
```

**First Render:**
main.ligh
	h1: "Dashboard"
	 section
	    p: "Count: 0"
		 button: "Increment"
	  button: "Dark mode"

**Second Render after user clicks**
main.ligh
	h1: "Dashboard"
	 section
	    p: "Count: 1"
		 button: "Increment"
	  button: "Dark mode"

**React Diff**
main className: same h1 text: same section: same p text: changed button text: same button handlers: maybe updated internally

#######
React render: React calls your components and calculates UI changes. React commit: React changes the DOM. Browser paint: Browser visually draws pixels on screen.

## Summary
- JSX is not HTML. 
- JSX becomes React elements. 
- React elements are not DOM nodes. 
- They are descriptions. 
- React components are functions. 
- They return UI descriptions. 
- React render does not mean DOM replacement. 
- It means recalculating the UI tree. 
- React commit means DOM mutation. 
- Fiber is React's internal memory/work structure. 
- It lets React preserve state, schedule work, and track updates. 
- Keys define identity in lists. 
- Bad keys cause state bugs. 
- State belongs to a component's position in the tree. 
- Change position/type/key, and state may reset.
