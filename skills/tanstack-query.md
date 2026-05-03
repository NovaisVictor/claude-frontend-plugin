---
description: "Padrões do TanStack React Query: provider config, MutationCache central com meta.invalidates / meta.successMessage, hooks gerados pelo Kubb. Use quando trabalhar com fetching de dados, cache, mutations, ou qualquer interação com a API."
---

# TanStack React Query — Convenções

## Provider e MutationCache

Configurado em `src/integrations/tanstack-query/root-provider.tsx`. Toda invalidação e toast de mutation passam pela `MutationCache` central:

```typescript
import { MutationCache, QueryClient } from '@tanstack/react-query'
import { toast } from 'sonner'

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
      toast.error(typeof errorMessage === 'string' ? errorMessage : error.message)
    },
  }),
})
```

Tipos do `meta` em `src/integrations/tanstack-query/mutation-meta.d.ts`:

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

## Hooks gerados pelo Kubb

Os hooks ficam em `src/gen/hooks/` agrupados por tag:

```typescript
import { useGetProducts, useCreateProduct, getGetProductsQueryKey } from '@/gen/hooks/products-hooks'
```

## Padrão de uso em queries

```typescript
function ProductsList() {
  const { data: products, isLoading, error } = useGetProducts()

  if (isLoading) return <Skeleton />
  if (error) return <ErrorMessage error={error} />

  return (
    <div>
      {products?.map(product => <ProductCard key={product.id} product={product} />)}
    </div>
  )
}
```

## Mutations — `meta.invalidates` + `meta.successMessage`

Invalidação e toast vivem no `meta`, não em wrapper `onSuccess`:

```typescript
const { mutate, isPending } = useCreateProduct({
  mutation: {
    meta: {
      invalidates: [getGetProductsQueryKey()],
      successMessage: 'Produto criado',
    },
  },
})

const handleSubmit = (data: CreateProductRequest) => mutate({ data })
```

## Quando criar wrapper hook

Wrapper trivial (`onSuccess: invalidate`) é redundante — não faça. Wrapper só se justifica quando há lógica adicional:

- **Bulk** com `Promise.all` — invalida uma vez no fim, não a cada item.
- **Navegação** após sucesso (`router.navigate`).
- **Side effect cross-domain** (atualiza store Zustand, dispara analytics).

```typescript
// Bulk operation com invalidação centralizada
export function useDeleteProductsBulk() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (ids: string[]) => {
      await Promise.all(ids.map((id) => deleteProduct({ id })))
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: getGetProductsQueryKey() })
    },
  })
}
```

## Forms + mutations

Quando a mutation alimenta um form, use `errorMessage: false` no `meta` e trate o erro com `setError('root')` (ver skill `form-patterns`):

```typescript
const { mutate } = useCreateProduct({
  mutation: { meta: { invalidates: [...], errorMessage: false } },
})

mutate({ data }, { onError: (err) => form.setError('root', { message: err.message }) })
```

## Regras

- Server state (dados da API) SEMPRE no React Query — nunca em `useState` ou Zustand.
- Client state (UI) no Zustand ou `useState` — nunca no React Query.
- Hooks do Kubb já configuram `queryKey` automaticamente — usar `getGetXQueryKey()` para invalidação.
- **Invalidação via `meta.invalidates`** — não usar `onSuccess: queryClient.invalidateQueries(...)` em wrapper trivial.
- **Toast de sucesso via `meta.successMessage`** — não chamar `toast.success` manualmente.
- **`meta.errorMessage: false`** quando o componente trata o erro (ex: form com `setError('root')`).
- Loading e error states vêm do hook — não criar estados manuais.
- Suspense hooks disponíveis (gerados pelo Kubb com `suspense: {}`).
- Prefetch em loaders do router usa o QueryClient do context.
