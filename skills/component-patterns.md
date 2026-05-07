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

## Theme dark/light (built-in)

Sempre vem por padrão no template (não é opt-in). Implementação **customizada** — não usa `next-themes`.

`src/components/theme/theme-provider.tsx`:

```typescript
import {
  createContext,
  type ReactNode,
  useContext,
  useEffect,
  useState,
} from 'react'

type Theme = 'dark' | 'light' | 'system'

interface ThemeContextValue {
  theme: Theme
  setTheme: (theme: Theme) => void
}

const ThemeContext = createContext<ThemeContextValue | undefined>(undefined)

const STORAGE_KEY = 'ui-theme'

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>(
    () => (localStorage.getItem(STORAGE_KEY) as Theme | null) ?? 'system',
  )

  useEffect(() => {
    const root = window.document.documentElement
    root.classList.remove('light', 'dark')

    if (theme === 'system') {
      const systemTheme = window.matchMedia('(prefers-color-scheme: dark)')
        .matches
        ? 'dark'
        : 'light'
      root.classList.add(systemTheme)
      return
    }

    root.classList.add(theme)
  }, [theme])

  const handleSetTheme = (next: Theme) => {
    localStorage.setItem(STORAGE_KEY, next)
    setTheme(next)
  }

  return (
    <ThemeContext.Provider value={{ theme, setTheme: handleSetTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

export function useTheme() {
  const ctx = useContext(ThemeContext)
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider')
  return ctx
}
```

`src/components/theme/theme-toggle.tsx` — dropdown shadcn com 3 opções (`light`, `dark`, `system`).

### Regras

- `localStorage` key fixa: `'ui-theme'`.
- Aplicar classe `light` ou `dark` no `<html>` (Tailwind v4 usa `dark:` modifier).
- `system` lê `window.matchMedia('(prefers-color-scheme: dark)')`.
- O `ThemeProvider` é montado uma vez no `__root.tsx`, fora dos providers do React Query e BetterAuth.
- O toggle aparece no footer do sidebar (next ao `UserMenu`).
- **Nunca** usar `next-themes` — só funciona com Next.js. Pra Vite + React, sempre o ThemeProvider customizado acima.
