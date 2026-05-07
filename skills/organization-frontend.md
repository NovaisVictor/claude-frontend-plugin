---
description: "Multi-tenant no frontend: rotas _app/$orgSlug/, useOrganization, OrganizationSwitcher, redirect /no-organization, gate de 2FA setup. Use quando o backend tem o plugin organization() do BetterAuth ou quando precisar implementar multi-tenant no React."
---

# Organization (Multi-tenant) — Frontend

Pattern complementar ao plugin `organization()` do BetterAuth (ver skill `organization-plugin` do `claude-auth-plugin`). Quando o backend é multi-tenant, o frontend precisa:

1. Rotear recursos da org via `_app/$orgSlug/`.
2. Trocar de org via `OrganizationSwitcher` (chama `authClient.organization.setActive`).
3. Tratar caso "user sem org" → redirect `/no-organization`.
4. Forçar setup de 2FA quando `passkey()` / `twoFactor()` estão ativos no backend.

## Estrutura de rotas

```
src/pages/
  _app/
    layout.tsx                       # gate: auth + 2FA setup
    setup-2fa.tsx                    # 3-step flow (rota standalone, não dentro de $orgSlug)
    -components/
      setup-2fa-card.tsx
    index.tsx                        # redirect → primeira org do user
    no-organization.tsx              # fallback se user sem org
    settings/                        # settings de USUÁRIO (não da org)
      profile.tsx
      security.tsx
    $orgSlug/                        # rotas org-scoped
      layout.tsx                     # extrai orgSlug e injeta no contexto
      index.tsx                      # redirect → primeiro recurso (ex: /products)
      {feature}/
        index.tsx
        -components/
        -hooks/
        -types.ts
      settings/                      # settings da ORG (members, invitations, general)
        general.tsx
        members.tsx
```

## `_app/layout.tsx` — gate de auth + 2FA setup

```tsx
import { createFileRoute, Navigate, Outlet, redirect } from '@tanstack/react-router'
import { authClient, useSession } from '@/integrations/better-auth/auth-client'

export const Route = createFileRoute('/_app')({
  beforeLoad: async () => {
    const { data: session } = await authClient.getSession({
      fetchOptions: { credentials: 'include' },
    })
    if (!session) throw redirect({ to: '/sign-in' })
  },
  component: AppLayout,
})

function AppLayout() {
  const { data: session } = useSession()
  const location = Route.useLocation()

  // Quando o backend tem twoFactor()/passkey() ativos, forçar setup
  const needsTwoFactorSetup =
    session?.user.twoFactorEnabled === false &&
    !location.pathname.startsWith('/setup-2fa')

  if (needsTwoFactorSetup) {
    return <Navigate to="/setup-2fa" />
  }

  return <Outlet />
}
```

A página `_app/setup-2fa.tsx` é uma rota standalone — não dentro de `$orgSlug` — porque o user ainda não escolheu org quando força o setup.

## Hook `useOrganization`

`src/hooks/use-organization.ts` — leitura da org ativa do user:

```typescript
import { useQuery } from '@tanstack/react-query'
import { authClient } from '@/integrations/better-auth/auth-client'

export function useOrganizations() {
  return useQuery({
    queryKey: ['organizations'],
    queryFn: async () => {
      const { data, error } = await authClient.organization.list()
      if (error) throw new Error(error.message)
      return data ?? []
    },
  })
}

export function useActiveOrganization() {
  return useQuery({
    queryKey: ['organizations', 'active'],
    queryFn: async () => {
      const { data, error } = await authClient.organization.getFullOrganization()
      if (error) throw new Error(error.message)
      return data
    },
  })
}
```

Esses hooks vivem em **`src/hooks/` global** (são cross-cutting), não em `feature/-hooks/`.

## Mutations de organization

`src/hooks/use-organization-mutations.ts` — wrappers que atualizam contexto de org:

```typescript
import { useMutation } from '@tanstack/react-query'
import { authClient } from '@/integrations/better-auth/auth-client'

const orgInvalidates = [['organizations'], ['session']] as const

export function useSetActiveOrganization() {
  return useMutation({
    meta: { invalidates: orgInvalidates },
    mutationFn: async (organizationId: string) => {
      const { data, error } = await authClient.organization.setActive({
        organizationId,
      })
      if (error) throw new Error(error.message)
      return data
    },
  })
}

export function useAcceptInvitation() {
  return useMutation({
    meta: {
      invalidates: orgInvalidates,
      successMessage: 'Convite aceito',
    },
    mutationFn: async (invitationId: string) => {
      const { data, error } = await authClient.organization.acceptInvitation({
        invitationId,
      })
      if (error) throw new Error(error.message)
      return data
    },
  })
}

export function useRejectInvitation() {
  return useMutation({
    meta: {
      invalidates: orgInvalidates,
      successMessage: 'Convite recusado',
    },
    mutationFn: async (invitationId: string) => {
      const { data, error } = await authClient.organization.rejectInvitation({
        invitationId,
      })
      if (error) throw new Error(error.message)
      return data
    },
  })
}
```

**Regra:** toda mutation que muda contexto de org invalida `[['organizations'], ['session']]` — sem isso, o `OrganizationSwitcher` não atualiza após troca, e `useSession` continua mostrando a org antiga.

