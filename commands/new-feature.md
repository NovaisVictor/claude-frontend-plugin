---
description: "Criar estrutura de uma nova feature no frontend: página + componente principal integrado com hooks do Kubb."
---

Criar estrutura de uma nova feature. Nome da feature em $ARGUMENTS (kebab-case, e.g. `products`).

## Nomes derivados

- Feature: kebab-case → `products`
- Pasta componentes: `src/components/{feature}/`
- Página: `src/pages/_app/{feature}/index.tsx`
- Hooks: verificar `src/gen/hooks/{feature}-hooks/`

## Passos

### 1. Verificar hooks gerados

Checar se existem hooks em `src/gen/hooks/{feature}-hooks/`. Se não existirem, avisar que o backend precisa expor a tag no OpenAPI e regerar com `npm run api:generate`.

### 2. Criar página

`src/pages/_app/{feature}/index.tsx`:

```typescript
import { createFileRoute } from '@tanstack/react-router'
import { {Feature}List } from '@/components/{feature}/{feature}-list'

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

`src/components/{feature}/{feature}-list.tsx`:

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

### 4. Adicionar tab de navegação (se necessário)

Atualizar `src/components/layout/tabs.tsx` com link pra nova feature.

## Próximos passos

Informar ao usuário:

1. Criar componentes específicos (card, form, delete dialog) dentro de `src/components/{feature}/`
2. Verificar se a página precisa de proteção com `auth` no `beforeLoad`
3. Adicionar navegação no layout
