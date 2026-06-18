# VOLTA E-Commerce Catalog — Full-Stack Capstone
## Architecture, Optimization & Deployment Guide

---

## 1. Project Architecture

```
volta-catalog/
├── public/
│   ├── index.html          # Entry point
│   └── favicon.svg         # Inline SVG favicon
├── src/
│   ├── components/
│   │   ├── Nav.jsx          # Sticky navigation with cart badge
│   │   ├── Hero.jsx         # Homepage hero with featured picks
│   │   ├── ProductCard.jsx  # Reusable product card
│   │   ├── ProductDetail.jsx# Full product page
│   │   ├── CatalogPage.jsx  # Filterable grid with search
│   │   ├── CartPanel.jsx    # Slide-out cart drawer
│   │   ├── AboutPage.jsx    # Company story page
│   │   └── Footer.jsx       # Site-wide footer
│   ├── hooks/
│   │   ├── useRouter.js     # Hash-based client-side routing
│   │   └── useToast.js      # Toast notification hook
│   ├── data/
│   │   └── products.js      # Product catalogue data
│   ├── styles/
│   │   └── tokens.css       # Design token variables
│   └── App.jsx              # Root component + route resolver
├── vercel.json              # Vercel deployment config
├── netlify.toml             # Netlify deployment config
└── package.json
```

---

## 2. Core Features Implemented

### Client-Side Routing (Hash Router)
```javascript
// No library needed — pure Web API
function useRouter() {
  const [route, setRoute] = useState(() =>
    window.location.hash.replace('#','') || '/'
  );
  useEffect(() => {
    const handler = () => setRoute(window.location.hash.replace('#','') || '/');
    window.addEventListener('hashchange', handler);
    return () => window.removeEventListener('hashchange', handler);
  }, []);
  const navigate = (path) => {
    window.location.hash = path;
    window.scrollTo(0, 0);
  };
  return { route, navigate };
}
```

**Routes:**
| Route             | Component       | Description              |
|-------------------|-----------------|--------------------------|
| `#/`              | Home            | Hero + featured products |
| `#/catalog`       | CatalogPage     | Full grid with filters   |
| `#/product/:id`   | ProductDetail   | Individual product page  |
| `#/about`         | AboutPage       | Company information      |

### Modular Components
Each component is single-responsibility and receives only the props it needs:
- `ProductCard` — display only, emits `onAdd` and `onView` events
- `CartPanel` — controlled via `open` prop, emits `onClose` and `onRemove`
- `Nav` — receives `route` and `cartCount`, emits `navigate` and `onCartOpen`

### State Architecture
```
App (root state owner)
├── cart[]        → passed down to Nav (count), CartPanel, ProductCard
├── cartOpen      → controls CartPanel visibility
└── toast msg     → displayed globally via fixed overlay
```

---

## 3. Performance Optimizations

### Asset Optimization
```html
<!-- Preconnect to reduce DNS + TLS handshake time -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- font-display: swap prevents layout shift -->
<link href="...?display=swap" rel="stylesheet" />
```

### React Optimizations
```javascript
// useMemo for expensive filter/sort operations
const filtered = useMemo(() => {
  let list = PRODUCTS;
  if (category !== "All") list = list.filter(p => p.cat === category);
  if (search.trim()) list = list.filter(/* ... */);
  if (sort !== "default") list = [...list].sort(/* ... */);
  return list;
}, [category, search, sort]);   // Only recomputes when these change

// useCallback prevents child re-renders
const addToCart = useCallback((product, qty = 1) => { /* ... */ }, [showToast]);
const removeFromCart = useCallback((id) => { /* ... */ }, []);
```

### CSS Optimizations
```css
/* GPU-accelerated transitions (compositor-only properties) */
.product-card {
  transition: transform 220ms cubic-bezier(0.4,0,0.2,1),
              box-shadow 220ms cubic-bezier(0.4,0,0.2,1);
}

/* Reduced motion respect */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}

/* backdrop-filter for glass nav — GPU accelerated */
.nav { backdrop-filter: blur(14px); }
```

