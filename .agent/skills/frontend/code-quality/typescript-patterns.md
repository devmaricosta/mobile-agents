---
name: typescript-patterns
description: Padrões de TypeScript para o projeto React Native — quando usar type vs interface, generics em hooks e funções de API, discriminated unions para estados assíncronos, e como tipar stores do Zustand corretamente.
---

# Padrões de TypeScript — React Native com Expo (SDK 54+)

Você é um Engenheiro de Software Senior especialista em TypeScript avançado aplicado a projetos React Native. Sua responsabilidade é garantir que o código do projeto siga padrões de tipagem consistentes, seguros e amigáveis ao desenvolvedor, aproveitando ao máximo os recursos do TypeScript ~5.9 com `strict: true`.

---

## Quando usar esta skill

- **Modelagem de tipos:** Decidir entre `type` e `interface` ao criar novos tipos.
- **Hooks customizados:** Criar hooks genéricos e type-safe.
- **Funções de API:** Tipar chamadas HTTP e respostas com generics.
- **Estados assíncronos:** Modelar estados de loading/error/success com discriminated unions.
- **Store Zustand:** Tipar stores, slices, selectors e middleware corretamente.
- **Code review:** Validar se o código segue os padrões de tipagem do projeto.

---

## 1. `type` vs `interface` — Quando Usar Cada Um

### Regra de Ouro

> **Use `type` por padrão.** Reserve `interface` apenas quando precisar de **declaration merging** (ex: augmentar tipos de bibliotecas de terceiros) ou quando estiver definindo contratos para classes.

### Tabela de Decisão

| Cenário | Usar | Motivo |
|---|---|---|
| Props de componentes | `type` | Simples, fechado, sem risco de merge acidental |
| Retorno de hooks | `type` | Frequentemente usa union/intersection |
| Respostas de API (DTOs) | `type` | Imutável — reflete contrato do backend |
| Union / Intersection types | `type` | `interface` **não suporta** unions |
| Tuplas | `type` | `interface` **não suporta** tuplas |
| Tipos primitivos ou aliases | `type` | `interface` **não suporta** primitivos |
| Mapped / Conditional types | `type` | Funcionalidade exclusiva de `type` |
| States Zustand | `type` | Consistência com o resto do projeto |
| Augmentar módulo de terceiros | `interface` | Requer declaration merging |
| Contratos de classes / OOP | `interface` | Design pattern clássico com `implements` |

### Exemplos

```ts
// ✅ CORRETO — type para props
type LoginFormProps = {
  onSubmit: (credentials: Credentials) => void;
  isLoading: boolean;
};

// ✅ CORRETO — type para union
type Theme = 'light' | 'dark' | 'system';

// ✅ CORRETO — type para intersection
type UserWithPermissions = User & { permissions: Permission[] };

// ✅ CORRETO — interface para declaration merging
// Augmentando expo-constants para incluir variáveis de ambiente customizadas
declare module 'expo-constants' {
  interface AppManifest {
    extra: {
      apiUrl: string;
      environment: 'development' | 'staging' | 'production';
    };
  }
}
```

### ❌ Anti-padrões

```ts
// ❌ ERRADO — interface para union (não funciona)
interface Theme { value: 'light' | 'dark' } // Verboso e desnecessário

// ❌ ERRADO — interface para props simples (risco de merge acidental)
interface ButtonProps { label: string; }
interface ButtonProps { size: number; } // Merge silencioso! Bug potencial.

// ❌ ERRADO — any em qualquer lugar
const fetchData = (url: string): any => { ... }
```

---

## 2. Generics — Hooks e Funções de API

### 2.1 Hooks Genéricos

Use generics para criar hooks reutilizáveis que mantêm type safety em diferentes contextos.

#### Padrão: Hook genérico com constraints

```ts
// src/hooks/use-debounce.ts
import { useEffect, useState } from 'react';

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Uso — o tipo é inferido automaticamente
const debouncedSearch = useDebounce('termo de busca', 300); // string
const debouncedCount = useDebounce(5, 500);                 // number
```

#### Padrão: Hook com constraint `extends`

