Ah, you're right — I got cut off. Let me pick up properly and give you the full front-end rundown with real examples and likely pushback, as you asked.

---

## FRONT-END — React 18, TypeScript, MUI, AG Grid

**Honest frame to open with:** *"Day-to-day on Montana I work in Aurelia — component-based, two-way binding, DI — so a lot of the concepts transfer. I've been building the React 18 side hands-on in DocIntel with TypeScript, MUI and AG Grid, deliberately, because it's the stack I want to be in."*

### 1. `useEffect` — dependency array + cleanup (the #1 probe)

**What to say:** *"`useEffect` runs a side effect after render. The dependency array controls when it re-runs — empty array means once on mount, no array means every render, and `[value]` means it re-runs when `value` changes. If the effect starts async work, the cleanup function — the thing you `return` — runs before the effect re-runs or when the component unmounts, so I can cancel or ignore stale work."*

**Real example — debounced search in DocIntel:**
```jsx
useEffect(() => {
  const timer = setTimeout(() => {
    fetchDocuments(searchTerm);
  }, 300);

  return () => clearTimeout(timer); // cancels previous timer on every keystroke
}, [searchTerm]);
```

**Likely pushback:** *"What happens if you forget the dependency array entirely?"*
**Answer:** *"It runs on every render, which usually means an infinite loop if the effect itself triggers a re-render — like calling `setState` inside it with no guard."*

**Likely pushback:** *"Why not just fetch inside the onChange handler instead of useEffect?"*
**Answer:** *"You could, but useEffect keeps the fetch logic tied to the state it depends on, not the event that triggered it — so if searchTerm changes some other way (URL param, external reset), the effect still fires. It's more declarative: 'whenever this data changes, do this,' rather than 'whenever this event fires, do this.'"*

---

### 2. Props vs state, lifting state up

**What to say:** *"Props flow one-way, parent to child. State is owned wherever it needs to be shared. If two sibling components both need the same data, I lift the state to their nearest common parent, and pass it down as props plus a setter callback for children to request changes."*

**Real example:** *"In DocIntel, a `SearchBox` and `DocumentList` both need the search term. A parent `DocumentSearch` component owns the state; `SearchBox` gets `searchTerm` and `onSearchChange` as props, `DocumentList` gets `searchTerm` to trigger its fetch."*

**Likely pushback:** *"What if the state needs to be shared across many components, not just a couple of siblings?"*
**Answer:** *"That's when lifting state up gets unwieldy — you'd reach for Context to avoid prop drilling, or an external store like Redux/Zustand if there's real business logic and actions involved, not just shared reads."*

**Aurelia bridge:** *"Aurelia defaults to two-way binding — a child can update a parent's bound property directly. React is deliberately one-way — a child can only ask the parent to change state via a callback. I've found one-way flow makes state easier to reason about, because you always know where a change originated."*

---

### 3. Controlled vs uncontrolled components

**What to say:** *"Controlled means React state owns the input value — you set it via `value` and update it via `onChange`. Uncontrolled means the DOM owns it, and you read it via a ref only when you need it, like on submit."*

**Real example — DocIntel search box:** *"I'd make it controlled. Even though I don't want to fire the actual AI search on every keystroke, I still want to react to every keystroke for things like a character count or showing 'searching...' states. So I update state on every keystroke, but defer the expensive fetch to a submit handler or a debounce timer."*

**Likely pushback:** *"When would uncontrolled make more sense?"*
**Answer:** *"Something like a file upload input, or a large form where I genuinely only care about the value at submit time and don't need to react to changes as they happen — less re-rendering overhead."*

---

### 4. `useMemo` vs `useCallback` vs `useEffect` — and when NOT to use them

**What to say:** *"These are three different tools. `useMemo` memoizes a computed *value* — skip recomputation if dependencies haven't changed. `useCallback` memoizes a *function reference* — useful only when passing a function to a `React.memo`-wrapped child, so the child doesn't see a 'new' prop every render. `useEffect` is for side effects entirely — data fetching, subscriptions — not for memoization at all."*

**The honest line (this is the good instinct to lead with):** *"I wouldn't reach for `useMemo` or `useCallback` by default — they have their own overhead. I'd measure first with the React DevTools profiler. AG Grid already virtualizes rows, so most cells aren't even rendering. If I saw a real bottleneck, then I'd memoize — not before."*

