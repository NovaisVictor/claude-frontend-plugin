---
description: "Configuração do BetterAuth client no React. Use quando trabalhar com login, signup, session, proteção de rotas no frontend, ou qualquer funcionalidade de autenticação client-side."
---

# BetterAuth Client — React

## Dependências

```bash
bun add @better-auth/react
```

## src/integrations/better-auth/auth-client.ts

```typescript
import { createAuthClient } from 'better-auth/react'

export const authClient = createAuthClient({
  baseURL: import.meta.env.VITE_API_URL,
})
```

## Uso em componentes

### Acessar session

```typescript
import { authClient } from '@/integrations/better-auth/auth-client'

function ProfileMenu() {
  const { data: session, isPending } = authClient.useSession()

  if (isPending) return <Skeleton />
  if (!session) return null

  return <span>{session.user.name}</span>
}
```

### Login

```typescript
function SignInForm() {
  const handleSignIn = async (data: { email: string; password: string }) => {
    await authClient.signIn.email({
      email: data.email,
      password: data.password,
    })
  }
}
```

### Magic link

```typescript
const handleMagicLink = async (email: string) => {
  await authClient.signIn.magicLink({ email })
}
```

### Logout

```typescript
const handleSignOut = async () => {
  await authClient.signOut()
  window.location.href = '/sign-in'
}
```

## Proteção de rotas no frontend

No layout `_app/layout.tsx`, verificar session e redirecionar:

```typescript
import { createFileRoute, Outlet, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_app')({
  beforeLoad: async () => {
    // Verificar session no servidor antes de renderizar
    const response = await fetch(`${import.meta.env.VITE_API_URL}/auth/get-session`, {
      credentials: 'include',
    })
    if (!response.ok) {
      throw redirect({ to: '/sign-in' })
    }
  },
  component: DashboardLayout,
})
```

## Regras

- Auth client é separado do Axios client — BetterAuth gerencia suas próprias requests
- Session é verificada via `authClient.useSession()` nos componentes
- Proteção de rotas usa `beforeLoad` do TanStack Router, não guards no componente
- Nunca armazenar tokens manualmente — BetterAuth usa cookies HTTP-only
- O `baseURL` do auth client aponta pro mesmo backend que o Axios
- Rotas de auth (`/sign-in`, `/sign-up`) ficam no layout group `_auth/` (público)
- Rotas do app ficam no layout group `_app/` (protegido)

## Wrapping BetterAuth com `useMutation`

Hooks do BetterAuth **não** passam pelo Kubb (não estão na OpenAPI spec). Ficam em `src/hooks/use-auth.ts` **global** — não em `feature/-hooks/`.

Padrão: sempre invalidar a session via `meta.invalidates` para os componentes que dependem dela re-renderizarem.

```typescript
// src/hooks/use-auth.ts
import { useMutation } from '@tanstack/react-query'
import { authClient } from '@/integrations/better-auth/auth-client'

const sessionInvalidates = [['session']] as const

export function useSignInEmailPassword() {
  return useMutation({
    meta: { invalidates: sessionInvalidates },
    mutationFn: async ({ email, password }: { email: string; password: string }) => {
      const result = await authClient.signIn.email({ email, password })
      if (result.error) throw new Error(result.error.message)
      return result.data
    },
  })
}

export function useSignOut() {
  return useMutation({
    meta: { invalidates: sessionInvalidates },
    mutationFn: async () => {
      const result = await authClient.signOut()
      if (result.error) throw new Error(result.error.message)
      return result.data
    },
  })
}
```

Aplica-se a sign-in, sign-up, sign-out, verify 2FA, enable passkey, e todos os wrappers sobre `authClient.organization.*`. Hooks que mudam contexto de organização invalidam `[['organizations'], ['session']]`.

Cross-cutting (auth, organization, media queries) é a única exceção que justifica viver em `src/hooks/` global. Hooks específicos de feature ficam em `feature/-hooks/`.
