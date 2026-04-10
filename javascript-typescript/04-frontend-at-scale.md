# JavaScript/TypeScript — Frontend at Scale

## Micro-Frontend Architecture

### When to Use Micro-Frontends
- Multiple teams owning different parts of the UI independently
- Different deployment cycles per section (checkout deploys 3x/day, CMS deploys weekly)
- Legacy migration: modernize one section at a time
- Large app where build times and bundle sizes are growing unmanageable

### When NOT to Use
- Small teams (< 3 frontend engineers): coordination overhead outweighs benefits
- Tightly coupled UI: if sections constantly share state or UI, micro-frontends fight you
- Greenfield with uncertain domains: premature decomposition

---

## Module Federation (Webpack 5)

This is what your `whatfix/Store` project implements. Here's the architecture:

### Container (Shell) — webpack.dev.js
```javascript
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'container',
      remotes: {
        header:    'header@http://localhost:3001/remoteEntry.js',
        products:  'products@http://localhost:3002/remoteEntry.js',
        cart:      'cart@http://localhost:3003/remoteEntry.js',
        checkout:  'checkout@http://localhost:3004/remoteEntry.js',
      },
      shared: {
        react:     { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
        'react-router-dom': { singleton: true },
      },
    }),
  ],
};
```

### Remote (Micro-App) — webpack.dev.js
```javascript
new ModuleFederationPlugin({
  name: 'cart',
  filename: 'remoteEntry.js',  // entry point consumed by container
  exposes: {
    './CartApp': './src/bootstrap',  // what this remote exposes
  },
  shared: {
    react: { singleton: true, eager: false },
    'react-dom': { singleton: true, eager: false },
  },
})
```

### Consuming a Remote in Container
```tsx
// Lazy-loaded at runtime — not bundled at build time
const CartApp = React.lazy(() => import('cart/CartApp'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/cart" element={<CartApp />} />
      </Routes>
    </Suspense>
  );
}
```

### Shared State Between Micro-Frontends
```
Challenge: Cart app needs to know about auth state owned by container.

Options:
1. Props/callbacks passed from container (tight coupling, simple)
2. Custom events on window:
   window.dispatchEvent(new CustomEvent('auth:changed', { detail: user }))
   window.addEventListener('auth:changed', handler)
3. Shared module (e.g., auth store exposed as shared federated module)
4. URL as state (path, query params) — naturally shared
5. Redux store in shared module (complex, use sparingly)

Rule: Minimize cross-MFE communication. If two apps constantly share state,
they may be one app with bad decomposition.
```

---

## Monorepo Strategies

### Nx (used in whatfix/Task/myorg)
```bash
# Generate a new app
nx generate @nx/react:app my-app

# Run only affected apps (based on git diff)
nx affected --target=test
nx affected --target=build --base=main

# Visualize dependency graph
nx graph

# Run all apps
nx run-many --target=serve --all
```

**Nx key concepts**:
- **Project graph**: Understands dependencies between apps and libs
- **Computation cache**: Never re-run unchanged builds (local + remote cache)
- **Affected**: Only build/test what changed — CI runs fast at scale
- **Generators**: Scaffolding with your organization's conventions built in
- **Executors**: Wrap build tools (webpack, jest, eslint) with unified interface

### Turborepo (alternative)
- Simpler than Nx; JS-only config; great for package-based monorepos
- Remote caching with Vercel or self-hosted
- Choose Nx for: opinionated code generation, enterprise scale, plugin ecosystem
- Choose Turborepo for: simpler setup, existing package-based structure

### Shared Libraries in Monorepo
```
apps/
  shell/
  cart/
  products/
libs/
  ui/          # shared component library
  auth/        # shared auth utilities + context
  api-client/  # typed API client (generated from OpenAPI or GraphQL)
  utils/       # date formatting, validation, etc.
  types/       # shared TypeScript types
```

---

## Web Performance (Core Web Vitals)

Google measures and ranks sites by these:

| Metric | Measures | Good | Poor |
|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Load time of largest visible element | <2.5s | >4s |
| **INP** (Interaction to Next Paint) | Responsiveness to user interaction | <200ms | >500ms |
| **CLS** (Cumulative Layout Shift) | Visual stability (elements jumping) | <0.1 | >0.25 |

### Improving LCP
```
1. Preload LCP image:
   <link rel="preload" as="image" href="/hero.webp" fetchpriority="high">

2. Server-side render or static generate the LCP element
   (don't lazy-load the hero image)

3. Optimize images:
   - WebP/AVIF formats (30-50% smaller than JPEG)
   - Correct sizing (don't serve 2000px for a 400px slot)
   - Use <img width height> to prevent layout shift
   - CDN with automatic optimization (Cloudinary, imgix, CloudFront)

4. Reduce TTFB (Time to First Byte):
   - CDN, edge caching, efficient DB queries
   - Streaming SSR (React 18): send HTML before full render completes
```

