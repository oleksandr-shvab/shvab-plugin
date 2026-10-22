# React Best Practices — Code Examples

Good/bad patterns for each rule in [SKILL.md](SKILL.md).

---

## Derive, Don't Store

```jsx
// BAD: Storing derived state
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);

// GOOD: Compute during render
const fullName = `${firstName} ${lastName}`;
```

```jsx
// BAD: Filtering in useEffect
const [filteredBlogs, setFilteredBlogs] = useState(blogs);
useEffect(() => {
  setFilteredBlogs(blogs.filter(b => b.category === selected));
}, [blogs, selected]);

// GOOD: Compute (memoize only if expensive)
const filteredBlogs = useMemo(
  () => blogs.filter(b => b.category === selected),
  [blogs, selected],
);
```

---

## Memoization

```jsx
// BAD: Over-memoizing trivial operations
const greeting = useMemo(() => `Hello, ${name}!`, [name]);
const handleClick = useCallback(() => setOpen(true), []);

// GOOD: Only memoize expensive operations
const sortedBlogs = useMemo(
  () => [...blogs].sort((a, b) => new Date(b.date) - new Date(a.date)),
  [blogs],
);
```

---

## Render Factories

```jsx
// BAD: Render factory (camelCase, called as function)
const renderBlogCard = (blog) => {
  return <div className="p-4">{blog.title}</div>;
};
return <div>{renderBlogCard(blog)}</div>;

// GOOD: Proper React component (PascalCase, used as JSX)
const BlogCard = ({ blog }) => {
  return <div className="p-4">{blog.title}</div>;
};
return <div><BlogCard blog={blog} /></div>;
```

---

## Inline Creation in JSX

```jsx
// BAD: New array on every render
<CategoryFilter categories={['Tech', 'Startup', 'Lifestyle']} />

// GOOD: Stable reference (module-level constant)
const CATEGORIES = ['Tech', 'Startup', 'Lifestyle'];
<CategoryFilter categories={CATEGORIES} />
```

```jsx
// BAD: Inline style object on every render
<div style={{ padding: '16px', background: '#fff' }}>

// GOOD: Use the project's styling system (Tailwind utility classes shown)
<div className="p-4 bg-white">
```

---

## Container / Presenter Split

```jsx
// BAD: Mixed data fetching and rendering
const BlogList = () => {
  const { data, loading, error } = useBlogs();
  if (loading) return <Loader />;
  if (error) return <p>Error loading blogs</p>;
  // ... 150 lines of rendering logic
};

// GOOD: Container fetches, presenter renders
const BlogListContainer = () => {
  const { data, loading, error } = useBlogs();
  if (loading) return <Loader />;
  if (error) return <p>Error loading blogs</p>;
  if (!data?.blogs?.length) return <p>No blogs found</p>;
  return <BlogGrid blogs={data.blogs} />;
};

const BlogGrid = ({ blogs }) => {
  // Pure rendering — no data fetching, no effects
  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
      {blogs.map(blog => <BlogCard key={blog._id} blog={blog} />)}
    </div>
  );
};
```

---

## State Colocation

```jsx
// BAD: State lifted too high — parent re-renders everything
const Home = () => {
  const [searchTerm, setSearchTerm] = useState('');
  return (
    <>
      <SearchBar value={searchTerm} onChange={setSearchTerm} />
      <BlogList /> {/* re-renders on every keystroke */}
      <Newsletter />
    </>
  );
};

// GOOD: State pushed down to where it's used
const Home = () => (
  <>
    <SearchSection /> {/* owns its own search state */}
    <BlogList />
    <Newsletter />
  </>
);

const SearchSection = () => {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedTerm = useDebounce(searchTerm, 300);
  return <SearchBar value={searchTerm} onChange={setSearchTerm} />;
};
```

---

## Data Fetching in Custom Hooks

```jsx
// BAD: Fetching directly in component
const BlogDetail = ({ id }) => {
  const [blog, setBlog] = useState(null);
  useEffect(() => {
    blogApi.getBlog(id).then(res => setBlog(res.data));
  }, [id]);
  // ...
};

// GOOD: Use a custom hook
const BlogDetail = ({ id }) => {
  const { data, loading, error } = useBlog(id);
  if (loading) return <Loader />;
  if (error) return <p>Error</p>;
  return <BlogContent blog={data.blog} />;
};
```

---

## useEffect Misuse

