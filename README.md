# claude-frontend-plugin

Plugin para projetos frontend com **React 19 + TanStack Router + TanStack Query + Kubb + Tailwind CSS + shadcn/ui + Zustand + BetterAuth**.

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

## O que muda na v1.2.0

- **Nova skill `organization-frontend.md`** — multi-tenant no front: `_app/$orgSlug/` routing, hooks `useOrganizations`/`useActiveOrganization`, `OrganizationSwitcher`, redirect `/no-organization`, gate de 2FA setup, página `setup-2fa.tsx` com 3 etapas.
- **Skill `component-patterns.md`** ganhou seção **Theme dark/light** (ThemeProvider customizado, `localStorage` `'ui-theme'`, `ThemeToggle`).
- **`auth-client.md`**: `beforeLoad` agora usa `authClient.getSession()` (em vez de `fetch` direto).
- **Padronização `bun run`**: `kubb-config`, `setup-kubb` e `new-feature` agora usam `bun run generate` em vez de `npx`/`npm run api:generate`.
- **Porta default 8080** em todos os exemplos (alinha com `claude-backend-plugin` v1.2.0).
- **`frontend-reviewer.md`** ganhou checks de multi-tenant: rota fora de `_app/$orgSlug/`, falta de `useParams({ from: '/_app/$orgSlug' })`, mutation de org sem invalidação de session, gate de 2FA ausente.
- **`new-feature` command** suporta variante org-scoped (`_app/$orgSlug/{feature}/`).
- `architecture.md` documenta a estrutura completa de pastas multi-tenant + setup-2fa standalone.

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

# Forms
bun add react-hook-form @hookform/resolvers zod

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
      "!**/src/route-tree.gen.ts",
      "!**/src/gen",
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
    path: "http://localhost:8080/openapi/json",
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

### 7. Configurar TanStack Query provider com `MutationCache` central

Criar `src/integrations/tanstack-query/root-provider.tsx`:

```typescript
import { MutationCache, QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { Toaster, toast } from 'sonner'
import type { ReactNode } from 'react'

export const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onSuccess: (_data, _variables, _context, mutation) => {
      const { invalidates, successMessage } = mutation.meta ?? {}
      if (invalidates?.length) {
        for (const queryKey of invalidates) {
          queryClient.invalidateQueries({ queryKey })
        }
      }
      if (successMessage) toast.success(successMessage)
    },
    onError: (error, _variables, _context, mutation) => {
      const { errorMessage } = mutation.meta ?? {}
      if (errorMessage === false) return
      toast.error(typeof errorMessage === 'string' ? errorMessage : (error as Error).message)
    },
  }),
})

export function Provider({ children }: { children: ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <Toaster />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

E `src/integrations/tanstack-query/mutation-meta.d.ts`:

```typescript
import '@tanstack/react-query'

declare module '@tanstack/react-query' {
  interface Register {
    mutationMeta: {
      invalidates?: readonly unknown[][]
      successMessage?: string
      errorMessage?: string | false
    }
  }
}
```

### 8. Adicionar variáveis de ambiente e scripts

**.env**

```env
VITE_API_URL=http://localhost:8080
VITE_APP_URL=http://localhost:3000
```

No `package.json`, adicionar:

```json
{
  "scripts": {
    "generate": "kubb generate",
    "lint": "bunx biome check .",
    "lint:fix": "bunx biome check --write ."
  }
}
```

Adicionar ao `.gitignore`:

```gitignore
src/gen/
```

### 9. Criar estrutura de pastas

```bash
mkdir -p \
  src/components/ui \
  src/components/layout \
  src/components/theme \
  src/hooks \
  src/stores \
  src/integrations/tanstack-query \
  src/integrations/better-auth \
  src/lib
```

### 10. Iniciar

```bash
bun run dev
```

Quando o backend estiver rodando em `VITE_API_URL`, gerar os hooks:

```bash
bun run generate
```

---

## O que o plugin inclui

### Skills

| Skill                   | Ativada quando                                                          |
| ----------------------- | ----------------------------------------------------------------------- |
| `architecture`          | Decisões de estrutura, organização de pastas, fluxo de dados            |
| `kubb-config`           | Configurar Kubb, regerar código, entender output                        |
| `tanstack-router`       | File-based routing, layouts, pathless groups                            |
| `tanstack-query`        | MutationCache central com `meta.invalidates`/`successMessage`           |
| `api-client`            | Axios config, interceptors, cookies                                     |
| `auth-client`           | BetterAuth client no React, `useSession`, `beforeLoad`                  |
| `zustand-patterns`      | Stores, selectors, Immer middleware                                     |
| `component-patterns`    | Organização de componentes, props, shadcn/ui, **Theme dark/light**      |
| `form-patterns`         | Zod + RHF + `setError('root')` + `useId()`                              |
| `organization-frontend` | Multi-tenant: `_app/$orgSlug/`, switcher, 2FA setup gate, `setup-2fa.tsx` |

### Commands

| Command                 | Uso                                                          |
| ----------------------- | ------------------------------------------------------------ |
| `/setup-kubb`           | Configurar Kubb do zero (porta 8080, `bun run generate`)     |
| `/new-feature products` | Criar página + componente principal (single-tenant ou org-scoped) |

### Agents

| Agent               | Função                                                              |
| ------------------- | ------------------------------------------------------------------- |
| `frontend-reviewer` | Review de componentes, state, routing, auth, mutations, multi-tenant |
