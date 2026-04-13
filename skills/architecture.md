---
description: "Arquitetura frontend com React 19, TanStack Router, TanStack Query, Kubb, Tailwind CSS, shadcn/ui. Use quando precisar tomar decisões de estrutura, organização de pastas, ou entender o fluxo de dados do frontend."
---

# Arquitetura Frontend

## Stack

| Camada | Tecnologia |
|--------|-----------|
| Framework | React 19 |
| Routing | TanStack Router (file-based) |
| Server state | TanStack React Query |
| Client state | Zustand (+ Immer para estados complexos) |
| API codegen | Kubb (types, hooks, zod, clients) |
| HTTP client | Axios |
| Validação | Zod |
| UI | Tailwind CSS 4 + Radix UI (shadcn/ui) |
| Linter/Formatter | Biome |
| Auth | BetterAuth client (@better-auth/react) |

## Estrutura de pastas

```
src/
  gen/                              ← Gerado pelo Kubb (NUNCA editar)
    models/                         ← Types do OpenAPI
    hooks/                          ← React Query hooks
    zod/                            ← Zod schemas
    clients/                        ← Axios client functions
  integrations/
    tanstack-query/
      root-provider.tsx             ← QueryClient config + Provider
      devtools.tsx
    better-auth/
      auth-client.ts                ← BetterAuth client config
  lib/
    utils.ts                        ← cn(), formatters, helpers
    api-client.ts                   ← Axios instance (baseURL, interceptors, cookies)
  components/
    ui/                             ← shadcn primitivos (nunca lógica de negócio)
    layout/                         ← header, tabs, logo, sidebar
    theme/                          ← theme provider, toggle
    {feature}/                      ← componentes da feature (só componentes)
  hooks/                            ← hooks customizados
  stores/                           ← Zustand stores
  pages/                            ← TanStack Router file-based routes
    __root.tsx                      ← Root layout
    _app/                           ← Dashboard layout (auth required)
    _auth/                          ← Auth layout (público)
```

## Regras de organização

### components/

- Contém APENAS componentes. Nenhum hook, nenhum arquivo de types, nenhuma constante solta.
- Props interface fica dentro do próprio componente que a usa.
- Pasta de feature (`banners/`) só se justifica com 3+ componentes. Componente isolado fica solto.
- `ui/` é exclusivamente shadcn — nunca colocar lógica de negócio ali.

### hooks/

- Hooks customizados ficam aqui, separados dos componentes.
- Hooks genéricos: `use-debounce.ts`, `use-media-query.ts`
- Hooks de feature: `use-banner-filters.ts`

### stores/

- Zustand stores para client state (UI state, filtros, sidebar, wizard steps).
- Server state (dados da API) fica no React Query, nunca no Zustand.
- Usar Immer middleware apenas quando o state tem estruturas aninhadas.

### pages/

- Páginas são thin — só compõem componentes, sem lógica.
- Prefixo `_` para pathless layout groups (convenção TanStack Router).
- `route-tree.gen.ts` é auto-gerado, nunca editar.

### gen/

- Gerado pelo Kubb a partir do OpenAPI spec do backend.
- NUNCA editar manualmente. Regerar com `npx kubb generate`.
- Types de API vêm de `gen/models/`, hooks de `gen/hooks/`.

### integrations/

- Configuração de setup de bibliotecas externas (providers, clients).
- Não é utilitário — é wiring de infraestrutura.

### lib/

- Utilitários genéricos do projeto.
- `api-client.ts` é a instância Axios configurada que o Kubb consome.

## Fluxo de dados

### Server state (API)
```
Componente → useGetProducts() (gen/hooks) → Axios client (lib/api-client) → Backend
```

### Auth
```
Componente → authClient.signIn.email() (integrations/better-auth) → /auth/*
```

### Client state
```
Componente → useSidebarStore() (stores/) → Zustand → Re-render
```

## Separação de concerns

| Tipo de dado | Onde vive | Gerenciado por |
|---|---|---|
| Dados do servidor | React Query cache | TanStack Query (via Kubb hooks) |
| Estado de UI | Zustand stores | Zustand |
| Estado de form | Componente local (useState) | React |
| Session/auth | BetterAuth | @better-auth/react |
| Dados derivados | Computados no componente | useMemo / selectors |
