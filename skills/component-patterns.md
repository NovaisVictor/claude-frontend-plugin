---
description: "Padrões de componentes React com shadcn/ui e Tailwind CSS. Use quando criar ou modificar componentes, trabalhar com UI, forms, dialogs, ou qualquer elemento visual."
---

# Componentes — Convenções

## Organização

```
src/components/
  ui/                       ← shadcn primitivos (nunca lógica de negócio)
    button.tsx
    dialog.tsx
    alert-dialog.tsx
  layout/                   ← Estrutura do app
    header.tsx
    tabs.tsx
    logo.tsx
  theme/                    ← Tema
    theme-provider.tsx
    theme-toggle.tsx
  banners/                  ← Feature com 3+ componentes
    banners-tabs.tsx
    banner-card.tsx
    banner-form.tsx
    delete-banner-alert.tsx
  brand-switcher.tsx        ← Componente solto (não justifica pasta)
```

## Props

Interface de props fica dentro do próprio componente:

```typescript
interface DeleteBannerAlertProps {
  open: boolean
  onOpenChange: (open: boolean) => void
  bannerTitle: string
  onConfirm: () => void
}

export function DeleteBannerAlert({
  open,
  onOpenChange,
  bannerTitle,
  onConfirm,
}: DeleteBannerAlertProps) {
  return (
    <AlertDialog open={open} onOpenChange={onOpenChange}>
      {/* ... */}
    </AlertDialog>
  )
}
```

## Estilo

- Tailwind CSS utility classes — nunca CSS modules ou styled-components
- `cn()` de `@/lib/utils` pra merge condicional de classes
- Composição com shadcn/Radix primitivos

```typescript
import { cn } from '@/lib/utils'

<Link
  className={cn(
    'py-1.5 px-3 hover:text-primary transition-all duration-150 font-medium',
    isActive
      ? 'border-b border-primary text-primary'
      : 'text-muted-foreground',
  )}
>
```

## Regras

- Componentes são funções exportadas com nome — nunca arrow functions exportadas como default
- Named exports sempre: `export function Header()`, não `export default function()`
- Props interface fica no próprio arquivo, não em arquivo separado
- Nenhum hook customizado dentro de `components/` — hooks ficam em `src/hooks/`
- Nenhum arquivo de types dentro de `components/` — usar Zod ou inline
- `ui/` é exclusivamente shadcn — personalizar via props, não modificando o primitivo
- Componentes de feature recebem dados via props — não fazem fetch diretamente (exceto se for o componente raiz da feature que usa o hook do Kubb)
- Pasta de feature só quando tem 3+ componentes relacionados
