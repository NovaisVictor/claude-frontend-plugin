---
description: "Configuração do Axios client para integração com Kubb e BetterAuth. Use quando precisar configurar HTTP client, base URL, interceptors, cookies de auth, ou entender como as requests chegam ao backend."
---

# API Client — Axios

## src/lib/api-client.ts

```typescript
import axios from 'axios'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  withCredentials: true,
})

apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      window.location.href = '/sign-in'
    }
    return Promise.reject(error)
  },
)
```

## Integração com Kubb

O Kubb pode consumir o client customizado via `client.importPath` no `pluginReactQuery`. Quando configurado, todos os hooks gerados usam essa instância Axios.

Se usar `importPath`, o módulo deve exportar:

```typescript
export type RequestConfig<TData = unknown> = {
  url?: string
  method: 'GET' | 'PUT' | 'PATCH' | 'POST' | 'DELETE'
  params?: object
  data?: TData | FormData
  responseType?: 'arraybuffer' | 'blob' | 'document' | 'json' | 'text' | 'stream'
  signal?: AbortSignal
  headers?: HeadersInit
}

export type ResponseConfig<TData = unknown> = {
  data: TData
  status: number
  statusText: string
}

export type ResponseErrorConfig<TError = unknown> = TError

export type Client = <TData, _TError = unknown, TVariables = unknown>(
  config: RequestConfig<TVariables>,
) => Promise<ResponseConfig<TData>>
```

## Auth via cookies

O BetterAuth gerencia session via cookies HTTP-only. O Axios com `withCredentials: true` envia esses cookies automaticamente em toda request. Não é necessário gerenciar tokens manualmente.

Fluxo:
1. User faz login via BetterAuth client → cookie de session setado
2. Request de domínio via hook Kubb → Axios envia cookie automaticamente
3. Backend valida session via macro `auth` → retorna dados ou 401

## Variáveis de ambiente

```env
VITE_API_URL=http://localhost:3333
```

## Regras

- `withCredentials: true` é obrigatório pra enviar cookies de session
- Interceptor de 401 redireciona pra sign-in (session expirada)
- Nunca armazenar tokens em localStorage — BetterAuth usa cookies HTTP-only
- Base URL vem de variável de ambiente com prefixo `VITE_`
