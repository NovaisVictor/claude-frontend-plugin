---
description: "Configuração do BetterAuth client no React. Use quando trabalhar com login, signup, session, proteção de rotas no frontend, ou qualquer funcionalidade de autenticação client-side."
---

# BetterAuth Client — React

## Dependências

```bash
npm install @better-auth/react
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
