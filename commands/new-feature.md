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

Checar se existem hooks em `src/gen/hooks/{feature}-hooks/`. Se não existirem, avisar que o backend precisa expor a tag no OpenAPI e regerar com `npm run api:generate`.

### 2. Criar página

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

Atualizar `src/components/layout/tabs.tsx` com link pra nova feature.

## Próximos passos

Informar ao usuário:

1. Criar componentes específicos em `-components/` (card, form, delete dialog)
2. Hooks específicos em `-hooks/`, tipos derivados em `-types.ts`
3. Verificar se a página precisa de proteção com `auth` no `beforeLoad`
4. Adicionar navegação no layout
5. Em mutations, sempre usar `meta.invalidates` + `meta.successMessage` (não wrapper trivial)