**Likely pushback:** *"Isn't more memoization always safer, even if unnecessary?"*
**Answer:** *"No — premature optimization here is a real cost, not just a null cost. Every `useMemo`/`useCallback` call still runs a dependency comparison on every render, and it adds cognitive overhead for anyone reading the code. If there's no measured problem, it's just noise."*

---

### 5. `useContext` vs prop drilling

**What to say:** *"Prop drilling is passing data through several component layers that don't use it themselves, just to reach a deeply nested child. `useContext` solves it — you provide a value once at the top with a Context Provider, and any descendant can read it directly with `useContext`, no matter how deep."*

**Real example:** *"DocIntel's `currentUser` — needed by the navbar, sidebar, and search results. Rather than threading it through `App → Layout → Navbar`, I'd wrap the tree in a `UserContext.Provider` and each component that needs it calls `useContext(UserContext)` directly."*

**Aurelia bridge:** *"Aurelia solves this with dependency injection — inject a shared service wherever you need it. React doesn't have built-in DI, so Context fills that role. Same goal — avoid threading data through components that don't care about it — different mechanism."*

**Likely pushback:** *"Why not just use Context for everything and skip local state entirely?"*
**Answer:** *"Context causes every consumer to re-render when the value changes, even if only a small piece of it changed. For truly local state — like a single input's value — local `useState` is cheaper and simpler. Context is for genuinely shared, cross-cutting data."*

---

### 6. TypeScript

**What to say:** *"TypeScript is JavaScript plus optional static types, checked at compile time, not runtime. It catches shape mismatches before you ship — passing a string where a number's expected, calling a property that doesn't exist on an object."*

**Real example — the FastAPI bridge:** *"On DocIntel, TypeScript on the frontend mirrors the Pydantic models on the FastAPI backend. If the API returns a `Document` with `id`, `title`, `created_at`, I define a matching TypeScript interface. That way the frontend and backend stay in sync — typed end-to-end."*

**Likely pushback:** *"Doesn't TypeScript slow you down for a small project?"*
**Answer:** *"There's upfront cost, yes, but it pays off fast once the API surface grows — catching a mismatched field name at compile time instead of a runtime crash in production is worth it, especially working solo without someone else to catch it in review."*

---

### 7. MUI and AG Grid

**What to say for MUI:** *"Theming and the `sx` prop for one-off styling overrides, without writing separate CSS files."*

**What to say for AG Grid:** *"Column definitions, cell renderers for custom display logic, built-in sorting and filtering. The big one is **client-side vs server-side row model** — client-side loads all data into the browser and AG Grid handles sort/filter/paginate locally; server-side only fetches what's visible, and sort/filter operations trigger new API calls. For large datasets you need server-side, plus row virtualization so you're never rendering more DOM nodes than fit on screen."*

**Real example:** *"Few candidates have actually used AG Grid — I have, in DocIntel, for the document results table. That's a genuine differentiator, worth leading with rather than downplaying."*

**Likely pushback:** *"When would you NOT use the server-side row model?"*
**Answer:** *"If the dataset is small enough to comfortably fit in memory — a few thousand rows — client-side is simpler, no extra API design for sort/filter/paginate params. Server-side earns its complexity at real scale."*

---

### 8. Framework-agnostic front-end fundamentals (your strongest zone)

Quick-fire, know these cold since they're framework-independent and you have real years behind them:

- **Debounce vs throttle:** debounce waits for a pause before acting (search-as-you-type); throttle limits to once per interval regardless of activity (scroll handlers).
- **Reflow vs repaint:** reflow recalculates layout (expensive — changing width/height/position), repaint just redraws pixels (cheaper — changing color).
- **XSS/CSRF:** XSS = injecting malicious script via unescaped user input; CSRF = tricking a logged-in user's browser into making an unwanted request. Ties to your JWT/auth knowledge.
- **Code splitting/lazy loading:** load JS bundles on demand rather than all upfront, for faster initial page load.

---

### The single most valuable line to have ready

*"Day-to-day I work in Aurelia — two-way binding, built-in DI. React deliberately does one-way data flow with no built-in DI. I've found one-way flow makes state easier to reason about, and I've built the React 18/TypeScript/MUI/AG Grid stack hands-on in DocIntel specifically to close that gap for this role."*

That one line does triple duty: shows real experience, shows you understand *why* frameworks differ (not just that they do), and bridges honestly to what you've built.

Want me to keep going with anything else from the doc, or is this a good stopping point for today?