## OrganizationSwitcher

Componente solto em `src/components/organization-switcher.tsx`:

```tsx
import { Link } from '@tanstack/react-router'
import { useNavigate, useParams } from '@tanstack/react-router'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
import {
  useActiveOrganization,
  useOrganizations,
} from '@/hooks/use-organization'
import { useSetActiveOrganization } from '@/hooks/use-organization-mutations'

export function OrganizationSwitcher() {
  const { data: orgs = [] } = useOrganizations()
  const { data: active } = useActiveOrganization()
  const setActive = useSetActiveOrganization()
  const navigate = useNavigate()

  return (
    <DropdownMenu>
      <DropdownMenuTrigger>{active?.name ?? 'Selecionar org'}</DropdownMenuTrigger>
      <DropdownMenuContent>
        {orgs.map((org) => (
          <DropdownMenuItem
            key={org.id}
            onClick={() => {
              setActive.mutate(org.id, {
                onSuccess: () =>
                  navigate({ to: '/$orgSlug', params: { orgSlug: org.slug } }),
              })
            }}
          >
            {org.name}
          </DropdownMenuItem>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

## Sidebar com link org-scoped

Sidebar usa `useParams({ from: '/_app/$orgSlug' })` para obter o slug e construir links:

```tsx
const { orgSlug } = useParams({ from: '/_app/$orgSlug' })

const menuItems = linkOptions([
  {
    title: 'Produtos',
    to: '/$orgSlug/products',
    params: { orgSlug },
    icon: Package,
  },
  {
    title: 'Configurações',
    to: '/$orgSlug/settings',
    params: { orgSlug },
    icon: Settings,
  },
])
```

## Redirect `/no-organization`

`_app/no-organization.tsx`:

```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/_app/no-organization')({
  component: NoOrganizationPage,
})

function NoOrganizationPage() {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Você ainda não pertence a nenhuma organização</CardTitle>
        <CardDescription>
          Peça a um admin para te enviar um convite, ou crie uma nova organização.
        </CardDescription>
      </CardHeader>
      {/* ... */}
    </Card>
  )
}
```

`_app/index.tsx` decide para onde mandar o user:

```tsx
function AppIndexRedirect() {
  const { data: orgs } = useOrganizations()

  if (!orgs?.length) return <Navigate to="/no-organization" />
  return <Navigate to="/$orgSlug" params={{ orgSlug: orgs[0].slug }} replace />
}
```

## Gate de 2FA setup

Quando o backend ativa `twoFactor()` ou `passkey()`, o frontend deve forçar setup:

1. User faz login → `_app/layout.tsx` lê session.
2. Se `session.user.twoFactorEnabled === false` e não está em `/setup-2fa` → `<Navigate to="/setup-2fa" />`.
3. `_app/setup-2fa.tsx` é a página standalone com 3 passos:
   - Confirmar senha (chama `authClient.twoFactor.enable({ password })`).
   - Mostrar QR code (TOTP URI) + backup codes (`<QRCodeSVG value={totpURI} />`).
   - Verificar TOTP de 6 dígitos (`authClient.twoFactor.verifyTotp({ code, trustDevice: true })`).
4. Após verificar com sucesso: `window.location.href = '/'`.

Mesmo padrão se aplica para regenerar backup codes na settings/security do user.

## Public invitation route (signup-via-invite)

A rota pública `GET /invitations/:id` (no backend) retorna payload mínimo (sem `inviterId`). No frontend é consumida via hook gerado pelo Kubb (`useGetInvitationsByInvitationId`) — usado em:

- `_auth/sign-up.tsx` para mostrar "você foi convidado para X com email Y" antes de criar conta.
- `_app/invite/$invitationId.tsx` para aceitar/recusar quando o user já tem sessão.

Não criar wrapper trivial sobre `useGetInvitationsByInvitationId` — usar direto.

## Regras

- Hooks de organization (`useOrganizations`, `useActiveOrganization`) ficam em `src/hooks/` global.
- Mutations de organization usam `meta.invalidates: [['organizations'], ['session']]`.
- Recursos da org vivem em `pages/_app/$orgSlug/{feature}/` — nunca em `pages/_app/{feature}/`.
- `useParams({ from: '/_app/$orgSlug' })` para extrair slug em componentes filhos.
- `_app/setup-2fa.tsx` é standalone (fora de `$orgSlug`).
- Settings de **usuário** (profile, security) ficam em `pages/_app/settings/`. Settings de **org** ficam em `pages/_app/$orgSlug/settings/`.

## Frontend-reviewer checks

- Recurso multi-tenant fora de `_app/$orgSlug/` → ERRO.
- Componente que usa orgSlug sem `useParams({ from: '/_app/$orgSlug' })` → AVISO.
- Mutation que muda contexto de org sem `meta.invalidates: [['organizations'], ['session']]` → AVISO.
- `_app/layout.tsx` sem gate de 2FA quando backend tem `twoFactor()`/`passkey()` → AVISO.
- Wrapper trivial sobre hooks de invitation/organization (que só forwarda sem lógica) → AVISO: usar Kubb ou BetterAuth client direto.