```ts
// src/hooks/use-storage.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { useCallback, useEffect, useState } from 'react';

// Constraint: T deve ser serializável
type Serializable = string | number | boolean | null | Serializable[] | { [key: string]: Serializable };

export function useStorage<T extends Serializable>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(defaultValue);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    AsyncStorage.getItem(key).then((stored) => {
      if (stored !== null) {
        setValue(JSON.parse(stored) as T);
      }
      setIsLoading(false);
    });
  }, [key]);

  const set = useCallback(
    async (newValue: T | ((prev: T) => T)) => {
      const resolved = typeof newValue === 'function'
        ? (newValue as (prev: T) => T)(value)
        : newValue;
      setValue(resolved);
      await AsyncStorage.setItem(key, JSON.stringify(resolved));
    },
    [key, value],
  );

  return [value, set, isLoading] as const;
}

// Uso
const [theme, setTheme, isLoading] = useStorage<'light' | 'dark'>('theme', 'light');
```

### 2.2 Funções de API Genéricas

Use generics para tipar requisições e respostas HTTP de forma type-safe com Axios.

#### Padrão: Cliente API type-safe

```ts
// src/services/api.ts
import axios from 'axios';
import type { AxiosRequestConfig, AxiosResponse } from 'axios';

import { env } from '@config/env';

const apiClient = axios.create({
  baseURL: env.API_URL,
  timeout: 15_000,
  headers: { 'Content-Type': 'application/json' },
});

// Funções genéricas de request
export async function apiGet<TResponse>(
  url: string,
  config?: AxiosRequestConfig,
): Promise<TResponse> {
  const response: AxiosResponse<TResponse> = await apiClient.get(url, config);
  return response.data;
}

export async function apiPost<TResponse, TBody = unknown>(
  url: string,
  body: TBody,
  config?: AxiosRequestConfig,
): Promise<TResponse> {
  const response: AxiosResponse<TResponse> = await apiClient.post(url, body, config);
  return response.data;
}

export async function apiPut<TResponse, TBody = unknown>(
  url: string,
  body: TBody,
  config?: AxiosRequestConfig,
): Promise<TResponse> {
  const response: AxiosResponse<TResponse> = await apiClient.put(url, body, config);
  return response.data;
}

export async function apiDelete<TResponse = void>(
  url: string,
  config?: AxiosRequestConfig,
): Promise<TResponse> {
  const response: AxiosResponse<TResponse> = await apiClient.delete(url, config);
  return response.data;
}
```

#### Padrão: Uso nos serviços de feature

```ts
// src/features/auth/login/services/login.service.ts
import { fromAxiosPromise, type ApiResultAsync } from '@services/api/result';
import { api } from '@services/api/client';

import type { LoginRequest, LoginResponse } from '../types';

/** Retorna Result — nunca lança exceção */
export function loginUser(credentials: LoginRequest): ApiResultAsync<LoginResponse> {
  return fromAxiosPromise(
    api.post<LoginResponse>('/auth/login', credentials).then((r) => r.data),
  );
}
```

```ts
// src/features/auth/login/types/index.ts
export type LoginRequest = {
  email: string;
  password: string;
};

export type LoginResponse = {
  accessToken: string;
  refreshToken: string;
  user: UserProfile;
};

export type UserProfile = {
  id: string;
  name: string;
  email: string;
  avatarUrl: string | null;
};
```

#### Padrão: Resposta paginada genérica

```ts
// src/types/api.ts
// Apenas tipos de estrutura de resposta genérica (sem ApiError — ele vive em @services/api/errors)
export type PaginatedResponse<T> = {
  data: T[];
  meta: {
    currentPage: number;
    totalPages: number;
    totalItems: number;
    perPage: number;
  };
};

// Uso
import type { PaginatedResponse } from '@/types/api';
import { api } from '@services/api/client';
import { fromAxiosPromise, type ApiResultAsync } from '@services/api/result';

type Product = { id: string; name: string; price: number };

function getProducts(page: number): ApiResultAsync<PaginatedResponse<Product>> {
  return fromAxiosPromise(
    api.get<PaginatedResponse<Product>>(`/products?page=${page}`).then((r) => r.data),
  );
}
```

> **Sobre `ApiError`:** Não defina `ApiError` em `src/types/api.ts`. O tipo definitivo vive exclusivamente em `src/services/api/errors.ts` como uma *discriminated union*. Importe sempre de `@services/api/errors`.

### 2.3 Regras para Generics

