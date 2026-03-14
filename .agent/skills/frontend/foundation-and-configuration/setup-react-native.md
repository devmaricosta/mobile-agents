---
name: setup-react-native
description: Configuração inicial de projetos React Native com Expo Managed Workflow (SDK 54+), estrutura de pastas, tsconfig com path aliases, ESLint Flat Config, Prettier, Husky com pre-commit hooks e Metro bundler.
---

# Setup Inicial — React Native com Expo (SDK 54+)

Você é um Engenheiro de Software Senior especialista em configuração de projetos React Native. Sua responsabilidade é garantir que a fundação do projeto seja sólida, padronizada e pronta para escalar, seguindo as melhores práticas da comunidade e compatível com o **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

---

## Quando usar esta skill

- **Novo projeto:** Inicializar um projeto React Native do zero.
- **Padronização:** Adicionar ou corrigir configurações de lint, format, hooks ou bundler em projetos existentes.
- **Onboarding:** Garantir que o ambiente de desenvolvimento de um novo membro do time esteja configurado corretamente.
- **Geração de arquivos:** Criar ou atualizar arquivos de configuração (`tsconfig.json`, `eslint.config.js`, `prettier.config.js`, `metro.config.js`, etc.).

---

## 1. Expo Managed vs Bare Workflow

### Decisão (Árvore de Critérios)

```text
Precisa de módulos nativos customizados (ex: Bluetooth, NFC, SDKs proprietários)?
├── NÃO → Use Expo Managed Workflow ✅ (padrão preferido)
└── SIM → O módulo existe no Expo SDK ou como Expo Module?
           ├── SIM → Use Expo Managed Workflow ✅
           └── NÃO → Use Expo Bare Workflow (com `npx expo prebuild`)
```

### Regra de ouro

> **Sempre comece com Expo Managed.** Migrar para Bare é simples com `npx expo prebuild`. O inverso é complexo.

### Inicialização

```bash
# Expo Managed (padrão)
npx create-expo-app@latest my-app --template blank-typescript

# Expo Bare (somente se necessário)
npx create-expo-app@latest my-app --template bare-minimum
```

> **Nota SDK 54:** Este é o último SDK com suporte à Legacy Architecture. Novos projetos devem usar a New Architecture (habilitada por padrão).

---

## 2. Estrutura de Pastas

A estrutura abaixo é **obrigatória** e alinhada com a skill `arquiteto-react-native-features`.

```text
my-app/
├── src/
│   ├── app/                    # ROTEAMENTO (Expo Router) — camada fina
│   │   ├── _layout.tsx
│   │   ├── (auth)/
│   │   │   ├── _layout.tsx
│   │   │   └── login.tsx
│   │   └── (home)/
│   │       ├── _layout.tsx
│   │       └── index.tsx
│   │
│   ├── features/               # LÓGICA DE DOMÍNIO
│   │   └── auth/
│   │       ├── store/          # Store Zustand do domínio auth
│   │       │   ├── auth.store.ts
│   │       │   └── index.ts    # Exporta hooks públicos da store
│   │       └── login/
│   │           ├── api/
│   │           ├── components/
│   │           ├── hooks/
│   │           ├── screens/
│   │           ├── types/
│   │           └── index.ts
│   │
│   ├── components/             # UI Kit global compartilhado
│   ├── config/                 # Variáveis de ambiente e constantes de configuração
│   │   └── env.ts
│   ├── constants/              # Constantes globais
│   │   └── routes.ts           # Rotas da aplicação (ROUTES)
│   ├── hooks/                  # Hooks globais reutilizáveis
│   ├── i18n/                   # Internacionalização (react-i18next)
│   │   ├── locales/
│   │   │   ├── pt-BR.json
│   │   │   └── en-US.json
│   │   └── index.ts
│   ├── services/               # Clientes externos (Axios, AsyncStorage)
│   │   ├── api.ts
│   │   └── storage.ts
│   ├── theme/                  # Design tokens
│   │   ├── colors.ts
│   │   ├── spacing.ts
│   │   └── index.ts
│   ├── types/                  # Tipos globais e declarações de módulo
│   │   └── svg.d.ts
│   └── utils/                  # Funções auxiliares puras
│       ├── format-date.ts
│       └── validators.ts
│
├── assets/                     # Imagens, fontes, ícones
├── .husky/                     # Git hooks
├── eslint.config.js            # ESLint Flat Config (SDK 53+)
├── prettier.config.js
├── commitlint.config.js
├── metro.config.js
├── tsconfig.json
└── package.json
```

