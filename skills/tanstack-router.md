---
description: "Convenções do TanStack Router: file-based routing, layouts, pathless groups, root context. Use quando trabalhar com rotas, páginas, navegação, layouts, ou qualquer código em src/pages/."
---

# TanStack Router — Convenções

## File-based routing

Rotas ficam em `src/pages/`. O arquivo `route-tree.gen.ts` é auto-gerado — NUNCA editar.

## Root route

`src/pages/__root.tsx` — injeta QueryClient no contexto do router:

```typescript
import type { QueryClient } from '@tanstack/react-query'
import {
  createRootRouteWithContext,
  HeadContent,
  Outlet,
} from '@tanstack/react-router'
import { ThemeProvider } from '@/components/theme/theme-provider'

interface MyRouterContext {
  queryClient: QueryClient
}

export const Route = createRootRouteWithContext<MyRouterContext>()({
  component: () => (
    <ThemeProvider defaultTheme="dark" storageKey="vite-ui-theme">
      <HeadContent />
      <Outlet />
    </ThemeProvider>
  ),
})
```

## Layout groups (pathless)

Prefixo `_` cria layout groups que não geram segmento na URL:

```
src/pages/
  _app/                 ← Layout dashboard (não gera /app na URL)
    layout.tsx          ← Header, sidebar, wrapper
    index.tsx           ← Página principal (/)
    settings/
      index.tsx         ← /settings
  _auth/                ← Layout auth (não gera /auth na URL)
    sign-in.tsx         ← /sign-in
```

### Layout de grupo

```typescript
// src/pages/_app/layout.tsx
import { createFileRoute, Outlet } from '@tanstack/react-router'
import Header from '@/components/layout/header'

export const Route = createFileRoute('/_app')({
  component: DashboardLayout,
})

function DashboardLayout() {
  return (
    <>
      <Header />
      <div className="flex mt-10 px-4 justify-center max-w-360 mx-auto">
        <Outlet />
      </div>
    </>
  )
}
```

### Página dentro do grupo

```typescript
// src/pages/_app/index.tsx
import { createFileRoute } from '@tanstack/react-router'
import { BannersTab } from '@/components/banners/banners-tabs'

export const Route = createFileRoute('/_app/')({
  component: App,
})

function App() {
  return (
    <div className="w-full">
      <h1 className="text-2xl font-bold mb-4 md:mb-6">Banners</h1>
      <BannersTab />
    </div>
  )
}
```

## Main entry (src/main.tsx)

```typescript
import { createRouter, RouterProvider } from '@tanstack/react-router'
import { StrictMode } from 'react'
import ReactDOM from 'react-dom/client'
import * as TanStackQueryProvider from './integrations/tanstack-query/root-provider.tsx'
import { routeTree } from './route-tree.gen.ts'
import './styles.css'

const TanStackQueryProviderContext = TanStackQueryProvider.getContext()

const router = createRouter({
  routeTree,
  context: { ...TanStackQueryProviderContext },
  defaultPreload: 'intent',
  scrollRestoration: true,
  defaultStructuralSharing: true,
  defaultPreloadStaleTime: 0,
})

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}

const rootElement = document.getElementById('app')
if (rootElement && !rootElement.innerHTML) {
  const root = ReactDOM.createRoot(rootElement)
  root.render(
    <StrictMode>
      <TanStackQueryProvider.Provider {...TanStackQueryProviderContext}>
        <RouterProvider router={router} />
      </TanStackQueryProvider.Provider>
    </StrictMode>,
  )
}
```

## Regras

- Páginas são thin — só compõem componentes, sem lógica de negócio
- Cada page exporta `Route` via `createFileRoute` com path correspondente
- Componente da page é uma função separada, não inline no `createFileRoute`
- `route-tree.gen.ts` é auto-gerado, nunca editar
- QueryClient no contexto do router permite prefetch em loaders
- `defaultPreload: 'intent'` — prefetch ao hover em links
- `scrollRestoration: true` — restaura scroll ao navegar