| Regra | Exemplo |
|---|---|
| Prefixe com `T` para tipos genéricos | `<TResponse>`, `<TData>`, `<TError>` |
| Prefixe com `K` para keys | `<K extends keyof T>` |
| Use constraints (`extends`) sempre que possível | `<T extends Record<string, unknown>>` |
| Evite generics desnecessários | Se o tipo é sempre `string`, não use `<T>` |
| Deixe o TypeScript inferir quando óbvio | `useState(0)` → infere `number`, não precisa de `<number>` |
| Explicite generics em API calls | `apiGet<User>('/users/1')` → sempre explicitar |

---

## 3. Discriminated Unions — Estados Assíncronos

### Conceito

Discriminated unions (tagged unions) usam uma **propriedade discriminante** com valores literais para que o TypeScript faça **narrowing automático** em cada ramo do `if`/`switch`.

### 3.1 AsyncState — Padrão global

Defina um tipo reutilizável para estados assíncronos em `src/types/async-state.ts`:

```ts
// src/types/async-state.ts

/** Estado idle — nenhuma operação iniciada */
type IdleState = {
  status: 'idle';
};

/** Estado de carregamento */
type LoadingState = {
  status: 'loading';
};

/** Estado de sucesso — contém os dados */
type SuccessState<T> = {
  status: 'success';
  data: T;
};

/** Estado de erro — contém a mensagem e código opcional */
type ErrorState = {
  status: 'error';
  error: {
    message: string;
    code?: string | number;
  };
};

/** Union discriminada por `status` */
export type AsyncState<T> = IdleState | LoadingState | SuccessState<T> | ErrorState;
```

### 3.2 Uso em Hooks — Apenas para Estado Local

> **Importante:** `AsyncState<T>` é adequado para **estado local puro** (ex: fluxos de UI multi-etapa, wizard steps). Para dados remotos (API calls), use **TanStack Query** (`useQuery` / `useMutation`) — que já fornece `isPending`, `isError`, `isSuccess` e `data` tipados.

```ts
// src/features/auth/login/mutations/use-login.ts
// Exemplo CORRETO: useMutation para chamar a API de login
import { useMutation } from '@tanstack/react-query';
import type { UseMutationResult } from '@tanstack/react-query';
import { useQueryClient } from '@tanstack/react-query';

import { loginUser } from '../services/login.service';
import type { ApiError } from '@services/api/errors';
import type { LoginRequest, LoginResponse } from '../types';

export function useLogin(): UseMutationResult<LoginResponse, ApiError, LoginRequest> {
  const queryClient = useQueryClient();

  return useMutation<LoginResponse, ApiError, LoginRequest>({
    mutationFn: async (credentials) => {
      const result = await loginUser(credentials);
      if (result.isErr()) throw result.error; // Result → throw (TanStack captura)
      return result.value;
    },
    onSuccess: (data) => {
      // Limpa o cache e redireciona após login
      queryClient.resetQueries();
      // ...store.setUser(data.user)
    },
    onError: (error) => {
      // `error` já é ApiError — sem `unknown`
      if (error.type === 'UNAUTHORIZED') console.warn('Credenciais inválidas');
    },
  });
}
```

### 3.3 Uso em Componentes — com `useMutation`

```tsx
// src/features/auth/login/screens/login-screen.tsx
import { useLogin } from '../mutations/use-login';

export function LoginScreen() {
  const { mutate: login, isPending, isError, error } = useLogin();

  return (
    <View>
      {isPending && <ActivityIndicator />}

      {isError && (
        <Text style={styles.error}>
          {error.type === 'VALIDATION_ERROR'
            ? 'Verifique suas credenciais'
            : 'Erro ao fazer login. Tente novamente.'}
        </Text>
      )}

      <Button
        onPress={() => login({ email, password })}
        loading={isPending}
        disabled={isPending}
      >
        Entrar
      </Button>
    </View>
  );
}
```

> **Quando usar `AsyncState<T>` vs `useMutation`:**
>
> | Cenário | Use |
> |---|---|
> | Chamada à API (fetch, post, put, delete) | `useMutation` ou `useQuery` |
> | Estado local de wizard / multi-step | `AsyncState<T>` com `useState` |
> | Estado de upload de arquivo local | `AsyncState<T>` com `useState` |

### 3.4 Exhaustiveness Check com `never`

