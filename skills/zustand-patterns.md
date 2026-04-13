---
description: "Padrões de Zustand para client state: stores, selectors, Immer middleware. Use quando trabalhar com estado de UI, filtros, sidebar, wizard steps, ou qualquer estado que não vem do servidor."
---

# Zustand — Convenções

## Dependências

```bash
npm install zustand immer
```

## Quando usar Zustand vs React Query

| Zustand | React Query |
|---------|-------------|
| Sidebar aberto/fechado | Lista de produtos da API |
| Filtros ativos na UI | Detalhes de um produto |
| Step atual de um wizard | Resultado de busca |
| Item selecionado pra edição | Session do usuário |
| Tema, preferências locais | Qualquer dado do servidor |

**Regra**: se o dado vem do servidor → React Query. Se é estado local da UI → Zustand.

## Store simples

```typescript
// src/stores/use-sidebar-store.ts
import { create } from 'zustand'

interface SidebarStore {
  isOpen: boolean
  toggle: () => void
  close: () => void
}

export const useSidebarStore = create<SidebarStore>((set) => ({
  isOpen: false,
  toggle: () => set((state) => ({ isOpen: !state.isOpen })),
  close: () => set({ isOpen: false }),
}))
```

## Store com Immer (estado complexo)

Usar Immer quando o estado tem objetos aninhados ou arrays que precisam de updates parciais:

```typescript
// src/stores/use-cart-store.ts
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

interface CartItem {
  id: string
  name: string
  quantity: number
}

interface CartStore {
  items: CartItem[]
  addItem: (item: Omit<CartItem, 'quantity'>) => void
  updateQuantity: (id: string, quantity: number) => void
  removeItem: (id: string) => void
}

export const useCartStore = create<CartStore>()(
  immer((set) => ({
    items: [],
    addItem: (item) =>
      set((state) => {
        const existing = state.items.find((i) => i.id === item.id)
        if (existing) {
          existing.quantity += 1
        } else {
          state.items.push({ ...item, quantity: 1 })
        }
      }),
    updateQuantity: (id, quantity) =>
      set((state) => {
        const item = state.items.find((i) => i.id === id)
        if (item) item.quantity = quantity
      }),
    removeItem: (id) =>
      set((state) => {
        state.items = state.items.filter((i) => i.id !== id)
      }),
  })),
)
```

## Uso em componentes

```typescript
function Sidebar() {
  const { isOpen, toggle } = useSidebarStore()
  // ...
}

// Ou com selector pra evitar re-renders desnecessários:
function SidebarToggle() {
  const toggle = useSidebarStore((state) => state.toggle)
  // ...
}
```

## Regras

- Stores ficam em `src/stores/` com prefixo `use-`
- Um store por domínio de estado (sidebar, cart, filters) — não um mega-store
- Usar Immer middleware apenas quando o state tem arrays ou objetos aninhados
- Selectors pra evitar re-renders: `useStore((state) => state.field)`
- Nunca colocar dados do servidor no Zustand — usar React Query
- Actions ficam dentro do store, não em funções externas
