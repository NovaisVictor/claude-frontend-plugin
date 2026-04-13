---
description: "Padrões do TanStack React Query: provider config, cache, hooks gerados pelo Kubb, invalidação, optimistic updates. Use quando trabalhar com fetching de dados, cache, mutations, ou qualquer interação com a API."
---

# TanStack React Query — Convenções

## Provider

Configurado em `src/integrations/tanstack-query/root-provider.tsx`. O QueryClient é compartilhado com o TanStack Router via context.

## Hooks gerados pelo Kubb

Os hooks ficam em `src/gen/hooks/` agrupados por tag:

```typescript
// Query (GET)
import { useGetProducts } from '@/gen/hooks/products-hooks'

// Mutation (POST, PUT, DELETE, PATCH)
import { useCreateProduct } from '@/gen/hooks/products-hooks'
import { useUpdateProduct } from '@/gen/hooks/products-hooks'
import { useDeleteProduct } from '@/gen/hooks/products-hooks'
```

## Padrão de uso em componentes

```typescript
function ProductsList() {
  const { data: products, isLoading, error } = useGetProducts()

  if (isLoading) return <Skeleton />
  if (error) return <ErrorMessage error={error} />

  return (
    <div>
      {products?.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  )
}
```

## Mutations com invalidação

```typescript
function CreateProductForm() {
  const queryClient = useQueryClient()
  const { mutate, isPending } = useCreateProduct({
    mutation: {
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ['products'] })
      },
    },
  })

  const handleSubmit = (data: CreateProductRequest) => {
    mutate({ data })
  }
}
```

## Regras

- Server state (dados da API) SEMPRE no React Query — nunca em useState ou Zustand
- Client state (UI) no Zustand ou useState — nunca no React Query
- Hooks do Kubb já configuram queryKey automaticamente — não redefinir manualmente
- Invalidação após mutation via `queryClient.invalidateQueries()`
- Loading e error states vêm do hook — não criar estados manuais
- Suspense hooks disponíveis (gerados pelo Kubb com `suspense: {}`)
- Prefetch em loaders do router usa o QueryClient do context