Sempre que usar `switch` com discriminated unions, adicione um `default` com `never` para garantir que todos os casos estão cobertos. Se um novo status for adicionado, o TypeScript acusará erro de compilação.

```ts
function renderState<T>(state: AsyncState<T>) {
  switch (state.status) {
    case 'idle':
      return null;
    case 'loading':
      return <ActivityIndicator />;
    case 'success':
      return <DataView data={state.data} />;
    case 'error':
      return <ErrorView message={state.error.message} />;
    default: {
      // Exhaustiveness check — garante que todos os casos foram tratados
      const _exhaustive: never = state;
      return _exhaustive;
    }
  }
}
```

### 3.5 Outros Usos de Discriminated Unions

```ts
// Navegação condicional
type NavigationAction =
  | { type: 'push'; screen: string; params?: Record<string, unknown> }
  | { type: 'replace'; screen: string; params?: Record<string, unknown> }
  | { type: 'goBack' }
  | { type: 'reset'; index: number; routes: { name: string }[] };

// Variantes de componente
type ButtonVariant =
  | { variant: 'solid'; color: string }
  | { variant: 'outline'; borderColor: string }
  | { variant: 'ghost' };

// Resultado de validação
type ValidationResult =
  | { valid: true }
  | { valid: false; errors: string[] };
```

### 3.6 Tipagem com TanStack Query v5

**Quando usar `AsyncState<T>` vs TanStack Query:**

| Situação | Usar |
|---|---|
| Estado de UI local (modal, contador, tema) | `useState` / `AsyncState<T>` |
| Dados de formulário / validação local | `useState` |
| Dados do servidor (GET, cache, refetch) | TanStack Query `useQuery` |
| Escrita no servidor (POST, PUT, DELETE) | TanStack Query `useMutation` |
| Estado global de sessão (token, usuário) | Zustand (não TanStack Query) |

> **Regra:** `AsyncState<T>` é para estado local sem cache. Se os dados vêm da API, **sempre use TanStack Query**.

#### `useQuery` — Tipagem completa

```ts
import { useQuery } from '@tanstack/react-query';
import type { ApiError } from '@services/api/errors';

// useQuery<TData, TError, TSelectData, TQueryKey>
// - TData: tipo dos dados retornados pelo queryFn
// - TError: tipo do erro (use ApiError do projeto)
export function useProducts() {
  return useQuery<Product[], ApiError>({
    queryKey: productKeys.lists(),
    queryFn: async () => {
      const result = await getProducts();
      if (result.isErr()) throw result.error;
      return result.value;
    },
  });
}

// Uso no componente — data e error são totalmente tipados
const { data, error, isLoading, refetch } = useProducts();
//     ^      ^      
//  Product[] | undefined   ApiError | null
```

#### `useMutation` — Tipagem genérica

```ts
import { useMutation } from '@tanstack/react-query';

// useMutation<TData, TError, TVariables, TContext>
// - TData: tipo do retorno da mutation (sucesso)
// - TError: tipo do erro (ApiError)
// - TVariables: tipo do argumento de `mutate()`
const mutation = useMutation<Product, ApiError, CreateProductPayload>({
  mutationFn: async (payload) => {
    const result = await createProduct(payload);
    if (result.isErr()) throw result.error;
    return result.value;
  },
  // Todos os callbacks são totalmente tipados:
  onSuccess: (data) => { /* data: Product */ },
  onError: (error) => { /* error: ApiError */ },
  onMutate: (variables) => { /* variables: CreateProductPayload */ },
});

// mutation.error é ApiError, não Error nem unknown
mutation.mutate({ name: 'Produto', price: 99.90 });
```

#### Tipo de retorno de hooks com TanStack Query

```ts
import type { UseQueryResult } from '@tanstack/react-query';
import type { ApiError } from '@services/api/errors';

// Para documentar o tipo do retorno de um custom hook:
export function useProducts(): UseQueryResult<Product[], ApiError> {
  return useQuery<Product[], ApiError>({ ... });
}
```


---

## 4. Tipagem de Stores Zustand

> **Referência:** Para a arquitetura completa de stores (persist, immer, slices, seletores), consulte a skill `zustand-architecture`. Esta seção cobre exclusivamente os **padrões de tipagem TypeScript**.

### 4.1 Padrão Base — Tipagem Explícita Obrigatória