```jsx
// BAD: useEffect for event handling
const handleSubmit = () => {
  setSubmitted(true);
};
useEffect(() => {
  if (submitted) {
    api.createBlog(formData);
    setSubmitted(false);
  }
}, [submitted]);

// GOOD: Logic in the event handler
const handleSubmit = async () => {
  await api.createBlog(formData);
  toast.success('Blog created!');
  navigate('/admin/blogs');
};
```

---

## Early Returns for States

```jsx
// BAD: Nested ternaries
return loading ? <Loader /> : error ? <Error /> : data ? <Content data={data} /> : <Empty />;

// GOOD: Early returns
if (loading) return <Loader />;
if (error) return <p className="text-red-500">Something went wrong</p>;
if (!data?.blogs?.length) return <p>No blogs yet</p>;
return <BlogGrid blogs={data.blogs} />;
```

---

## Error Boundaries

```jsx
import { ErrorBoundary } from 'react-error-boundary';
import { useLocation } from 'react-router-dom'; // React Router shown; use your router's equivalent

// BAD: No error boundary — unhandled errors crash the whole app
const App = () => <Routes>...</Routes>;

// GOOD: Error boundary with route-aware reset and recovery
const ErrorFallback = ({ error, resetErrorBoundary }) => (
  <div className="p-8 text-center">
    <p className="text-red-500">Something went wrong</p>
    <button onClick={resetErrorBoundary} className="mt-4 px-4 py-2 bg-blue-500 text-white rounded">
      Try again
    </button>
  </div>
);

const App = () => {
  const location = useLocation();
  return (
    <ErrorBoundary resetKeys={[location.pathname]} FallbackComponent={ErrorFallback}>
      <Routes>...</Routes>
    </ErrorBoundary>
  );
};
```

---

## Key Prop Patterns

```jsx
// BAD: Index as key — causes state bugs when list changes
{blogs.map((blog, index) => <BlogCard key={index} blog={blog} />)}

// GOOD: Stable unique ID as key
{blogs.map((blog) => <BlogCard key={blog._id} blog={blog} />)}

// BAD: Random key — forces full remount every render
{blogs.map((blog) => <BlogCard key={Math.random()} blog={blog} />)}

// GOOD: Key on Fragment when mapping fragments
{items.map((item) => (
  <React.Fragment key={item.id}>
    <dt>{item.label}</dt>
    <dd>{item.value}</dd>
  </React.Fragment>
))}
```

---

## Conditional Rendering Gotcha

```jsx
// BAD: Renders literal "0" when count is 0
{count && <Badge>{count}</Badge>}

// GOOD: Explicit comparison
{count > 0 && <Badge>{count}</Badge>}

// GOOD: Ternary for clarity
{count ? <Badge>{count}</Badge> : null}
```

---

## Accessibility

```jsx
// BAD: Icon button without label — invisible to screen readers
<button onClick={onDelete}><TrashIcon /></button>

// GOOD: Accessible icon button
<button onClick={onDelete} aria-label="Delete blog post"><TrashIcon /></button>

// BAD: Error not associated with field
<input type="email" />
{error && <span className="text-red-500">{error}</span>}

// GOOD: Error linked to field
<input type="email" aria-invalid={!!error} aria-describedby="email-error" />
{error && <span id="email-error" className="text-red-500">{error}</span>}

// GOOD: Live region for dynamic updates
<div aria-live="polite">{searchResults.length} results found</div>
```

---

## Route-Level Code Splitting

```jsx
import { lazy, Suspense } from 'react';

// BAD: All pages in main bundle
import AdminDashboard from './pages/admin/Dashboard';
import AddBlog from './pages/admin/AddBlog';

// GOOD: Lazy-loaded routes
const AdminDashboard = lazy(() => import('./pages/admin/Dashboard'));
const AddBlog = lazy(() => import('./pages/admin/AddBlog'));

const App = () => (
  <Suspense fallback={<PageLoader />}>
    <Routes>
      <Route path="/admin" element={<AdminDashboard />} />
      <Route path="/admin/add" element={<AddBlog />} />
    </Routes>
  </Suspense>
);
```

---

## Request Cancellation

