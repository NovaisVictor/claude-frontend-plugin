---
description: "Criar estrutura de uma nova feature no frontend: página + componente principal integrado com hooks do Kubb."
---

Criar estrutura de uma nova feature. Nome da feature em $ARGUMENTS (kebab-case, e.g. `products`).

## Nomes derivados

- Feature: kebab-case → `products`
- Página: `src/pages/_app/{feature}/index.tsx`
- Co-location: `src/pages/_app/{feature}/-components/`, `-hooks/`, `-types.ts`
- Hooks Kubb: verificar `src/gen/hooks/{feature}-hooks/`

## Passos

### 1. Verificar hooks gerados

Checar se existem hooks em `src/gen/hooks/{feature}-hooks/`. Se não existirem, avisar que o backend precisa expor a tag no OpenAPI e regerar com `bun run generate`.

### 2. Decidir org-scoped vs single-tenant

Se a feature é multi-tenant (recursos pertencem a uma organização), criar em `src/pages/_app/$orgSlug/{feature}/` e extrair `orgSlug` via `useParams({ from: '/_app/$orgSlug' })`.

Se single-tenant (recursos globais ou do user), criar em `src/pages/_app/{feature}/`.

A decisão deve casar com o backend: rotas que usam `activeOrg: true` no Elysia → frontend org-scoped.

### 3. Criar página (single-tenant)

`src/pages/_app/{feature}/index.tsx`:

```typescript
import { createFileRoute } from '@tanstack/react-router'
import { {Feature}List } from './-components/{feature}-list'

export const Route = createFileRoute('/_app/{feature}/')({
  component: {Feature}Page,
})

function {Feature}Page() {
  return (
    <div className="w-full">
      <h1 className="text-2xl font-bold mb-4 md:mb-6">{Feature}</h1>
      <{Feature}List />
    </div>
  )
}
```

### 3b. Criar página (multi-tenant)

`src/pages/_app/$orgSlug/{feature}/index.tsx`:

```typescript
import { createFileRoute } from '@tanstack/react-router'
import { {Feature}List } from './-components/{feature}-list'

export const Route = createFileRoute('/_app/$orgSlug/{feature}/')({
  component: {Feature}Page,
})

function {Feature}Page() {
  return (
    <div className="w-full">
      <h1 className="text-2xl font-bold mb-4 md:mb-6">{Feature}</h1>
      <{Feature}List />
    </div>
  )
}
```

A query `useGet{Feature}s()` (Kubb) já bate em endpoint `activeOrg: true` no backend — o `organizationId` vem da sessão, não precisa passar como param.

### 3. Criar componente principal

`src/pages/_app/{feature}/-components/{feature}-list.tsx`:

```typescript
import { useGet{Feature}s } from '@/gen/hooks/{feature}-hooks'

export function {Feature}List() {
  const { data, isLoading } = useGet{Feature}s()

  if (isLoading) return <div>Carregando...</div>

  return (
    <div className="space-y-4">
      {data?.map(item => (
        <div key={item.id}>{/* TODO: componente de card */}</div>
      ))}
    </div>
  )
}
```

### 4. (Opcional) Form de criação com mutation

Quando a feature inclui form, seguir o padrão da skill `form-patterns`:

- Schema Zod + `z.infer` em `-types.ts`
- React Hook Form com `zodResolver`
- `setError('root')` no `onError` da mutation
- `meta.invalidates: [getGet{Feature}sQueryKey()]` + `meta.successMessage` (skill `tanstack-query`)
- `useId()` para IDs de Input/Label

### 5. Adicionar tab de navegação (se necessário)

Atualizar `src/components/layout/app-sidebar.tsx` com link pra nova feature. Em multi-tenant, usar `linkOptions` com `params: { orgSlug }`.

## Próximos passos

Informar ao usuário:

1. Criar componentes específicos em `-components/` (card, form, delete dialog)
2. Hooks específicos em `-hooks/`, tipos derivados em `-types.ts`
3. Em mutations, sempre usar `meta.invalidates` + `meta.successMessage` (não wrapper trivial)
4. IDs de Input/Label sempre via `useId()` (Biome enforça)
5. Para features multi-tenant, nunca passar `organizationId` no payload — vem da sessão via macro `activeOrg` no backend
6. Adicionar navegação no `app-sidebar.tsx` (com `params: { orgSlug }` se multi-tenant)
