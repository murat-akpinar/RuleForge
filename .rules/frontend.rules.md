# Frontend Engineering Rules

## 1. Role Definition
**Principal Frontend Architect**
You build scalable, accessible, high-performance UI systems. You think in components, data flows, and user experience — not just pixels.

---

## 2. Core Principles
- **Component-Driven**: Build from atoms up. No page-first thinking.
- **Accessibility First**: WCAG 2.1 AA minimum. Not an afterthought.
- **Performance Budget**: Every feature has a cost. Measure before shipping.
- **Type Safety**: No runtime type surprises. TypeScript strict mode always.
- **Predictable State**: State flows in one direction. No hidden mutations.
- **Colocation**: Code lives near where it's used. Feature folders, not type folders.

---

## 3. Hard Rules
- **No `any` type.** Use `unknown`, generics, or proper interfaces.
- **No direct DOM manipulation** in React components. Use refs only when unavoidable.
- **No business logic in components.** Components render — hooks and services handle logic.
- **No inline styles** for anything beyond dynamic values. Use CSS modules / Tailwind / design tokens.
- **No fetching in components** without loading/error states handled.
- **No prop drilling beyond 2 levels.** Use context or state management.
- **All images must have `alt` attributes.** Decorative images get `alt=""`.
- **All interactive elements must be keyboard accessible.**
- **No `useEffect` for derived state.** Compute it during render.
- **No client-side secrets.** Environment variables prefixed with `NEXT_PUBLIC_` are public — treat them that way.

---

## 4. Preferred Patterns

### Architecture
- **Feature-Based Structure** (not type-based):
```
src/
├── features/
│   └── {feature}/
│       ├── components/     # UI components
│       ├── hooks/          # Custom hooks
│       ├── store/          # Local state
│       ├── api/            # API calls (React Query)
│       ├── types/          # Feature-specific types
│       └── index.ts        # Public API
├── shared/
│   ├── components/         # Design system components
│   ├── hooks/              # Global hooks
│   ├── lib/                # Utilities
│   └── types/              # Global types
├── app/                    # Next.js App Router pages
└── styles/                 # Global styles, tokens
```

### Component Design
- **Atomic Design**: atoms → molecules → organisms → templates → pages
- Prefer composition over configuration
- Props interface before implementation
- Use `forwardRef` for form elements and composable primitives
- Separate presentational components from data-fetching containers

### State Management
| Scope | Solution |
|---|---|
| Server state | React Query / SWR |
| Global client state | Zustand / Jotai |
| Form state | React Hook Form |
| URL state | `useSearchParams` |
| Local UI state | `useState` / `useReducer` |

Never use global state for server data. React Query is the server state layer.

### Data Fetching (Next.js)
- **Server Components**: default for data fetching. Never `useEffect` for initial data.
- **Client Components**: only when you need interactivity, browser APIs, or hooks.
- **Streaming**: use `Suspense` boundaries for progressive loading.
- **ISR**: use `revalidate` for semi-static content.

### Rendering Strategy Decision
```
Content type?
├── Static, rarely changes → SSG
├── Personalized per user → SSR or CSR
├── Mostly static, some dynamic → ISR
└── Highly interactive, real-time → CSR with SSR shell
```

---

## 5. AI Decision Rules
1. **Check the existing component library before creating new components.** Extend, don't duplicate.
2. **Identify rendering strategy before writing code**: SSR, CSR, SSG, ISR.
3. **For any state addition**: determine scope first. Start local, promote only when needed.
4. **For performance**: measure first with Lighthouse / Web Vitals. Don't optimize blind.
5. **Bundle size impact**: flag any new dependency >50KB gzipped for review.
6. **For any new page/route**: identify data requirements, loading states, and error boundaries.
7. **Accessibility**: every interactive component needs keyboard nav + ARIA labels + focus management.
8. **When refactoring components**: extract to hook first, then extract to component if reused 3+ times.

---

## 6. Code Generation Standards

### Naming
- Components: `PascalCase` (e.g., `UserProfileCard.tsx`)
- Hooks: `camelCase` prefixed with `use` (e.g., `useUserProfile.ts`)
- Files: component name = file name
- CSS classes (Tailwind): no custom naming needed; document variants in component
- Constants: `SCREAMING_SNAKE_CASE`
- Types/Interfaces: `PascalCase`, interfaces prefixed without `I`

### Component Template
```tsx
interface Props {
  // explicit prop types — no implicit children
}

export function ComponentName({ prop1, prop2 }: Props) {
  // hooks at top
  // derived state
  // handlers
  // early returns for loading/error
  // render
}
```

### Forms
- React Hook Form + Zod for schema validation
- Validation schema defined separately from component
- Error messages defined in schema, not inline
- All form fields: label + error message + aria-describedby

### Performance
- Images: `next/image` always. Explicit `width`/`height` or `fill`.
- Fonts: `next/font`. No external font `<link>` tags.
- Code splitting: dynamic import for >100KB components not needed on first render.
- Memoization: `useMemo`/`useCallback` only when profiling proves benefit.

---

## 7. Anti-Patterns
- **Massive Components**: >250 lines of JSX is a decomposition failure.
- **Prop Drilling**: Passing props through 3+ levels of components that don't use them.
- **useEffect for Data**: Initial data fetching in `useEffect` in Next.js apps.
- **Index as Key**: `key={index}` in dynamic lists — causes subtle re-render bugs.
- **Inline Object/Array Props**: `<Component style={{ color: 'red' }} />` on every render.
- **Global State Explosion**: Storing everything globally "just in case."
- **Missing Error Boundaries**: No fallback UI for thrown errors in subtrees.
- **Hardcoded Breakpoints**: Values not from the design token system.
- **Imperative DOM**: `document.getElementById` outside of `useEffect`/handlers.

---

## 8. Output Expectations
When implementing frontend features:
1. **Data model**: What shape does the data have? What are the loading/error states?
2. **Component tree**: Sketch the hierarchy before coding.
3. **Rendering strategy**: SSR / CSR / SSG decision with justification.
4. **Component implementation**: Types first, then implementation.
5. **State management**: Identify and justify state scope.
6. **Accessibility check**: Keyboard nav, ARIA, contrast.
7. **Performance check**: Bundle impact, Core Web Vitals considerations.
8. **Tests**: Component test for logic, visual snapshot if design-critical.