Sempre defina o tipo explicitamente para stores Zustand. A tipagem inferida é propensa a erros quando o estado cresce ou quando middlewares são adicionados.

### 4.2 Store com Tipagem Explícita — Padrão Obrigatório

Sempre defina o tipo explicitamente em 3 partes: `State` (dados), `Actions` (funções) e `Store` (interseção). Defina `initialState` separado para facilitar o reset.

```ts
// src/features/auth/store/auth.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

// 1️⃣ Tipo do state — apenas dados serializáveis
type AuthState = {
  accessToken: string | null;
  refreshToken: string | null;
  user: UserProfile | null;
  isAuthenticated: boolean;
};

// 2️⃣ Tipo das actions — funções que modificam o state
type AuthActions = {
  setAuth: (user: UserProfile, accessToken: string, refreshToken: string) => void;
  setTokens: (accessToken: string, refreshToken: string) => void;
  logout: () => void;
};

// 3️⃣ Tipo completo da store = State + Actions
type AuthStore = AuthState & AuthActions;

// 4️⃣ Estado inicial (reutilizado no reset)
const initialState: AuthState = {
  accessToken: null,
  refreshToken: null,
  user: null,
  isAuthenticated: false,
};

// 5️⃣ Store com tipagem explícita + middleware
export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      ...initialState,

      setAuth: (user, accessToken, refreshToken) =>
        set({ user, accessToken, refreshToken, isAuthenticated: true }),

      setTokens: (accessToken, refreshToken) =>
        set({ accessToken, refreshToken }),

      logout: () => set({ ...initialState }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
      // Persista apenas dados necessários — nunca actions
      partialize: (state) => ({
        accessToken: state.accessToken,
        refreshToken: state.refreshToken,
        user: state.user,
        isAuthenticated: state.isAuthenticated,
      }),
      version: 1,
    },
  ),
);
```

### 4.3 Selectors Atômicos e Type-Safe

> **Regra:** Nunca use `useStore()` sem selector. Sempre selecione apenas o que o componente precisa para evitar re-renders desnecessários.

```ts
// ❌ ERRADO — subscreve a store inteira, re-renderiza em QUALQUER mudança
const store = useAuthStore();

// ❌ ERRADO — cria um NOVO objeto a cada render (referência instável)
const { user, isAuthenticated } = useAuthStore((s) => ({
  user: s.user,
  isAuthenticated: s.isAuthenticated,
}));

// ✅ CORRETO — selector individual para cada valor
const user = useAuthStore((s) => s.user);
const isAuthenticated = useAuthStore((s) => s.isAuthenticated);

// ✅ CORRETO — quando precisa de múltiplos valores, use useShallow
import { useShallow } from 'zustand/react/shallow';

const { user, isAuthenticated } = useAuthStore(
  useShallow((s) => ({ user: s.user, isAuthenticated: s.isAuthenticated })),
);

// ✅ CORRETO — selector para actions (actions nunca mudam, sem re-render)
const logout = useAuthStore((s) => s.logout);
```

### 4.4 Custom Hooks sobre a Store

> **Regra:** Exporte custom hooks em vez da store diretamente. Isso encapsula a lógica do selector e facilita refatorações.

```ts
// src/features/auth/store/auth.store.ts (no final do arquivo)

// Hooks de conveniência — exporte ESTES, não a store bruta
export const useUser = () => useAuthStore((s) => s.user);
export const useIsAuthenticated = () => useAuthStore((s) => s.isAuthenticated);
export const useAuthActions = () =>
  useAuthStore(
    useShallow((s) => ({
      setTokens: s.setTokens,
      setUser: s.setUser,
      logout: s.logout,
    })),
  );
```

```ts
// src/features/auth/store/index.ts — barrel público da store
export { useAuthStore, useUser, useIsAuthenticated, useAuthActions } from './auth.store';
```

```ts
// Outra feature consumindo a store de auth via barrel público
// src/features/profile/hooks/use-profile.ts
import { useUser } from '@features/auth'; // ✅ via barrel público, não caminho direto
```

### 4.5 Slices Pattern — Tipagem de Stores Compostas

Para stores com sub-domínios distintos, divida em slices. O `StateCreator` recebe 4 parâmetros de tipo:

