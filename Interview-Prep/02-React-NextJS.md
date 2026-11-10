# Section 2: React + Next.js (12 Qs)

> [← Back to index](README.md)


### 1. Server Components vs Client Components
- **Server Components (default in App Router):** run only on the server, ship **zero JS** to the browser, can query the DB/secrets directly with `async/await`. Use them for data fetching, layouts, static content, SEO pages.
- **Client Components (`'use client'`):** needed for **state, effects, event handlers, browser APIs** (useState, onClick, localStorage, sockets).
- **Pattern:** keep pages as server components and push `'use client'` down to small interactive "leaves" (a button, form or chat box). Pass server-fetched data as props.
```tsx
// app/products/page.tsx (server)
export default async function Page() {
  const products = await prisma.product.findMany();
  return <ProductList products={products} />;   // ProductList can be a client component
}
```

### 2. SSR vs SSG vs ISR vs CSR
| | When HTML is built | Use for | Next.js App Router |
|---|---|---|---|
| **SSG** | at build time | Landing, blog, docs | default for static fetches |
| **ISR** | build + regenerate every N sec or on demand | Product pages, catalog | `export const revalidate = 60` or `fetch(url, { next: { revalidate: 60 } })` |
| **SSR** | on every request | User-specific dashboard, order page | `cache: 'no-store'`, `cookies()`, `dynamic = 'force-dynamic'` |
| **CSR** | in the browser after JS loads | Highly interactive parts behind login (chat, live tracking) | client component + SWR/React Query |

**[My project] answer:** "Marketing pages are SSG, product pages are ISR with revalidation when an admin updates a product, the order/checkout page is SSR because it's user-specific, and the realtime order status is CSR over sockets."

### 3. useEffect cleanup — why needed?
Cleanup runs **before the effect re-runs and when the component unmounts**. Without it you get memory leaks, duplicate listeners, and "setState on unmounted component" bugs.
```tsx
useEffect(() => {
  socket.emit('join', orderId);
  const onStatus = (s) => setStatus(s);
  socket.on('order:status', onStatus);
  return () => {                     // cleanup
    socket.off('order:status', onStatus);
    socket.emit('leave', orderId);
  };
}, [orderId]);
```
Other examples: `clearInterval`, `AbortController.abort()` for fetch, `removeEventListener('resize')`. In dev, React StrictMode runs effects twice on purpose so that missing cleanups show up.

### 4. useMemo vs useCallback vs React.memo
- **useMemo:** caches a **computed value**. `useMemo(() => expensiveFilter(list), [list])`
- **useCallback:** caches a **function reference**. `useCallback(() => onSelect(id), [id])`
- **React.memo:** caches a **component**, which re-renders only if its props change (shallow compare).
- They work **together**: `React.memo(Child)` is useless if the parent passes a new function every render, so wrap that function in `useCallback`.
- **Don't overuse them.** Memoization has a cost. Use it for expensive calculations, large lists, or props passed to memoized children. (React Compiler can now do much of this automatically.)

### 5. App Router vs Pages Router
| Pages Router (`pages/`) | App Router (`app/`) |
|---|---|
| `getServerSideProps`, `getStaticProps` | `async` server components + `fetch` cache options |
| All components are client components | Server Components by default |
| `_app.tsx`, `_document.tsx` | Nested `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx` |
| API routes in `pages/api` | Route handlers `app/api/x/route.ts` + **Server Actions** |
| No built-in streaming | Streaming with Suspense |

"I use App Router for new projects because of nested layouts, streaming, server actions and less client JS."

### 6. Next.js caching + revalidatePath
Next.js has several cache layers:
1. **Request memoization:** the same `fetch` in one render runs only once.
2. **Data Cache:** `fetch` results stored on the server (`force-cache`, `next: { revalidate, tags }`).
3. **Full Route Cache:** pre-rendered HTML/RSC payload for static routes.
4. **Router Cache (client):** pages already visited are kept in browser memory.

*(Note: Next 15+ changed `fetch` to **not cached by default**, so you opt in with `cache: 'force-cache'` or `'use cache'`. Mention that you know about this change.)*

**On-demand revalidation:**
```ts
'use server';
export async function updateProduct(id: string, data: FormData) {
  await prisma.product.update({ where: { id }, data: { /* ... */ } });
  revalidatePath(`/products/${id}`);   // rebuild this page
  revalidateTag('products');           // or: everything fetched with tags: ['products']
}
```