### Improving INP (formerly FID)
```
Long tasks (>50ms) block the main thread and delay input response.

1. Break long tasks with scheduler:
   await scheduler.postTask(() => heavyWork(), { priority: 'background' });
   // or: setTimeout(heavyWork, 0)

2. Use Web Workers for CPU work (JSON parsing, calculations)

3. Virtualize long lists (react-window, tanstack-virtual)
   Never render 10,000 DOM nodes

4. Debounce/throttle expensive handlers:
   const handleSearch = useDeferredValue(query); // React 18
   // or: debounce(fn, 300)
```

### Improving CLS
```
1. Set explicit dimensions on images and videos
2. Reserve space for ads and embeds (min-height)
3. Avoid inserting content above existing content
4. Use CSS transform for animations (not top/left/margin)
5. font-display: swap with size-adjusted fallback font:
   @font-face {
     font-family: 'MyFont';
     font-display: swap;
     size-adjust: 98%; /* match fallback metrics */
   }
```

---

## Bundle Optimization

### Code Splitting
```tsx
// Route-level (mandatory)
const CheckoutPage = lazy(() => import('./pages/CheckoutPage'));

// Component-level (for large, conditionally rendered components)
const RichEditor = lazy(() => import('./components/RichEditor'));
const MapComponent = lazy(() => import('./components/Map'));

// Preloading (on hover/likely navigation)
const preloadCheckout = () => import('./pages/CheckoutPage');
<button onMouseEnter={preloadCheckout}>Checkout</button>
```

### Tree Shaking Checklist
```
✓ Use ES modules (import/export), not CommonJS (require)
✓ Import specifically: import { debounce } from 'lodash-es' (not import _ from 'lodash')
✓ Mark side-effect-free in package.json: "sideEffects": false
✓ Check bundle with: webpack-bundle-analyzer, vite-bundle-visualizer
✓ Avoid barrel files (index.ts re-exporting everything) — breaks tree shaking
```

### Dependency Audit
```bash
# Find what's in your bundle
npx webpack-bundle-analyzer stats.json

# Find duplicate packages
npx duplicate-package-checker-webpack-plugin

# Check if a package has an ESM build
npx publint some-package

# Common swaps:
moment → date-fns (tree-shakeable, modular)
lodash → lodash-es (or native ES methods)
axios → native fetch (if browser-only)
```

### Next.js / Rendering Strategies
```tsx
// SSG (Static Site Generation) — pre-render at build time
export async function generateStaticParams() { ... }

// SSR (Server-Side Rendering) — render per request
export async function getServerSideProps() { ... }

// ISR (Incremental Static Regeneration) — stale-while-revalidate
export const revalidate = 60; // regenerate every 60s

// Streaming SSR (React 18 + Next.js App Router)
// Sends HTML progressively; user sees content before full render
export default async function Page() {
  return (
    <Suspense fallback={<ProductSkeleton />}>
      <Products /> {/* fetches data, doesn't block page shell */}
    </Suspense>
  );
}
```

---

## Design System Architecture

### Component API Design
```tsx
// Bad: too many one-off props
<Button primary large disabled loading onClick={...} iconLeft={<Icon />} />

// Good: composable, variant-based
<Button
  variant="primary"    // 'primary' | 'secondary' | 'ghost' | 'danger'
  size="lg"            // 'sm' | 'md' | 'lg'
  isLoading={loading}
  leftIcon={<Icon />}
  onClick={handleClick}
>
  Submit Order
</Button>
```

### Compound Components Pattern
```tsx
// Instead of: <Select label="Color" options={[...]} onChange={...} />
// Use: composable, each part independently styleable

<Select value={color} onChange={setColor}>
  <Select.Trigger>
    <Select.Value placeholder="Choose color" />
    <Select.Icon />
  </Select.Trigger>
  <Select.Content>
    <Select.Item value="red">Red</Select.Item>
    <Select.Item value="blue">Blue</Select.Item>
  </Select.Content>
</Select>
```

### Storybook + Visual Regression
```
Storybook: document + develop components in isolation
  Each component has stories for each state/variant
  
Chromatic (or Percy): auto-detect visual regressions in CI
  Screenshots every PR → compare to baseline → flag differences
  Principal concern: "how do we prevent accidental visual regressions?"
```

---

## Accessibility (Principal-Level Awareness)

```
WCAG 2.1 AA is the standard baseline for most companies (legally required in many).

Key technical concerns:
1. Semantic HTML: <button> not <div onClick>; <nav>, <main>, <article>
2. Focus management: modal opens → focus moves to modal; closes → returns to trigger
3. ARIA labels: icons need aria-label; dynamic regions need aria-live
4. Keyboard navigation: all interactive elements reachable + operable by keyboard
5. Color contrast: 4.5:1 for normal text, 3:1 for large text (WCAG AA)
6. Screen reader testing: VoiceOver (Mac), NVDA (Windows)

Tooling:
  axe-core (browser extension + jest-axe for automated tests)
  eslint-plugin-jsx-a11y (lint-time catches)
  Lighthouse (accessibility audit)

Principal angle:
  "Accessibility is a quality attribute, not a feature. 
   We build it in by default through component library enforcement."
```
