# plugin-frontend

Plugin para projetos frontend com **React 19 + TanStack Router + TanStack Query + Kubb + Tailwind CSS + shadcn/ui + Zustand**.

## Instalação

```bash
claude plugin install github:NovaisVictor/claude-frontend-plugin
```

## O que inclui

### Skills

| Skill                | Ativada quando                                                  |
| -------------------- | --------------------------------------------------------------- |
| `architecture`       | Decisões de estrutura, organização de pastas, fluxo de dados    |
| `kubb-config`        | Configurar Kubb, regerar código, entender output gerado         |
| `tanstack-router`    | File-based routing, layouts, pathless groups, root context      |
| `tanstack-query`     | Provider, hooks gerados, cache, invalidação, mutations          |
| `api-client`         | Axios config, interceptors, cookies, integração com Kubb        |
| `auth-client`        | BetterAuth client no React, session, proteção de rotas          |
| `zustand-patterns`   | Stores, selectors, Immer middleware, quando usar vs React Query |
| `component-patterns` | Organização de componentes, props, shadcn/ui, Tailwind          |

### Commands

| Command                 | Uso                        | O que faz                                                 |
| ----------------------- | -------------------------- | --------------------------------------------------------- |
| `/setup-kubb`           | Configurar Kubb do zero    | Deps, kubb.config.ts, api-client, scripts, .gitignore     |
| `/new-feature products` | Criar estrutura de feature | Página + componente principal integrado com hooks do Kubb |

### Agents

| Agent               | Modo             | Função                                                    |
| ------------------- | ---------------- | --------------------------------------------------------- |
| `frontend-reviewer` | plan (read-only) | Review de componentes, state, routing, auth, API patterns |

## Fluxo de dados

```
Dados do servidor:  Componente → useGetX() (Kubb) → Axios → Backend
Auth:               Componente → authClient (BetterAuth) → /auth/*
Estado de UI:       Componente → useXStore() (Zustand) → Re-render
```

## Separação de state

| Tipo              | Onde              | Gerenciado por              |
| ----------------- | ----------------- | --------------------------- |
| Dados do servidor | React Query cache | TanStack Query (hooks Kubb) |
| Estado de UI      | Zustand stores    | Zustand                     |
| Estado de form    | Componente local  | useState                    |
| Session/auth      | Cookies           | BetterAuth client           |

## Organização de componentes

```
src/components/
  ui/                   ← shadcn (nunca lógica de negócio)
  layout/               ← header, tabs, sidebar
  theme/                ← provider, toggle
  {feature}/            ← componentes da feature (SÓ componentes)
  componente-solto.tsx  ← quando não justifica pasta
```

Regras:

- `components/` contém apenas componentes — sem hooks, sem types, sem constantes
- Props interface dentro do próprio componente
- Hooks em `src/hooks/`, stores em `src/stores/`
- Pasta de feature com 3+ componentes relacionados
- Pages são thin — só compõem componentes

## Código gerado (Kubb)

```
src/gen/               ← NUNCA editar
  models/              ← Types TypeScript
  hooks/               ← React Query hooks (agrupados por tag)
  zod/                 ← Zod schemas
  clients/             ← Axios client functions
```

Regerar: `npm run api:generate`

## Requisitos

- React 19 + Vite
- TanStack Router + TanStack React Query
- Tailwind CSS 4 + shadcn/ui
- Biome (linter/formatter)
- Axios
- Kubb (dev dependency)
- Zustand + Immer (opcional)
- BetterAuth client (se usar auth)
