# claude-frontend-plugin

Plugin para projetos frontend com **React 19 + TanStack Router + TanStack Query + Kubb + Tailwind CSS + shadcn/ui + Zustand**.

## Instalação

### 1. Adicionar o marketplace (apenas na primeira vez)

```bash
claude plugin marketplace add NovaisVictor/claude-marketplace
```

### 2. Instalar o plugin

```bash
claude plugin install claude-frontend-plugin@novais-plugins
```

---

## Setup de projeto do zero

### 1. Criar projeto com TanStack Router

```bash
bunx @tanstack/cli create --router-only
```

O CLI vai perguntar:

- **Route configuration**: File-based (recomendado)
- **TypeScript**: Yes
- **Tailwind CSS**: Yes
- **Toolchain**: Vite

Após completar:

```bash
cd my-app
bun install
```

### 2. Instalar dependências adicionais

```bash
# TanStack Query
bun add @tanstack/react-query @tanstack/react-query-devtools

# HTTP client
bun add axios

# State management
bun add zustand immer

# shadcn/ui (seguir setup do shadcn para Vite + Tailwind)
bunx shadcn@latest init

# Auth client (se usar BetterAuth no backend)
bun add @better-auth/react

# Kubb (geração de código)
bun add -D @kubb/core @kubb/plugin-oas @kubb/plugin-ts @kubb/plugin-react-query @kubb/plugin-zod @kubb/plugin-client

# Biome (substituir ESLint/Prettier se o scaffold trouxe)
bun add -D @biomejs/biome
```

### 3. Configurar Biome

Criar `biome.json` na raiz:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.2.4/schema.json",
  "files": {
    "includes": [
      "**/src/**/*",
      "!**/src/routeTree.gen.ts",
      "!**/src/gen/**",
      "!**/src/styles.css"
    ]
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 80,
    "lineEnding": "lf"
  },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  },
  "javascript": {
    "formatter": { "quoteStyle": "single", "semicolons": "asNeeded" }
  },
  "assist": { "actions": { "source": { "organizeImports": "on" } } }
}
```

### 4. Configurar Kubb

Criar `kubb.config.ts` na raiz:

```typescript
import { defineConfig } from "@kubb/core";
import { pluginOas } from "@kubb/plugin-oas";
import { pluginTs } from "@kubb/plugin-ts";
import { pluginReactQuery } from "@kubb/plugin-react-query";
import { pluginZod } from "@kubb/plugin-zod";

function toKebabCase(str: string): string {
  return str
    .replace(/([a-z])([A-Z])/g, "$1-$2")
    .replace(/[\s_]+/g, "-")
    .toLowerCase();
}

export default defineConfig({
  root: ".",
  input: {
    path: "http://localhost:3333/openapi/json",
  },
  output: {
    path: "./src/gen",
    clean: true,
    format: "biome",
    lint: "biome",
  },
  plugins: [
    pluginOas({ validate: false }),
    pluginTs({
      output: { path: "./models", barrelType: "named" },
    }),
    pluginZod({
      output: { path: "./zod", barrelType: "named" },
      typed: true,
      inferred: true,
    }),
    pluginReactQuery({
      output: { path: "./hooks", barrelType: "named" },
      client: { dataReturnType: "data" },
      query: {
        methods: ["get"],
        importPath: "@tanstack/react-query",
      },
      mutation: {
        methods: ["post", "put", "delete", "patch"],
      },
      suspense: {},
      exclude: [{ type: "tag", pattern: "Better Auth" }],
      group: {
        type: "tag",
        name: ({ group }) => `${toKebabCase(group)}-hooks`,
      },
    }),
  ],
});
```

### 5. Configurar Axios client

Criar `src/lib/api-client.ts`:

```typescript
import axios from "axios";

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  withCredentials: true,
});

apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      window.location.href = "/sign-in";
    }
    return Promise.reject(error);
  },
);
```

### 6. Configurar BetterAuth client (se usar auth)

Criar `src/integrations/better-auth/auth-client.ts`:

```typescript
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  baseURL: import.meta.env.VITE_API_URL,
});
```

### 7. Configurar TanStack Query provider

Criar `src/integrations/tanstack-query/root-provider.tsx`:

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import type { ReactNode } from 'react'

const queryClient = new QueryClient()

export function getContext() {
  return { queryClient }
}

export function Provider({
  queryClient,
  children,
}: {
  queryClient: QueryClient
  children: ReactNode
}) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

### 8. Ajustar main.tsx

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

### 9. Adicionar variáveis de ambiente e scripts

**.env**

```env
VITE_API_URL=http://localhost:3333
```

No `package.json`, adicionar:

```json
{
  "scripts": {
    "api:generate": "kubb generate",
    "api:watch": "kubb generate --watch",
    "lint": "bunx biome check .",
    "lint:fix": "bunx biome check --write ."
  }
}
```

Adicionar ao `.gitignore`:

```gitignore
src/gen/
```

### 10. Criar estrutura de pastas

```bash
mkdir -p src/components/ui src/components/layout src/components/theme src/hooks src/stores
```

### 11. Iniciar

```bash
bun run dev
```

Quando o backend estiver rodando, gerar os hooks:

```bash
bun run api:generate
```

---

## O que o plugin inclui

### Skills

| Skill                | Ativada quando                                               |
| -------------------- | ------------------------------------------------------------ |
| `architecture`       | Decisões de estrutura, organização de pastas, fluxo de dados |
| `kubb-config`        | Configurar Kubb, regerar código, entender output             |
| `tanstack-router`    | File-based routing, layouts, pathless groups                 |
| `tanstack-query`     | Provider, hooks gerados, cache, invalidação                  |
| `api-client`         | Axios config, interceptors, cookies                          |
| `auth-client`        | BetterAuth client no React, session, proteção de rotas       |
| `zustand-patterns`   | Stores, selectors, Immer middleware                          |
| `component-patterns` | Organização de componentes, props, shadcn/ui                 |

### Commands

| Command                 | Uso                                                     |
| ----------------------- | ------------------------------------------------------- |
| `/setup-kubb`           | Configurar Kubb do zero                                 |
| `/new-feature products` | Criar página + componente principal integrado com hooks |

### Agents

| Agent               | Função                                                  |
| ------------------- | ------------------------------------------------------- |
| `frontend-reviewer` | Review de componentes, state, routing, auth (read-only) |