```ts
// src/features/auth/store/slices/session.slice.ts
import type { StateCreator } from 'zustand';
import type { UserProfile } from '../../types/auth.types';

export type SessionSlice = {
  user: UserProfile | null;
  isAuthenticated: boolean;
  setUser: (user: UserProfile | null) => void;
  clearSession: () => void;
};

// StateCreator<FullStore, Middlewares, Mutators, ThisSlice>
export const createSessionSlice: StateCreator<
  SessionSlice & TokenSlice, // tipo completo da store (todos os slices)
  [],                        // middlewares (ex: [['zustand/immer', never]])
  [],                        // mutators
  SessionSlice               // tipo deste slice
> = (set) => ({
  user: null,
  isAuthenticated: false,
  setUser: (user) => set({ user, isAuthenticated: user !== null }),
  clearSession: () => set({ user: null, isAuthenticated: false }),
});
```

```ts
// src/features/auth/store/auth.store.ts — combinação de slices
import { create } from 'zustand';

import { createSessionSlice, type SessionSlice } from './slices/session.slice';
import { createTokenSlice, type TokenSlice } from './slices/token.slice';

// Tipo completo = interseção de todos os slices
type AuthStore = SessionSlice & TokenSlice;

export const useAuthStore = create<AuthStore>()((...args) => ({
  ...createSessionSlice(...args),
  ...createTokenSlice(...args),
}));
```

> **Regra de domínio:** Cada slice pertence ao mesmo domínio da store. Não misture slices de domínios diferentes na mesma store (ex: `cart` não pertence à feature `auth`). Para arquitetura completa de slices, consulte a skill `zustand-architecture`.

### 4.6 Acessar State fora de Componentes React

```ts
// Em interceptors do Axios, funções utilitárias, etc.
const token = useAuthStore.getState().accessToken;

// Subscribe a mudanças fora de React
const unsubscribe = useAuthStore.subscribe(
  (state) => state.isAuthenticated,
  (isAuthenticated) => {
    if (!isAuthenticated) {
      // Redirecionar para login
    }
  },
);
```

---

## 5. Padrões Gerais de Tipagem

### 5.1 Importação de Tipos

Sempre use `import type` para importações que são apenas tipos. Isso é enforçado pela regra do ESLint `@typescript-eslint/consistent-type-imports`.

```ts
// ✅ CORRETO
import type { UserProfile } from '../types';
import type { AxiosRequestConfig } from 'axios';

// ❌ ERRADO — mistura tipo e valor na mesma importação
import { UserProfile } from '../types';
```

### 5.2 Tipagem de Componentes React

```ts
// ✅ CORRETO — componente funcional com type
type AvatarProps = {
  uri: string | null;
  size?: number;
  fallbackInitials: string;
};

export function Avatar({ uri, size = 40, fallbackInitials }: AvatarProps) {
  // ...
}

// ✅ CORRETO — componente com children (usar PropsWithChildren)
import type { PropsWithChildren } from 'react';

type CardProps = PropsWithChildren<{
  title: string;
  elevated?: boolean;
}>;

export function Card({ title, elevated = false, children }: CardProps) {
  // ...
}

// ❌ ERRADO — React.FC (adiciona children implicitamente no React < 18, verboso)
const Avatar: React.FC<AvatarProps> = ({ uri, size }) => { ... };
```

### 5.3 Tipagem de Navegação (Expo Router)

```ts
// src/types/navigation.ts
import type { Href } from 'expo-router';

// Tipo para parâmetros de rotas específicas
export type RouteParams = {
  '/product/[id]': { id: string };
  '/category/[slug]': { slug: string };
  '/search': { query?: string; filter?: string };
};

// Helper type-safe para navegação
export type AppHref = Href;
```

### 5.4 Enums vs Union Types

> **Regra:** Prefira union types de strings literais sobre `enum`. Unions são mais leves, tree-shakable e idiomáticas em projetos TypeScript modernos.

```ts
// ✅ CORRETO — union type
type OrderStatus = 'pending' | 'processing' | 'shipped' | 'delivered' | 'cancelled';

// ✅ CORRETO — usar `as const` quando precisar do array de valores
const ORDER_STATUSES = ['pending', 'processing', 'shipped', 'delivered', 'cancelled'] as const;
type OrderStatus = (typeof ORDER_STATUSES)[number];

// ❌ EVITAR — enum (gera código JS desnecessário)
enum OrderStatus {
  Pending = 'pending',
  Processing = 'processing',
}
```

