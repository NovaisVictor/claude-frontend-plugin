Configurar Kubb num projeto frontend existente com React + TanStack Query.

## Pré-requisitos

Verificar que o projeto tem: React, TanStack React Query, Vite, Biome.

## Passos

### 1. Instalar dependências

```bash
npm install axios
npm install -D @kubb/core @kubb/plugin-oas @kubb/plugin-ts @kubb/plugin-react-query @kubb/plugin-zod @kubb/plugin-client
```

### 2. Criar kubb.config.ts

Na raiz do projeto, seguir o padrão da skill `kubb-config`.

### 3. Criar src/lib/api-client.ts

Seguir o padrão da skill `api-client`.

### 4. Adicionar variável de ambiente

Em `.env`:
```env
VITE_API_URL=http://localhost:3333
```

### 5. Adicionar scripts ao package.json

```json
{
  "scripts": {
    "api:generate": "kubb generate",
    "api:watch": "kubb generate --watch"
  }
}
```

### 6. Adicionar src/gen/ ao .gitignore

```gitignore
src/gen/
```

### 7. Adicionar ao biome.json

Incluir `!**/src/gen/**` nos files includes pra ignorar código gerado.

### 8. Gerar código

```bash
npm run api:generate
```

### Próximos passos

1. Verificar `src/gen/` com os arquivos gerados
2. Importar hooks em componentes: `import { useGetX } from '@/gen/hooks/x-hooks'`
3. Configurar `client.importPath` se precisar de Axios customizado