> **Nota sobre stores:** Não existe `src/store/` global. Cada store Zustand reside dentro da feature de domínio que a possui (`features/<feature>/store/`). Features distintas que precisam de dados de outra store importam via barrel público da feature dona.

---

## 3. TypeScript — tsconfig com Path Aliases

### Resolução de aliases no SDK 54

> **IMPORTANTE:** A partir do SDK 53, o Expo CLI resolve path aliases definidos no `tsconfig.json` **automaticamente** via `tsconfigPaths` (habilitado por padrão). **NÃO é necessário** instalar `babel-plugin-module-resolver`. O Metro e o Babel já leem os paths do `tsconfig.json` diretamente.

### `tsconfig.json`

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "exactOptionalPropertyTypes": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@features/*": ["src/features/*"],
      "@hooks/*": ["src/hooks/*"],
      "@services/*": ["src/services/*"],
      "@theme/*": ["src/theme/*"],
      "@utils/*": ["src/utils/*"],
      "@config/*": ["src/config/*"],
      "@constants/*": ["src/constants/*"],
      "@i18n/*": ["src/i18n/*"]
    }
  },
  "include": ["src"]
}
```

### Tabela de aliases

| Alias | Aponta para | Uso típico |
|---|---|---|
| `@/*` | `src/*` | Importação genérica de qualquer módulo em `src/` |
| `@components/*` | `src/components/*` | UI Kit global |
| `@features/*` | `src/features/*` | Acesso ao barrel público de uma feature (telas, stores, tipos) |
| `@hooks/*` | `src/hooks/*` | Hooks globais reutilizáveis |
| `@services/*` | `src/services/*` | Axios, AsyncStorage, etc. |
| `@theme/*` | `src/theme/*` | Cores, espaçamentos, tipografia |
| `@utils/*` | `src/utils/*` | Funções puras auxiliares |
| `@config/*` | `src/config/*` | Variáveis de ambiente |
| `@constants/*` | `src/constants/*` | Constantes globais (ex: `ROUTES`) |
| `@i18n/*` | `src/i18n/*` | Internacionalização |

> **Nota:** O alias `@store/*` foi removido pois não existe mais `src/store/` global. Stores ficam dentro de cada feature e são acessadas via `@features/<feature>` (ex: `import { useAuthStore } from '@features/auth'`).

> **Nota sobre Query Keys:** Não existe `src/constants/query-keys.ts` global. Cada feature define sua própria fábrica de query keys em `features/<feature>/queries/query-keys.ts`. O arquivo global só existe como exceção para entidades consultadas por 3+ features sem um dono claro.

### Considerações

- **Reinicie o Expo CLI** após modificar `tsconfig.json` para atualizar os aliases.
- Não é necessário limpar o cache do Metro quando os aliases mudam.
- Para desabilitar a resolução automática (não recomendado):
  ```json
  // app.json
  { "expo": { "experiments": { "tsconfigPaths": false } } }
  ```

---

## 4. Babel

No SDK 54 com `tsconfigPaths` habilitado, o `babel.config.js` permanece **simples** — sem necessidade de `babel-plugin-module-resolver`.

### `babel.config.js`

```js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
  };
};
```

> **Nota:** O React Compiler é habilitado por padrão no template default do SDK 54. Se estiver usando o template `blank-typescript`, o preset `babel-preset-expo` já inclui tudo o que é necessário.

---

## 5. ESLint (Flat Config — SDK 53+)

### Mudança importante

> **A partir do SDK 53/54**, o ESLint usa **Flat Config** (`eslint.config.js`) como padrão. O formato legado `.eslintrc.js` está deprecado. O pacote `eslint-config-expo` v10+ usa o formato flat.

### Instalação

```bash
# Instala ESLint + config do Expo (via Expo CLI para garantir compatibilidade)
npx expo install -- --save-dev \
  eslint \
  eslint-config-expo \
  @typescript-eslint/eslint-plugin \
  @typescript-eslint/parser \
  eslint-plugin-import \
  eslint-plugin-react-hooks \
  eslint-plugin-react-native \
  eslint-import-resolver-typescript
```

### `eslint.config.js`

```js
const expoConfig = require('eslint-config-expo/flat');
const tseslint = require('@typescript-eslint/eslint-plugin');
const tsParser = require('@typescript-eslint/parser');
const importPlugin = require('eslint-plugin-import');
const reactHooksPlugin = require('eslint-plugin-react-hooks');
const reactNativePlugin = require('eslint-plugin-react-native');

/** @type {import('eslint').Linter.FlatConfig[]} */
module.exports = [
  // Base do Expo (inclui React, TypeScript e regras essenciais)
  ...expoConfig,

  // Configuração global
  {
    languageOptions: {
      parser: tsParser,
      parserOptions: {
        project: './tsconfig.json',
      },
    },
    settings: {
      'import/resolver': {
        typescript: { project: './tsconfig.json' },
      },
    },
  },

  // Regras customizadas para arquivos TS/TSX
  {
    files: ['**/*.{ts,tsx}'],
    plugins: {
      '@typescript-eslint': tseslint,
      import: importPlugin,
      'react-hooks': reactHooksPlugin,
      'react-native': reactNativePlugin,
    },
    rules: {
      // TypeScript
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
      '@typescript-eslint/no-explicit-any': 'error',

      // React Hooks
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',

      // Imports — ordenação consistente
      'import/order': [
        'error',
        {
          groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
          pathGroups: [
            { pattern: 'react', group: 'external', position: 'before' },
            { pattern: 'react-native', group: 'external', position: 'before' },
            { pattern: 'expo*', group: 'external', position: 'before' },
            { pattern: '@/**', group: 'internal' },
          ],
          pathGroupsExcludedImportTypes: ['react', 'react-native'],
          'newlines-between': 'always',
          alphabetize: { order: 'asc', caseInsensitive: true },
        },
      ],
      'import/no-duplicates': 'error',
      'import/no-cycle': 'error',

      // Geral
      'no-console': ['warn', { allow: ['warn', 'error'] }],
    },
  },

  // Arquivos ignorados
  {
    ignores: ['node_modules/', '.expo/', 'dist/', 'coverage/', 'android/', 'ios/'],
  },
];
```

---

## 6. Prettier

### Instalação

```bash
npx expo install -- --save-dev \
  prettier \
  eslint-config-prettier \
  eslint-plugin-prettier