```jsx
// BAD: No cleanup — stale responses update unmounted component
useEffect(() => {
  axios.get('/api/blogs').then(res => setBlogs(res.data));
}, []);

// GOOD (Axios): Cancel on cleanup with AbortController
useEffect(() => {
  const controller = new AbortController();
  axios.get('/api/blogs', { signal: controller.signal })
    .then(res => setBlogs(res.data.blogs))
    .catch(err => {
      if (!axios.isCancel(err)) setError(err.message);
    });
  return () => controller.abort();
}, []);

// GOOD (fetch): Same pattern — pass the signal, ignore the abort error
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/blogs', { signal: controller.signal })
    .then((res) => res.json())
    .then((data) => setBlogs(data.blogs))
    .catch((err) => {
      if (err.name !== 'AbortError') setError(err.message);
    });
  return () => controller.abort();
}, []);
```

---

## React 19: ref as Prop

```jsx
// OLD (React 18): forwardRef boilerplate
const Input = forwardRef((props, ref) => (
  <input ref={ref} {...props} />
));

// NEW (React 19): ref as a regular prop
const Input = ({ ref, ...props }) => (
  <input ref={ref} {...props} />
);
```

---

## Derive, Don't Store: Minimal State

```jsx
// BAD: Storing the rendered shape — every update must re-run the derivation
const [rows, setRows] = useState([]); // one { letter, status }[] per guess

// GOOD: Store the minimal shape, derive the rest during render
const [guesses, setGuesses] = useState([]); // ['crane', 'slate']
const rows = guesses.map((guess) => checkGuess(guess, answer));
const isWon = guesses.at(-1) === answer;
```

---

## State Immutability

```jsx
// BAD: Mutating state in place — same reference, React may skip the update
const handleAdd = (label) => {
  items.push({ id: crypto.randomUUID(), label });
  setItems(items);
};
const sorted = items.sort((a, b) => a.label.localeCompare(b.label)); // sort() mutates `items`

// GOOD: A new array every time
const handleAdd = (label) => setItems([...items, { id: crypto.randomUUID(), label }]);
const handleRemove = (id) => setItems(items.filter((item) => item.id !== id));
const handleRename = (id, label) =>
  setItems(items.map((item) => (item.id === id ? { ...item, label } : item)));
const sorted = items.toSorted((a, b) => a.label.localeCompare(b.label));
```

```jsx
import { produce } from 'immer';

// BAD: Spreading every level just to flip one nested flag
setSettings({
  ...settings,
  profile: {
    ...settings.profile,
    notifications: { ...settings.profile.notifications, email: false },
  },
});

// GOOD: Immer — write the mutation, get a new immutable object
setSettings(
  produce((draft) => {
    draft.profile.notifications.email = false;
  }),
);
```

---

## State Ownership

```jsx
// BAD: Child receives the raw setter and can do anything to the list
<AddItemForm setItems={setItems} />

// GOOD: Owner exposes one narrow operation
const handleAddItem = (label) =>
  setItems((current) => [...current, { id: crypto.randomUUID(), label }]);
<AddItemForm onAddItem={handleAddItem} />
```

```jsx
// BAD: Two sources of truth kept in sync with an effect
const SearchBox = ({ value, onChange }) => {
  const [text, setText] = useState(value);
  useEffect(() => {
    setText(value);
  }, [value]);
  // ...
};

// GOOD (controlled): the parent owns the value
const SearchBox = ({ value, onChange }) => (
  <input value={value} onChange={(e) => onChange(e.target.value)} />
);

// GOOD (uncontrolled): the prop name says it only seeds the state
const SearchBox = ({ initialValue = '', onSearch }) => {
  const [text, setText] = useState(initialValue);
  // ...
};
```

---

## Reducers

```jsx
// BAD: Impure (random id inside) and the action carries precomputed state
function reducer(state, action) {
  switch (action.type) {
    case 'set-items':
      return { ...state, items: action.items };
    case 'add': {
      const item = { id: crypto.randomUUID(), label: action.label }; // differs on every run
      return { ...state, items: [...state.items, item] };
    }
  }
}

// GOOD: Handler creates the id; actions say what happened; reducer decides the result
dispatch({ type: 'item-added', item: { id: crypto.randomUUID(), label } });

function reducer(state, action) {
  switch (action.type) {
    case 'item-added': {
      return { ...state, items: [...state.items, action.item] };
    }
    case 'item-removed': {
      return { ...state, items: state.items.filter((item) => item.id !== action.id) };
    }
  }
}
```

---

## Context Value and Providers