### 7. Auth in Next.js + Middleware
- **Options:** NextAuth/Auth.js, Clerk, or custom JWT in an **httpOnly, Secure, SameSite cookie**. Never store tokens in localStorage, because XSS can steal them.
- **Middleware (`middleware.ts`)** runs on the Edge **before** the route. It's used for quick checks: is the token present and valid? If not, redirect to `/login`.
```ts
export function middleware(req: NextRequest) {
  const token = req.cookies.get('access_token')?.value;
  if (!token && req.nextUrl.pathname.startsWith('/dashboard'))
    return NextResponse.redirect(new URL('/login', req.url));
  return NextResponse.next();
}
export const config = { matcher: ['/dashboard/:path*', '/orders/:path*'] };
```
- **Important:** middleware is not your only security layer. **Check the session again inside server actions / route handlers / the DB layer** (role checks, "does this order belong to this user?").

### 8. Hydration error — cause and fix
**Hydration** = React attaching event handlers to server-rendered HTML. The error happens when **server HTML ≠ first client render**.
**Common causes:** `Date.now()`/`Math.random()`, `window`/`localStorage` used during render, timezone/locale formatting, invalid HTML (`<div>` inside `<p>`), browser extensions changing the DOM.
**Fixes:**
- Move browser-only logic into `useEffect` (render a placeholder first).
- `dynamic(() => import('./Chart'), { ssr: false })`.
- Format dates with a fixed timezone, or on the client only.
- `suppressHydrationWarning` only for small, unavoidable cases (like a timestamp).

### 9. Image / performance optimization in Next.js
- `next/image`: automatic WebP/AVIF, lazy loading, responsive `sizes`, and no layout shift (requires width/height). Use `priority` for the LCP hero image.
- `next/font`: self-hosted fonts with no layout shift.
- Server Components mean less client JS. Use `dynamic()` imports for heavy components (charts, editors).
- Streaming with `loading.tsx` and `<Suspense>`.
- Caching/ISR, a CDN, and `next/script` with `strategy="lazyOnload"` for 3rd-party scripts.
- Measure with Lighthouse, Core Web Vitals (LCP, INP, CLS) and `@next/bundle-analyzer`.

### 10. Context API vs Zustand vs Redux
| | Best for | Downside |
|---|---|---|
| **Context** | Rarely-changing global values: theme, auth user, locale | Every consumer re-renders on any change |
| **Zustand** | Small/medium client state: cart, UI state, chat state | Less structure for huge teams |
| **Redux Toolkit** | Large apps, complex flows, devtools/time-travel, big teams | More boilerplate |
| **React Query / SWR** | **Server state** (API data, caching, refetch) | — |

**Answer:** "I keep server data in React Query (or in server components), auth/theme in Context, and the cart plus realtime UI state in Zustand, because it's tiny, has selectors (fewer re-renders) and no providers. I'd pick Redux Toolkit for a large team with complex state."

### 11. Forms + validation in Next.js
**React Hook Form + Zod** on the client, and the **same Zod schema re-validated on the server** (never trust the client).
```ts
// schema.ts (shared)
export const checkoutSchema = z.object({
  name: z.string().min(2),
  phone: z.string().regex(/^[6-9]\d{9}$/),
  pincode: z.string().length(6),
});
// client
const form = useForm({ resolver: zodResolver(checkoutSchema) });
// server action
'use server';
export async function placeOrder(data: unknown) {
  const parsed = checkoutSchema.safeParse(data);
  if (!parsed.success) return { errors: parsed.error.flatten().fieldErrors };
  // ...
}
```
With Server Actions you can also use `useActionState` (`useFormState` in older versions) to show server errors, and `useFormStatus` for the pending state.

### 12. Error Boundaries
- A class component with `componentDidCatch` / `getDerivedStateFromError` that catches **render errors in its children** and shows a fallback UI instead of a white screen. (`react-error-boundary` gives a hook-friendly version.)
- It does **not** catch errors in event handlers, async code, or SSR. Use try/catch there.
- **Next.js App Router:** `error.tsx` in a route folder is an automatic error boundary for that segment (it gets `error` and `reset`). `global-error.tsx` covers the root layout.
- Log the errors to Sentry inside the boundary.