```

Adicione as configs do Prettier ao array do `eslint.config.js`:

```js
const prettierConfig = require('eslint-config-prettier');
const prettierPlugin = require('eslint-plugin-prettier');

module.exports = [
  // ... demais configs acima

  // Prettier — deve ser sempre o último
  prettierConfig,
  {
    plugins: { prettier: prettierPlugin },
    rules: { 'prettier/prettier': 'error' },
  },
];
```

### `prettier.config.js`

```js
/** @type {import('prettier').Config} */
module.exports = {
  // Formatação geral
  printWidth: 100,
  tabWidth: 2,
  useTabs: false,
  semi: true,
  singleQuote: true,
  quoteProps: 'as-needed',
  trailingComma: 'all',
  bracketSpacing: true,
  bracketSameLine: false,
  arrowParens: 'always',

  // Arquivos específicos
  overrides: [
    {
      files: '*.json',
      options: { printWidth: 80 },
    },
    {
      files: ['*.md', '*.mdx'],
      options: { proseWrap: 'always' },
    },
  ],
};
```

### `.prettierignore`

```
node_modules/
.expo/
dist/
coverage/
android/
ios/
*.lock
```

---

## 7. Husky + lint-staged + commitlint

### Instalação

```bash
# Husky e lint-staged
npm install --save-dev husky lint-staged

# Commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Inicializar Husky
npx husky init
```

> **Nota:** Husky, lint-staged e commitlint são dev tools que não interagem com o Expo SDK. Use `npm install --save-dev` para instalá-los.

### Pre-commit hook — `.husky/pre-commit`

```bash
npx lint-staged
```

### Commit-msg hook — `.husky/commit-msg`

```bash
npx --no -- commitlint --edit "$1"
```

### `lint-staged` — em `package.json`

```json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix --max-warnings=0",
      "prettier --write"
    ],
    "*.{js,json,md}": [
      "prettier --write"
    ]
  }
}
```

> **Por que não rodar `tsc` no pre-commit?** O type-check completo é lento e deve rodar na CI/CD. O pre-commit deve ser rápido (< 10s) para não frustrar o desenvolvedor.

### `commitlint.config.js`

```js
/** @type {import('@commitlint/types').UserConfig} */
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'build', 'ci', 'chore', 'revert'],
    ],
    'subject-case': [2, 'always', 'lower-case'],
    'subject-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 100],
  },
};
```

### Padrão de commits (Conventional Commits)

```
<type>(<scope>): <subject>

feat(auth): add biometric login support
fix(home): correct scroll behavior on android
docs(readme): update setup instructions
refactor(api): extract error handler to service layer
```

---

## 8. Metro Bundler

### Suporte a SVG

```bash
npx expo install react-native-svg
npx expo install -- --save-dev react-native-svg-transformer
```

### `metro.config.js`

```js
const { getDefaultConfig } = require('expo/metro-config');

/** @type {import('expo/metro-config').MetroConfig} */
const config = getDefaultConfig(__dirname);

// ─── Suporte a SVG ────────────────────────────────────────────────────────────
const { transformer, resolver } = config;

config.transformer = {
  ...transformer,
  babelTransformerPath: require.resolve('react-native-svg-transformer'),
};

config.resolver = {
  ...resolver,
  assetExts: resolver.assetExts.filter((ext) => ext !== 'svg'),
  sourceExts: [...resolver.sourceExts, 'svg'],
};

module.exports = config;
```

> **Nota SDK 54:**
> - `experimentalImportSupport` é habilitado por padrão (para suporte ao React Compiler e tree shaking). Não é necessário configurá-lo manualmente.
> - Os path aliases são resolvidos automaticamente pelo Expo CLI via `tsconfigPaths`. **Não é necessário** configurar `extraNodeModules` no Metro.
> - Imports internos do Metro mudaram de `metro/src/..` para `metro/private/..` (relevante apenas para autores de bibliotecas).

### Declaração de tipos para SVG — `src/types/svg.d.ts`

```ts
declare module '*.svg' {
  import type React from 'react';
  import type { SvgProps } from 'react-native-svg';
  const content: React.FC<SvgProps>;
  export default content;
}
```

---

## 9. TanStack Query v5

O **TanStack Query v5** é o padrão do projeto para gerenciamento de **estado de servidor** (cache, loading, erro de chamadas de API). Complementa o Zustand, que gerencia estado de sessão e UI global.

> **Regra:** Use TanStack Query para tudo que vem do servidor. Use Zustand para estado de sessão (token, usuário logado) e estado de UI global.

### 9.1 Instalação

```bash
npx expo install @tanstack/react-query
npm install --save-dev @tanstack/react-query-devtools
```

### 9.2 Configuração do `QueryClient`

Crie o `QueryClient` com defaults otimizados para aplicativos mobile:

```ts
// src/services/query-client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Dados ficam "frescos" por 2 minutos — evita refetches desnecessários
      staleTime: 2 * 60 * 1000,

      // Cache mantido por 5 minutos após componente desmontar (padrão)
      gcTime: 5 * 60 * 1000,

      // Retry inteligente: 2 tentativas com backoff exponencial
      // (o interceptor do Axios já trata 401/403 — não fazer retry neles)
      retry: 2,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30_000),

      // Refetch ao reconectar à rede (importante em mobile)
      refetchOnReconnect: true,

      // Não refetch ao focar a janela (comportamento web — irrelevante em mobile)
      refetchOnWindowFocus: false,
    },
    mutations: {
      // Não faz retry em mutations por padrão (evita duplicar ações)
      retry: false,
    },
  },
});
```

### 9.3 `QueryClientProvider` no Layout Raiz

```tsx
// src/app/_layout.tsx
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@services/query-client';

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      {/* demais providers (ThemeProvider, etc.) */}
      <Stack />
    </QueryClientProvider>
  );
}
```

### 9.4 Query Key Factories — Co-localizadas na Feature

Use o padrão **Query Key Factory** (recomendado pelo TkDodo) para estruturar as chaves de query de forma hierárquica. Cada feature define sua própria fábrica dentro da pasta `queries/`.

> **Regra de localização:** Query keys **pertencem à feature** que possui aquela entidade. Defina cada factory em `features/<feature>/queries/query-keys.ts`. Um arquivo global em `src/constants/query-keys.ts` só existe como exceção para entidades consultadas por 3 ou mais features sem um dono claro.

```ts
// src/features/products/queries/query-keys.ts
export const productKeys = {
  // Raiz: invalida TUDO relacionado a produtos
  all: () => ['products'] as const,

  // Lista: invalida todas as listas (com ou sem filtros)
  lists: () => [...productKeys.all(), 'list'] as const,

  // Lista com filtros específicos
  list: (filters?: Record<string, unknown>) =>
    [...productKeys.lists(), { filters }] as const,

  // Detalhe: invalida todos os detalhes
  details: () => [...productKeys.all(), 'detail'] as const,

  // Detalhe de um item específico
  detail: (id: string) => [...productKeys.details(), id] as const,
} as const;

