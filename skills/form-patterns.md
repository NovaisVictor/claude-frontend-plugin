---
description: "Padrão de forms: Zod + React Hook Form + setError('root') para erros de API + useId() para IDs únicos. Use quando criar ou modificar formulários, validação de inputs, ou tratamento de erros de mutation em forms."
---

# Forms — Zod + RHF + `setError('root')`

## Tripé do form

1. **Zod** define o schema (validação + tipo).
2. **React Hook Form** gerencia state e submit.
3. **`setError('root')`** propaga erros da API pro form.

## Dependências

```bash
bun add react-hook-form @hookform/resolvers
```

## Exemplo completo

```typescript
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { useId } from 'react'
import { z } from 'zod'
import { useCreateProduct } from '@/gen/hooks/products-hooks'
import { getGetProductsQueryKey } from '@/gen/hooks/products-hooks'

const schema = z.object({
  name: z.string().min(1, 'Nome obrigatório'),
  price: z.coerce.number().positive(),
})

type Schema = z.infer<typeof schema>

export function CreateProductForm() {
  const nameId = useId()
  const priceId = useId()

  const form = useForm<Schema>({
    resolver: zodResolver(schema),
    defaultValues: { name: '', price: 0 },
  })

  const { mutate, isPending } = useCreateProduct({
    mutation: {
      meta: {
        invalidates: [getGetProductsQueryKey()],
        successMessage: 'Produto criado',
        errorMessage: false, // form trata o erro inline
      },
    },
  })

  const onSubmit = (data: Schema) => {
    mutate(
      { data },
      {
        onError: (err) => form.setError('root', { message: err.message }),
      },
    )
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <Label htmlFor={nameId}>Nome</Label>
      <Input id={nameId} {...form.register('name')} />
      {form.formState.errors.name && <span>{form.formState.errors.name.message}</span>}

      <Label htmlFor={priceId}>Preço</Label>
      <Input id={priceId} type="number" {...form.register('price')} />

      {form.formState.errors.root && (
        <p className="text-destructive">{form.formState.errors.root.message}</p>
      )}

      <Button type="submit" disabled={isPending}>Criar</Button>
    </form>
  )
}
```

## Regras

- **Schema Zod sempre** — tipo via `z.infer<typeof schema>`, nunca declarar tipo manual.
- **`zodResolver`** conecta Zod ao RHF.
- **`setError('root', { message })`** em `onError` da mutation — renderiza erro inline, não toast.
- **Toast de sucesso** vem de `meta.successMessage` (ver skill `tanstack-query`), não de código manual.
- **`errorMessage: false`** desabilita o toast de erro automático (form trata inline).

## IDs com `useId()`

Biome enforça `lint/correctness/useUniqueElementIds`. Hardcodar `id="..."` quebra build.

```typescript
// ❌ Quebra Biome
<Input id="email" />

// ✅ Usar useId()
function EmailField() {
  const id = useId()
  return (
    <>
      <Label htmlFor={id}>Email</Label>
      <Input id={id} />
    </>
  )
}
```

Aplica-se a `<Input>`, `<Checkbox>`, `<Textarea>`, `<Select>`, `<RadioGroup>` e qualquer primitivo shadcn que aceite `id`.

## Quando NÃO usar `setError('root')`

- Erro de validação que casa com um campo específico → use `setError('fieldName', ...)`.
- Erro que aparece antes do submit (ex: validação async) → use `form.setError` no handler do campo.

`root` é especificamente para erros de API/backend que não casam com nenhum campo (ex: 409 Conflict, 500, network).