```jsx
// BAD: New value object every render — every consumer re-renders, memoized or not
const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>{children}</ThemeContext.Provider>
  );
};

// GOOD: Memoized value; the context lives in its own Provider component
const ThemeContext = createContext(null);

export const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light');
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return <ThemeContext value={value}>{children}</ThemeContext>; // React 19; use .Provider before
};
```

---

## Rules of Hooks

```jsx
// BAD: Hook after an early return — hook count changes when `loading` flips
const Profile = ({ id }) => {
  const { data, loading } = useUser(id);
  if (loading) return <Loader />;
  const [tab, setTab] = useState('info'); // "Rendered more hooks than during the previous render"
  // ...
};

// GOOD: All hooks first, early returns after
const Profile = ({ id }) => {
  const { data, loading } = useUser(id);
  const [tab, setTab] = useState('info');
  if (loading) return <Loader />;
  // ...
};
```

---

## useEffect: Async, Stale Values, Cleanup

```jsx
// BAD: async callback returns a Promise, not a cleanup function
useEffect(async () => {
  const res = await fetch(`/api/users/${id}`);
  setUser(await res.json());
}, [id]);

// GOOD: inner async function; cleanup returned from the outer callback
// (a query library is still preferable to this hand-rolled shape)
useEffect(() => {
  const controller = new AbortController();
  async function runEffect() {
    try {
      const res = await fetch(`/api/users/${id}`, { signal: controller.signal });
      setUser(await res.json());
    } catch (err) {
      if (err.name !== 'AbortError') setError(err);
    }
  }
  runEffect();
  return () => controller.abort();
}, [id]);
```

```jsx
// BAD: `[]` effect captures the first render's `isOn` — the handler always sees false
useEffect(() => {
  const onKeyDown = (e) => {
    if (e.key === ' ') setIsOn(!isOn);
  };
  window.addEventListener('keydown', onKeyDown);
  return () => window.removeEventListener('keydown', onKeyDown);
}, []);

// GOOD: functional updater reads the latest state; no eslint-disable needed
useEffect(() => {
  const onKeyDown = (e) => {
    if (e.key === ' ') setIsOn((current) => !current);
  };
  window.addEventListener('keydown', onKeyDown);
  return () => window.removeEventListener('keydown', onKeyDown);
}, []);
```

```jsx
// GOOD: guard with an early return; cleanup undoes setup
useEffect(() => {
  const node = elementRef.current;
  if (!node) return;
  const observer = new IntersectionObserver(([entry]) => setIsOnscreen(entry.isIntersecting));
  observer.observe(node);
  return () => observer.disconnect();
}, [elementRef]);

// BAD: .current is still undefined while the component body runs
const isOnscreen = useIsOnscreen(elementRef.current);

// GOOD: pass the ref object; the hook reads .current inside its effect
const isOnscreen = useIsOnscreen(elementRef);
```

---

## Keys: Creation-Time IDs and Resetting State

```jsx
// BAD: length as an id — duplicates after a delete
const addTodo = (text) => setTodos([...todos, { id: todos.length, text }]);

// GOOD: id generated once, when the item is created, and stored on the item
const addTodo = (text) => setTodos([...todos, { id: crypto.randomUUID(), text }]);
{todos.map((todo) => <TodoItem key={todo.id} todo={todo} />)}
```

```jsx
// BAD: Effect to reset the form when the user changes — extra render, flashes stale values
useEffect(() => {
  setDraft(user.bio);
}, [user.id]);

// GOOD: A different key means a fresh instance with fresh state
<ProfileForm key={user.id} user={user} />;
// inside ProfileForm: const [draft, setDraft] = useState(user.bio);
```

---

## Forms

```jsx
// BAD: Click handler on the button — no Enter-to-submit, no native validation
<input value={email} onChange={(e) => setEmail(e.target.value)} />
<button onClick={() => subscribe(email)}>Subscribe</button>

// GOOD: <form> + onSubmit
const handleSubmit = (event) => {
  event.preventDefault();
  subscribe(email);
};
<form onSubmit={handleSubmit}>
  <input type="email" required value={email} onChange={(e) => setEmail(e.target.value)} />
  <button>Subscribe</button>
</form>
```

```jsx
// BAD: Hardcoded id collides when the component renders twice; undefined starts uncontrolled
const [name, setName] = useState();
<label htmlFor="name">Name</label>
<input id="name" value={name} onChange={(e) => setName(e.target.value)} />

// GOOD: useId namespaced per field; state starts as ''
const id = useId();
const nameId = `${id}-name`;
const [name, setName] = useState('');
<label htmlFor={nameId}>Name</label>
<input id={nameId} value={name} onChange={(e) => setName(e.target.value)} />
```