// Uso:
// queryClient.invalidateQueries({ queryKey: productKeys.all() })      // invalida tudo
// queryClient.invalidateQueries({ queryKey: productKeys.lists() })    // invalida listas
// queryClient.invalidateQueries({ queryKey: productKeys.detail(id) }) // invalida um item
```

### 9.5 Padrão de Hook com `useQuery`

```ts
// src/features/products/queries/use-products.ts
import { useQuery } from '@tanstack/react-query';
import type { UseQueryResult } from '@tanstack/react-query';
import { getProducts } from '../services/products.service';
import type { ApiError } from '@services/api/errors';
import type { Product } from '../types/products.types';
import { productKeys } from './query-keys'; // co-localizado na feature

export function useProducts(filters?: Record<string, unknown>): UseQueryResult<Product[], ApiError> {
  return useQuery<Product[], ApiError>({
    queryKey: productKeys.list(filters),
    queryFn: async () => {
      const result = await getProducts(filters);
      // Converte Result<T, E> → throw para o TanStack Query capturar
      if (result.isErr()) throw result.error;
      return result.value;
    },
    // Não faz retry para erros de autenticação/permissão
    retry: (failureCount, error) => {
      if (error.type === 'UNAUTHORIZED' || error.type === 'FORBIDDEN') return false;
      return failureCount < 2;
    },
  });
}
```

### 9.6 Padrão de Hook com `useMutation`

```ts
// src/features/products/mutations/use-create-product.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import type { UseMutationResult } from '@tanstack/react-query';
import { createProduct } from '../services/products.service';
import type { ApiError } from '@services/api/errors';
import type { CreateProductPayload, Product } from '../types/products.types';
import { productKeys } from '../queries/query-keys'; // co-localizado na feature

