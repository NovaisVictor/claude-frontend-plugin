---
description: "Configuração do Kubb para geração de código a partir do OpenAPI spec. Use quando precisar configurar, regerar, ou entender o código gerado pelo Kubb (types, hooks, zod schemas, clients)."
---

# Kubb — Configuração e Geração

## Dependências

```bash
bun add -d @kubb/core @kubb/plugin-oas @kubb/plugin-ts @kubb/plugin-react-query @kubb/plugin-zod @kubb/plugin-client
```

## kubb.config.ts

```typescript
import { defineConfig } from '@kubb/core'
import { pluginOas } from '@kubb/plugin-oas'
import { pluginTs } from '@kubb/plugin-ts'
import { pluginReactQuery } from '@kubb/plugin-react-query'
import { pluginZod } from '@kubb/plugin-zod'

function toKebabCase(str: string): string {
  return str
    .replace(/([a-z])([A-Z])/g, '$1-$2')
    .replace(/[\s_]+/g, '-')
    .toLowerCase()
}

export default defineConfig({
  root: '.',
  input: {
    path: 'http://localhost:8080/openapi/json',
  },
  output: {
    path: './src/gen',
    clean: true,
    format: 'biome',
    lint: 'biome',
  },
  plugins: [
    pluginOas({
      validate: false,
    }),
    pluginTs({
      output: {
        path: './models',
        barrelType: 'named',
      },
    }),
    pluginZod({
      output: {
        path: './zod',
        barrelType: 'named',
      },
      typed: true,
      inferred: true,
    }),
    pluginReactQuery({
      output: {
        path: './hooks',
        barrelType: 'named',
      },
      client: {
        dataReturnType: 'data',
      },
      query: {
        methods: ['get'],
        importPath: '@tanstack/react-query',
      },
      mutation: {
        methods: ['post', 'put', 'delete', 'patch'],
      },
      suspense: {},
      exclude: [
        { type: 'tag', pattern: 'Better Auth' },
      ],
      group: {
        type: 'tag',
        name: ({ group }) => `${toKebabCase(group)}-hooks`,
      },
    }),
  ],
})
```

## O que é gerado

```
src/gen/
  models/           ← Types TypeScript (request, response, path params)
  hooks/            ← React Query hooks (useQuery, useMutation, suspense)
    {tag}-hooks/    ← Agrupados por tag do OpenAPI
  zod/              ← Zod schemas com tipos inferidos
  clients/          ← Funções Axios client
```

## Comandos

```bash
bun run generate           # Gerar uma vez
bun run generate --watch   # Watch mode (dev)
```

Adicionar ao `package.json`:

```json
{
  "scripts": {
    "generate": "kubb generate"
  }
}
```

## Regras

- `src/gen/` é NUNCA editado manualmente — sempre regerar via Kubb
- Adicionar `src/gen/` ao `.gitignore` (regerar no CI)
- Rotas de auth (`Better Auth` tag) são excluídas — auth usa BetterAuth client
- Hooks são agrupados por tag em kebab-case
- `dataReturnType: 'data'` retorna direto o data, sem wrapper ResponseConfig
- Output formatado e lintado pelo Biome automaticamente
- O input path aponta pro endpoint OpenAPI do backend Elysia

## Usando os hooks gerados

```typescript
import { useGetProducts, useCreateProduct, getGetProductsQueryKey } from '@/gen/hooks/products-hooks'
import { z } from 'zod'
import { getProductResponseSchema } from '@/gen/zod'

type Product = z.infer<typeof getProductResponseSchema>

function ProductsList() {
  const { data: products, isLoading } = useGetProducts()
  const { mutate: createProduct } = useCreateProduct()

  // types, loading states, cache — tudo automático
}
```

## Imports permitidos

- `@/gen/hooks/*` ✅ — primeira opção para qualquer endpoint OpenAPI.
- `@/gen/zod/*` ✅ — derive tipos via `z.infer<typeof xxxSchema>`.
- `@/gen/models/*` ❌ **PROIBIDO** — interno do Kubb, sujeito a renomeações entre versões.
- Manual `useQuery`/`useMutation` apenas para BetterAuth SDK (não está na OpenAPI spec — ver skill `auth-client`).

## Custom client (importPath)

Se precisar customizar o client Axios (ex: baseURL, interceptors), configurar `client.importPath` no pluginReactQuery apontando pra `@/lib/api-client`.