---

## Request Status

```jsx
// BAD: Booleans can contradict each other (isLoading && hasError)
const [isLoading, setIsLoading] = useState(false);
const [hasError, setHasError] = useState(false);

// GOOD: One status; reset fields only after confirmed success
const [status, setStatus] = useState('idle'); // 'idle' | 'loading' | 'success' | 'error'

async function handleSubmit(event) {
  event.preventDefault();
  setStatus('loading');
  try {
    const res = await fetch('/api/subscribe', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email }),
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`); // fetch doesn't reject on HTTP errors
    setStatus('success');
    setEmail('');
  } catch {
    setStatus('error');
  }
}
// <input disabled={status === 'loading'} /> and render feedback from `status`
```

---

## Prop Delegation

```jsx
// BAD: Native attributes listed by hand — no className, aria-*, onFocus, …
const Checkbox = ({ label, checked, onChange }) => (
  <label>
    <input type="checkbox" checked={checked} onChange={onChange} /> {label}
  </label>
);

// GOOD: Delegate the rest; lock what must not change; merge className
const Checkbox = ({ id, label, className, ...delegated }) => {
  const generatedId = useId();
  const appliedId = id ?? generatedId; // one id for input + label
  return (
    <div className={clsx('flex items-center gap-2', className)}>
      <input
        {...delegated} // checked, onChange, aria-*, … pass through
        id={appliedId}
        type="checkbox" // after the spread → consumers can't override it
      />
      <label htmlFor={appliedId}>{label}</label>
    </div>
  );
};
```

---

## Semantics & Polymorphism

```jsx
// BAD: div + onClick — no keyboard support, no link semantics
<div className="btn" onClick={() => navigate('/pricing')}>Pricing</div>

// GOOD: Tag follows meaning, chosen from props
const Button = ({ href, ...delegated }) => {
  const Tag = typeof href === 'string' ? 'a' : 'button';
  return <Tag href={href} {...delegated} />;
};
```

```jsx
// GOOD: Heading level from document structure, size from CSS
const Heading = ({ level = 2, className, ...delegated }) => {
  const Tag = `h${level}`;
  return <Tag className={clsx('text-xl font-semibold', className)} {...delegated} />;
};
```

---

## Slots and Variants

```jsx
// BAD: Wrapper owns a giant lookup — every usage bundles every icon
import * as icons from './icons';
const Badge = ({ iconName, children }) => {
  const Icon = icons[iconName];
  return <span><Icon size={16} />{children}</span>;
};

// GOOD: Consumer passes the component; wrapper still controls its props
const Badge = ({ icon: Icon, children }) => (
  <span className="inline-flex items-center gap-1">
    <Icon size={16} aria-hidden />
    {children}
  </span>
);
<Badge icon={Award}>Top rated</Badge>;

// GOOD: Named slot when position matters at the call site
<PageHeader title="Billing" actions={<Button>Export</Button>} />;
```

```jsx
// BAD: Call site decides styling details
<Alert color="red" borderColor="darkred" />

// GOOD: Semantic variant; the mapping lives in one place
const STATUS_CLASSES = {
  info: 'border-blue-500 bg-blue-50',
  danger: 'border-red-500 bg-red-50',
};
const Alert = ({ status = 'info', children }) => (
  <div role="alert" className={clsx('border-l-4 p-3', STATUS_CLASSES[status])}>{children}</div>
);
<Alert status="danger" />;
```

---

## Suspense Boundaries

```jsx
// BAD: Each widget loads on its own and pops in at a different time ("spinner hell")
<>
  <Stats /> {/* own spinner */}
  <Chart /> {/* own spinner */}
  <Activity /> {/* own spinner */}
</>

// GOOD: One boundary — they appear together (children suspend via async Server
// Components, `use()`, or `useSuspenseQuery`)
<Suspense fallback={<DashboardSkeleton />}>
  <Stats />
  <Chart />
  <Activity />
</Suspense>
```

---

## Visually Hidden Text

```jsx
// BAD: aria-label on a div is ignored by screen readers
<div className="size-2 rounded-full bg-green-500" aria-label="Online" />

// GOOD: Hidden text carries the meaning
<span className="size-2 rounded-full bg-green-500">
  <span className="sr-only">Online</span>
</span>
```