### Image Strategy (Production)
For a real deployment, replace emoji with optimized images:
```html
<!-- Use modern formats with fallbacks -->
<picture>
  <source srcset="product.avif" type="image/avif" />
  <source srcset="product.webp" type="image/webp" />
  <img src="product.jpg" alt="Product name" loading="lazy" width="400" height="300" />
</picture>
```

---

## 4. Deploy to Vercel (Recommended — Free)

### Step 1: Convert to Vite project
```bash
npm create vite@latest volta-catalog -- --template react
cd volta-catalog
npm install
```

### Step 2: Copy your components into `src/`

### Step 3: Install Vercel CLI
```bash
npm install -g vercel
vercel login
```

### Step 4: Deploy
```bash
vercel
# Answer prompts:
#   Project name: volta-catalog
#   Directory: ./
#   Override settings: No
```

**Your app is live at:** `https://volta-catalog.vercel.app`

### vercel.json (for React SPA routing)
```json
{
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ],
  "headers": [
    {
      "source": "/assets/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ]
}
```

---

## 5. Deploy to Netlify (Alternative — Free)

### netlify.toml
```toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

### Deploy via Netlify Drop
1. Run `npm run build` → creates `dist/` folder
2. Go to **app.netlify.com/drop**
3. Drag and drop the `dist/` folder
4. Live in 30 seconds ✓

---

## 6. Deploy to Render (Full-Stack Option)

Use Render when you add a backend (API + DB):

```yaml
# render.yaml
services:
  - type: web
    name: volta-frontend
    env: static
    buildCommand: npm run build
    staticPublishPath: dist
    routes:
      - type: rewrite
        source: /*
        destination: /index.html
```

---

## 7. Vite Build Configuration (Production)

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        // Code splitting: separate vendor chunk
        manualChunks: {
          vendor: ['react', 'react-dom'],
        }
      }
    },
    // Minification
    minify: 'terser',
    terserOptions: {
      compress: { drop_console: true }
    },
    // Asset inlining threshold (< 4kb → inline as base64)
    assetsInlineLimit: 4096,
  }
})
```

**Expected bundle sizes after build:**
| Chunk      | Size (gzipped) |
|------------|----------------|
| vendor.js  | ~42kb          |
| index.js   | ~8kb           |
| index.css  | ~4kb           |
| **Total**  | **~54kb**      |

---

## 8. Performance Checklist

- [x] `preconnect` for external fonts
- [x] `font-display: swap` to prevent FOIT
- [x] `useMemo` on filtered product lists
- [x] `useCallback` on event handlers passed to children
- [x] CSS transitions on compositor-only properties (`transform`, `opacity`)
- [x] `prefers-reduced-motion` media query respected
- [x] Semantic HTML (`<nav>`, `<main>`, `<aside>`, `<footer>`)
- [x] `aria-label` on icon-only buttons
- [x] Keyboard-navigable interactive elements
- [x] Mobile-responsive at 320px, 480px, 768px, 1200px
- [ ] **Production:** Replace emoji with `<img loading="lazy">`
- [ ] **Production:** Add `react-helmet` for per-page `<title>` and meta tags
- [ ] **Production:** Add error boundary components
- [ ] **Production:** Add Lighthouse CI to deployment pipeline

---

## 9. Extending This Project

### Add React Router (upgrade from hash routing)
```bash
npm install react-router-dom
```
```jsx
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom'
// Replace useRouter hook with <Routes> + <Route> declarations
```

### Add a Backend API
```javascript
// Example: fetch products from your own API
useEffect(() => {
  fetch('/api/products')
    .then(r => r.json())
    .then(setProducts);
}, []);
```

### Add State Management (when cart gets complex)
```bash
npm install zustand   # Lightweight, no boilerplate
```
```javascript
const useCartStore = create((set) => ({
  items: [],
  add: (product) => set(state => ({ items: [...state.items, product] })),
  remove: (id) => set(state => ({ items: state.items.filter(i => i.id !== id) }))
}))
```

---

## 10. Lighthouse Target Scores

| Metric          | Target | Strategy                              |
|-----------------|--------|---------------------------------------|
| Performance     | 95+    | Code splitting, lazy images, tiny CSS |
| Accessibility   | 95+    | Semantic HTML, ARIA, color contrast   |
| Best Practices  | 100    | HTTPS, no deprecated APIs             |
| SEO             | 90+    | Meta tags, structured data            |