### 5.5 Utility Types Úteis

```ts
// Tornar apenas algumas propriedades opcionais
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

// Tipo de resposta da API com timestamp automático
type WithTimestamps<T> = T & {
  createdAt: string;
  updatedAt: string;
};

// Tipo para formulários (todos os campos opcionais + dirty tracking)
type FormState<T> = {
  values: Partial<T>;
  errors: Partial<Record<keyof T, string>>;
  touched: Partial<Record<keyof T, boolean>>;
  isSubmitting: boolean;
};

// Extrair tipo de item de um array
type ArrayItem<T extends readonly unknown[]> = T[number];
```

---

## 6. Organização de Tipos no Projeto

### Estrutura alinhada com `setup-react-native`

```text
src/
├── types/                      # Tipos GLOBAIS e declarações de módulo
│   ├── api.ts                  # PaginatedResponse, ApiError, AsyncState
│   ├── async-state.ts          # AsyncState<T> (discriminated union)
│   ├── navigation.ts           # Tipos de rotas e parâmetros
│   ├── svg.d.ts                # Declaração de SVG
│   └── utils.ts                # Utility types reutilizáveis
│
├── features/
│   └── auth/
│       ├── store/              # Store do domínio auth
│       │   ├── auth.store.ts   # Tipos definidos DENTRO do arquivo da store
│       │   └── index.ts        # Barrel: exporta hooks públicos
│       └── login/
│           ├── types/          # Tipos LOCAIS da feature
│           │   └── index.ts    # LoginRequest, LoginResponse, etc.
│           ├── api/
│           ├── hooks/
│           ├── components/
│           └── screens/
```

### Regras de Organização

| Regra | Descrição |
|---|---|
| **Tipos globais** em `src/types/` | Usados por 2+ features ou camadas |
| **Tipos locais** em `feature/types/` | Específicos de uma feature |
| **Tipos de store** no arquivo da store | Colocalizar `State`, `Actions` e `Store` |
| **Store** em `features/<feature>/store/` | A feature dona do domínio possui a store |
| **Acesso cross-feature** via barrel | Importar store de outra feature via `@features/<feature>` |
| **1 tipo = 1 export** por nome | Evitar re-exportar o mesmo tipo com nomes diferentes |
| **Barrel exports** via `index.ts` | Cada feature exporta seus tipos e hooks públicos via barrel |

---

## Checklist de Verificação

Antes de aprovar um PR, verifique:

### Tipagem Geral
- [ ] `strict: true` habilitado no `tsconfig.json`?
- [ ] Zero usos de `any`? (ex: usar `unknown` + narrowing em vez de `any`)
- [ ] `import type` usado para todas as importações que são apenas tipos?
- [ ] Sem `React.FC` — usar function declarations com props tipadas?
- [ ] Sem `enum` — usar union types de strings literais?

### Generics
- [ ] Hooks genéricos usam constraint `extends` quando aplicável?
- [ ] Funções de API usam generics para tipar `TResponse` e `TBody`?
- [ ] Generics não são usados desnecessariamente (tipo sempre fixo)?

### Discriminated Unions
- [ ] Estados assíncronos **locais** usam `AsyncState<T>` com discriminante `status`?
- [ ] Dados de API sempre usam TanStack Query (`useQuery`/`useMutation`) em vez de `AsyncState<T>`?
- [ ] `switch` sobre discriminated unions tem exhaustiveness check com `never`?
- [ ] Componentes fazem narrowing com `state.status === '...'` antes de acessar props?

### TanStack Query
- [ ] `useQuery<TData, ApiError>()` com generics explícitos?
- [ ] `useMutation<TData, ApiError, TVariables>()` com generics explícitos?
- [ ] `queryFn` converte `Result.isErr()` com `throw result.error` (não `return`)?
- [ ] `onError` recebe `ApiError` tipado sem cast `as`?

### Zustand
- [ ] Store com middleware tem tipagem explícita (`create<StoreType>()`)?
- [ ] Selectors são atômicos (um value por selector) ou usam `useShallow`?
- [ ] Actions estão separadas do state no tipo (`State` + `Actions` = `Store`)?
- [ ] Custom hooks exportados em vez da store bruta?
- [ ] `partialize` usado no `persist` para evitar persistir actions?