export function useCreateProduct(): UseMutationResult<Product, ApiError, CreateProductPayload> {
  const queryClient = useQueryClient();

  return useMutation<Product, ApiError, CreateProductPayload>({
    mutationFn: async (payload) => {
      const result = await createProduct(payload);
      if (result.isErr()) throw result.error;
      return result.value;
    },
    onSuccess: () => {
      // Invalida a lista de produtos para refetch automático
      queryClient.invalidateQueries({ queryKey: productKeys.lists() });
    },
    // onError é tipado como ApiError — sem `unknown`, sem cast
    onError: (error) => {
      if (error.type === 'UNKNOWN') console.error('[useCreateProduct] Erro:', error.message);
    },
  });
}
```

---

## 10. Fluxo de Setup Completo (Passo a Passo)

Ao receber a solicitação de configurar um novo projeto:

**1. Criar o projeto**
```bash
npx create-expo-app@latest my-app --template blank-typescript
cd my-app
```

**2. Criar a estrutura de pastas**
Criar os diretórios conforme a seção 2. Adicionar `.gitkeep` em pastas vazias.

**3. Configurar TypeScript**
Substituir/atualizar `tsconfig.json` conforme a seção 3 (aliases são resolvidos automaticamente).

**4. Confirmar Babel**
O `babel.config.js` padrão do template já é suficiente (seção 4). Não adicionar `module-resolver`.

**5. Configurar ESLint (Flat Config)**
Instalar dependências e criar `eslint.config.js` conforme a seção 5.

**6. Configurar Prettier**
Instalar dependências e criar `prettier.config.js` e `.prettierignore` conforme a seção 6.

**7. Configurar Husky + lint-staged + commitlint**
Seguir os passos da seção 7 na ordem exata.

**8. Configurar Metro**
Instalar dependências SVG e atualizar `metro.config.js` conforme a seção 8.
Criar `src/types/svg.d.ts`.

**9. Configurar TanStack Query**
Instalar dependências e criar `src/services/query-client.ts` e `src/constants/query-keys.ts` conforme a seção 9.
Adicionar `QueryClientProvider` ao `src/app/_layout.tsx`.

**10. Adicionar scripts ao `package.json`**
```json
{
  "scripts": {
    "start": "expo start",
    "android": "expo start --android",
    "ios": "expo start --ios",
    "lint": "npx expo lint",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,js,json,md}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,js,json,md}\"",
    "type-check": "tsc --noEmit",
    "prepare": "husky"
  }
}
```

---

## Checklist de Verificação

Antes de considerar o setup concluído, verifique cada item:

### Expo e Estrutura
- [ ] Projeto criado com `blank-typescript` (Managed Workflow)?
- [ ] Usando Expo SDK 54 (React Native 0.81)?
- [ ] Estrutura de pastas criada conforme a seção 2?
- [ ] Pastas `src/config/`, `src/constants/`, `src/i18n/`, `src/types/` criadas?

### TanStack Query
- [ ] `@tanstack/react-query` instalado via `npx expo install`?
- [ ] `src/services/query-client.ts` criado com defaults otimizados para mobile?
- [ ] `QueryClientProvider` adicionado ao `src/app/_layout.tsx`?
- [ ] Query Key Factories criadas em `features/<feature>/queries/query-keys.ts` (co-localizadas)?
- [ ] `refetchOnWindowFocus: false` configurado (comportamento web — irrelevante em mobile)?
- [ ] `refetchOnReconnect: true` configurado (importante em mobile)?

### TypeScript
- [ ] `tsconfig.json` extende `expo/tsconfig.base` com `strict: true`?
- [ ] Todos os aliases (`@/*`, `@components/*`, etc.) definidos em `paths`?
- [ ] `tsconfigPaths` está habilitado (padrão — **não** desabilitado no `app.json`)?
- [ ] **NÃO** tem `babel-plugin-module-resolver` instalado (desnecessário no SDK 54)?
- [ ] `npx tsc --noEmit` roda sem erros?

### ESLint
- [ ] Usando **Flat Config** (`eslint.config.js`), não `.eslintrc.js`?
- [ ] `eslint-config-expo` v10+ instalado?
- [ ] `npx expo lint` roda sem erros no projeto inicial?

### Prettier
- [ ] `prettier.config.js` e `.prettierignore` criados?
- [ ] `eslint-config-prettier` como último item no array do flat config?
- [ ] `npm run format:check` roda sem erros?

### Husky e Commits
- [ ] `.husky/pre-commit` criado e executável?
- [ ] `.husky/commit-msg` criado e executável?
- [ ] `lint-staged` configurado no `package.json`?
- [ ] `commitlint.config.js` criado?
- [ ] Um commit de teste com mensagem inválida é rejeitado?
- [ ] Um commit de teste com mensagem válida (`feat: initial setup`) é aceito?

### Metro
- [ ] `metro.config.js` configurado com suporte a SVG?
- [ ] `src/types/svg.d.ts` criado?
- [ ] **NÃO** tem `extraNodeModules` manual para aliases (resolvido pelo `tsconfigPaths`)?
- [ ] Um arquivo `.svg` pode ser importado como componente sem erros?

### Scripts
- [ ] Script `lint` usa `npx expo lint`?
- [ ] Todos os scripts (`format`, `format:check`, `type-check`, `prepare`) definidos?
