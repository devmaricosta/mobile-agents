---
name: error-handling
description: Padrão centralizado de tratamento de erros para React Native com Expo SDK 54+. Error Boundaries por feature com expo-router e react-error-boundary, padrão Result<T, E> com neverthrow para funções de API, interceptors do Axios para erros HTTP, integração com TanStack Query v5 (throwOnError, useMutation tipado, onError global), e como exibir feedback de erro consistente na UI.
---

# Tratamento de Erros — React Native com Expo (SDK 54+)

Você é um Engenheiro de Software Senior especialista em resiliência e qualidade de software para React Native. Sua responsabilidade é garantir que o projeto trate erros de forma **centralizada, tipada e consistente**, cobrindo todas as camadas da aplicação com as ferramentas corretas, seguindo as melhores práticas da comunidade e compatível com o **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

---

## Quando usar esta skill

- **Novo projeto:** Configurar a infraestrutura de tratamento de erros do zero.
- **Error Boundaries:** Adicionar limites de erro por feature ou por rota do Expo Router.
- **Camada de API:** Implementar o padrão `Result<T, E>` para funções que chamam a API.
- **Axios:** Configurar interceptors para tratar erros HTTP globalmente (401, 403, 5xx).
- **TanStack Query:** Integrar tratamento de erros com `useQuery`/`useMutation` de forma tipada.
- **Feedback na UI:** Exibir mensagens de erro de forma consistente (Toast, Alert, inline).

---

## 1. Visão Geral da Estratégia

```text
┌─────────────────────────────────────────────────────┐
│                    CAMADAS DE ERRO                  │
├─────────────────────────────────────────────────────┤
│  UI (Render)     → Error Boundary (react-error-boundary + expo-router) │
│  Eventos/Async   → try/catch + padrão Result<T, E>  │
│  Rede (HTTP)     → Axios Interceptors               │
│  Feedback        → Toast / Alert / Inline Error     │
└─────────────────────────────────────────────────────┘
```

| Camada | Ferramenta | O que captura |
|---|---|---|
| **Render** | `ErrorBoundary` (class component / `react-error-boundary`) | Erros de renderização JSX |
| **Rota** | `export ErrorBoundary` no arquivo de rota | Erros por tela no Expo Router |
| **API** | Padrão `Result<T, E>` com `neverthrow` | Erros de chamadas assíncronas |
| **HTTP** | Axios interceptors | Erros de rede e status HTTP |
| **UI** | Toast / Alert / Inline | Feedback visual ao usuário |

> **Regra de ouro:** Error Boundaries capturam erros de **renderização**. Erros em event handlers (`onPress`) e código assíncrono (`fetch`, `setTimeout`) **não** são capturados por Error Boundaries — use `try/catch` ou o padrão `Result<T, E>`.

---

## 2. Instalação e Dependências

```bash
npx expo install -- --save-dev react-error-boundary
npx expo install -- --save neverthrow axios
```

> **Nota SDK 54:** Use sempre `npx expo install` para garantir versões compatíveis com o SDK. O `react-error-boundary` é compatível com React 19.1 (usado no SDK 54).

---

## 3. Error Boundaries

### 3.1 Tipos de Error Boundary

| Tipo | Quando usar |
|---|---|
| **Global** (`app/_layout.tsx`) | Captura qualquer erro não tratado no app |
| **Por rota** (`export ErrorBoundary`) | Isola o erro em uma tela específica do Expo Router |
| **Por feature** (componente wrapper) | Isola seções críticas de uma tela (ex: lista de produtos) |

### 3.2 Error Boundary Global — `app/_layout.tsx`

O Expo Router v4 permite exportar um `ErrorBoundary` diretamente de qualquer arquivo de rota. No layout raiz, ele age como o boundary global do app.

```tsx
// src/app/_layout.tsx
import { ErrorBoundaryProps } from 'expo-router';
import { View, Text, Pressable, StyleSheet } from 'react-native';

// Expo Router detecta automaticamente este export e o usa como Error Boundary
export function ErrorBoundary({ error, retry }: ErrorBoundaryProps) {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Algo deu errado</Text>
      <Text style={styles.message}>{error.message}</Text>
      <Pressable style={styles.button} onPress={retry}>
        <Text style={styles.buttonText}>Tentar novamente</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 24,
    backgroundColor: '#fff',
  },
  title: {
    fontSize: 20,
    fontWeight: '700',
    marginBottom: 8,
    color: '#1a1a1a',
  },
  message: {
    fontSize: 14,
    color: '#666',
    textAlign: 'center',
    marginBottom: 24,
  },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  buttonText: {
    color: '#fff',
    fontWeight: '600',
  },
});

export default function RootLayout() {
  // ... layout normal
}
```

### 3.3 Error Boundary por Rota — Expo Router

Exporte `ErrorBoundary` de qualquer arquivo de rota para isolar erros naquela tela:

```tsx
// src/app/(home)/products/index.tsx
import { ErrorBoundaryProps } from 'expo-router';
import { View, Text, Pressable } from 'react-native';

// Isola erros apenas nesta rota — o restante do app continua funcionando
export function ErrorBoundary({ error, retry }: ErrorBoundaryProps) {
  return (
    <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
      <Text>Erro ao carregar produtos</Text>
      <Pressable onPress={retry}>
        <Text>Recarregar</Text>
      </Pressable>
    </View>
  );
}

export default function ProductsScreen() {
  // ... tela normal
}
```

### 3.4 Error Boundary por Feature — `react-error-boundary`

Para isolar seções específicas dentro de uma tela, use o `ErrorBoundary` do `react-error-boundary`:

```tsx
// src/components/feature-error-boundary.tsx
import { ErrorBoundary, FallbackProps } from 'react-error-boundary';
import { View, Text, Pressable, StyleSheet } from 'react-native';

function FeatureFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <View style={styles.container}>
      <Text style={styles.text}>Esta seção está indisponível</Text>
      <Pressable onPress={resetErrorBoundary}>
        <Text style={styles.retry}>Tentar novamente</Text>
      </Pressable>
    </View>
  );
}

interface FeatureErrorBoundaryProps {
  children: React.ReactNode;
  onError?: (error: Error) => void;
}

export function FeatureErrorBoundary({ children, onError }: FeatureErrorBoundaryProps) {
  return (
    <ErrorBoundary
      FallbackComponent={FeatureFallback}
      onError={(error, info) => {
        // Loga no serviço de monitoramento (ex: Sentry)
        console.error('[FeatureErrorBoundary]', error, info.componentStack);
        onError?.(error);
      }}
    >
      {children}
    </ErrorBoundary>
  );
}

const styles = StyleSheet.create({
  container: {
    padding: 16,
    alignItems: 'center',
    backgroundColor: '#FFF3F3',
    borderRadius: 8,
    margin: 8,
  },
  text: { color: '#CC0000', marginBottom: 8 },
  retry: { color: '#007AFF', fontWeight: '600' },
});
```

```tsx
// Uso em uma tela
import { FeatureErrorBoundary } from '@components/feature-error-boundary';
import { ProductList } from '@features/products';

export default function HomeScreen() {
  return (
    <View>
      <HeroSection />
      {/* Isola apenas a lista de produtos — se quebrar, o resto da tela continua */}
      <FeatureErrorBoundary>
        <ProductList />
      </FeatureErrorBoundary>
      <Footer />
    </View>
  );
}
```

### 3.5 Quando cada abordagem usar

| Cenário | Abordagem |
|---|---|
| Erro catastrófico que quebra o app inteiro | `ErrorBoundary` no `_layout.tsx` |
| Erro que quebra uma tela específica | `export ErrorBoundary` no arquivo de rota |
| Erro em uma seção/widget dentro de uma tela | `<FeatureErrorBoundary>` wrapper |
| Erro em event handler (`onPress`) | `try/catch` + padrão `Result<T, E>` |
| Erro em chamada de API | Padrão `Result<T, E>` + Axios interceptor |

---

## 4. Padrão `Result<T, E>` para Funções de API

O padrão `Result<T, E>`, popularizado pelo Rust, torna os erros **explícitos no tipo** da função. Em vez de lançar exceções, a função retorna um tipo que representa sucesso (`Ok`) ou falha (`Err`). A biblioteca `neverthrow` implementa este padrão para TypeScript.

### 4.1 Definindo os Tipos de Erro de Domínio

```ts
// src/services/api/errors.ts

/** Erros tipados da camada de API */
export type ApiError =
  | { type: 'NETWORK_ERROR'; message: string }
  | { type: 'UNAUTHORIZED'; message: string }
  | { type: 'FORBIDDEN'; message: string }
  | { type: 'NOT_FOUND'; resource: string }
  | { type: 'VALIDATION_ERROR'; fields: Record<string, string[]> }
  | { type: 'SERVER_ERROR'; statusCode: number; message: string }
  | { type: 'UNKNOWN'; message: string };

/** Helpers para criar erros tipados */
export const ApiErrors = {
  network: (message = 'Sem conexão com a internet'): ApiError => ({
    type: 'NETWORK_ERROR',
    message,
  }),
  unauthorized: (): ApiError => ({
    type: 'UNAUTHORIZED',
    message: 'Sessão expirada. Faça login novamente.',
  }),
  forbidden: (): ApiError => ({
    type: 'FORBIDDEN',
    message: 'Você não tem permissão para realizar esta ação.',
  }),
  notFound: (resource: string): ApiError => ({
    type: 'NOT_FOUND',
    resource,
  }),
  validation: (fields: Record<string, string[]>): ApiError => ({
    type: 'VALIDATION_ERROR',
    fields,
  }),
  server: (statusCode: number, message = 'Erro interno do servidor'): ApiError => ({
    type: 'SERVER_ERROR',
    statusCode,
    message,
  }),
  unknown: (message = 'Ocorreu um erro inesperado'): ApiError => ({
    type: 'UNKNOWN',
    message,
  }),
} as const;
```

### 4.2 Wrapper Base para Chamadas de API

```ts
// src/services/api/result.ts
import { Result, ResultAsync, ok, err } from 'neverthrow';
import { AxiosError } from 'axios';
import { ApiError, ApiErrors } from './errors';

export type ApiResult<T> = Result<T, ApiError>;
export type ApiResultAsync<T> = ResultAsync<T, ApiError>;

/**
 * Converte uma Promise de Axios em ResultAsync<T, ApiError>.
 * Use este wrapper em TODAS as funções de serviço de API.
 */
export function fromAxiosPromise<T>(promise: Promise<T>): ApiResultAsync<T> {
  return ResultAsync.fromPromise(promise, (error) => {
    if (error instanceof AxiosError) {
      return mapAxiosError(error);
    }
    return ApiErrors.unknown(String(error));
  });
}

function mapAxiosError(error: AxiosError<{ message?: string; errors?: Record<string, string[]> }>): ApiError {
  if (!error.response) {
    // Sem resposta = erro de rede
    return ApiErrors.network();
  }

  const { status, data } = error.response;

  switch (status) {
    case 401:
      return ApiErrors.unauthorized();
    case 403:
      return ApiErrors.forbidden();
    case 404:
      return ApiErrors.notFound(error.config?.url ?? 'recurso');
    case 422:
      return ApiErrors.validation(data?.errors ?? {});
    default:
      if (status >= 500) {
        return ApiErrors.server(status, data?.message);
      }
      return ApiErrors.unknown(data?.message ?? error.message);
  }
}
```

### 4.3 Funções de Serviço com `Result<T, E>`

```ts
// src/features/auth/services/auth.service.ts
import { api } from '@services/api/client'; // instância Axios
import { fromAxiosPromise, ApiResultAsync } from '@services/api/result';
import type { User } from '../types';

interface LoginPayload {
  email: string;
  password: string;
}

interface LoginResponse {
  user: User;
  token: string;
}

/** Retorna Result — nunca lança exceção */
export function login(payload: LoginPayload): ApiResultAsync<LoginResponse> {
  return fromAxiosPromise(
    api.post<LoginResponse>('/auth/login', payload).then((r) => r.data),
  );
}

export function getProfile(): ApiResultAsync<User> {
  return fromAxiosPromise(
    api.get<User>('/auth/me').then((r) => r.data),
  );
}
```

### 4.4 Consumindo `Result<T, E>` em Hooks com `useMutation`

> **Regra:** Para qualquer operação que chama a API, use `useMutation` (ou `useQuery`) do TanStack Query. O padrão `Result<T, E>` vive **dentro do `mutationFn`** — converta com `if (result.isErr()) throw result.error` para que o TanStack Query capture o erro tipado.

```ts
// src/features/auth/mutations/use-login.ts
import { useMutation } from '@tanstack/react-query';
import type { UseMutationResult } from '@tanstack/react-query';
import { useQueryClient } from '@tanstack/react-query';
import { login } from '../services/auth.service';
import { useAuthStore } from '../store/auth.store';
import type { ApiError } from '@services/api/errors';
import type { LoginPayload, LoginResponse } from '../types';

export function useLogin(): UseMutationResult<LoginResponse, ApiError, LoginPayload> {
  const queryClient = useQueryClient();
  const setAuth = useAuthStore((s) => s.setAuth);

  return useMutation<LoginResponse, ApiError, LoginPayload>({
    mutationFn: async (credentials) => {
      const result = await login(credentials);
      // Converte Result → throw para o TanStack Query capturar como `error` tipado
      if (result.isErr()) throw result.error;
      return result.value;
    },
    onSuccess: ({ user, token }) => {
      setAuth(user, token);
      // Reseta o cache após login (dados do usuário anterior)
      queryClient.resetQueries();
    },
    onError: (error) => {
      // `error` já é ApiError — sem `unknown`, sem cast
      if (error.type === 'UNAUTHORIZED') {
        // Credenciais inválidas — não precisa logar
      } else {
        console.error('[useLogin] Erro inesperado:', error.type, error.message);
      }
    },
  });
}
```

**Uso no componente:**
```tsx
const { mutate: handleLogin, isPending, isError, error } = useLogin();

// Renderização
{isPending && <ActivityIndicator />}
{isError && <ErrorMessage error={error} />}
<Button onPress={() => handleLogin({ email, password })} loading={isPending} />
```

### 4.5 Encadeando Operações com `andThen`

```ts
// Exemplo: buscar perfil após login
import { login, getProfile } from '../services/auth.service';

async function loginAndFetchProfile(email: string, password: string) {
  const result = await login({ email, password })
    .andThen(({ token }) => {
      // Só executa se login foi bem-sucedido
      setTokenInStorage(token);
      return getProfile();
    });

  return result; // Result<User, ApiError>
}
```

---

## 5. Interceptors do Axios para Erros HTTP

### 5.1 Criando a Instância Axios Centralizada

```ts
// src/services/api/client.ts
import axios, { AxiosInstance } from 'axios';
import { useAuthStore } from '@features/auth';
import { router } from 'expo-router';

export const api: AxiosInstance = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 15_000,
  headers: {
    'Content-Type': 'application/json',
    Accept: 'application/json',
  },
});

// ─── Request Interceptor ──────────────────────────────────────────────────────
// Injeta o token de autenticação em todas as requisições
api.interceptors.request.use(
  (config) => {
    const token = useAuthStore.getState().token;
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error),
);

// ─── Response Interceptor ─────────────────────────────────────────────────────
// Trata erros HTTP globalmente antes de chegarem nos serviços
api.interceptors.response.use(
  // Resposta bem-sucedida — passa direto
  (response) => response,

  // Erro de resposta — tratamento centralizado
  async (error) => {
    const originalRequest = error.config;

    // 401 — Token expirado: tenta renovar uma vez
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      const refreshed = await tryRefreshToken();
      if (refreshed) {
        // Reenvia a requisição original com o novo token
        originalRequest.headers.Authorization = `Bearer ${useAuthStore.getState().token}`;
        return api(originalRequest);
      }

      // Refresh falhou — desloga e redireciona para login
      useAuthStore.getState().logout();
      router.replace('/(auth)/login');
    }

    // 403 — Sem permissão: redireciona para tela de acesso negado
    if (error.response?.status === 403) {
      router.replace('/(error)/forbidden');
    }

    // Repassa o erro para ser tratado pelo `fromAxiosPromise`
    return Promise.reject(error);
  },
);

async function tryRefreshToken(): Promise<boolean> {
  try {
    const refreshToken = useAuthStore.getState().refreshToken;
    if (!refreshToken) return false;

    const { data } = await axios.post(`${process.env.EXPO_PUBLIC_API_URL}/auth/refresh`, {
      refreshToken,
    });

    useAuthStore.getState().setToken(data.token, data.refreshToken);
    return true;
  } catch {
    return false;
  }
}
```

### 5.2 Tabela de Tratamento por Status HTTP

| Status | Ação no Interceptor | Feedback na UI |
|---|---|---|
| `401` (primeira vez) | Tenta renovar o token | Nenhum (transparente) |
| `401` (após retry) | Desloga + redireciona para login | Toast: "Sessão expirada" |
| `403` | Redireciona para tela de acesso negado | Tela de erro dedicada |
| `404` | Repassa para o `Result` | Mensagem inline na tela |
| `422` | Repassa para o `Result` | Erros de validação por campo |
| `429` | Repassa para o `Result` | Toast: "Muitas tentativas" |
| `5xx` | Repassa para o `Result` | Toast: "Erro no servidor" |
| Sem resposta | Repassa para o `Result` | Toast: "Sem conexão" |

### 5.3 Configurando Timeout e Retry

```ts
// src/services/api/client.ts — adicionar ao interceptor de resposta

// Retry automático para erros de rede (sem resposta do servidor)
api.interceptors.response.use(undefined, async (error) => {
  const config = error.config;

  // Só faz retry se não houver resposta (erro de rede) e não for uma tentativa de retry
  if (!error.response && !config._retryCount) {
    config._retryCount = 1;

    // Aguarda 1 segundo antes de tentar novamente
    await new Promise((resolve) => setTimeout(resolve, 1000));
    return api(config);
  }

  return Promise.reject(error);
});
```

---

## 6. Feedback de Erro Consistente na UI

### 6.1 Hierarquia de Feedback

| Tipo de Erro | Mecanismo de Feedback | Quando usar |
|---|---|---|
| **Erro crítico de render** | Tela de erro (Error Boundary) | App/tela não consegue renderizar |
| **Erro de ação do usuário** | Mensagem inline no formulário | Validação, campos inválidos |
| **Erro de operação** | Toast / Snackbar | Ação falhou (salvar, deletar) |
| **Erro de carregamento** | Estado vazio com retry | Lista/tela não carregou dados |
| **Erro de rede** | Toast + botão retry | Sem conexão |

### 6.2 Hook Centralizado de Feedback de Erro

```ts
// src/hooks/use-error-feedback.ts
import { Alert } from 'react-native';
import type { ApiError } from '@services/api/errors';

/**
 * Converte um ApiError em mensagem amigável para o usuário.
 * Use este hook para exibir feedback de erro de forma consistente.
 */
export function useErrorFeedback() {
  function getErrorMessage(error: ApiError): string {
    switch (error.type) {
      case 'NETWORK_ERROR':
        return 'Sem conexão com a internet. Verifique sua rede e tente novamente.';
      case 'UNAUTHORIZED':
        return 'Sua sessão expirou. Faça login novamente.';
      case 'FORBIDDEN':
        return 'Você não tem permissão para realizar esta ação.';
      case 'NOT_FOUND':
        return `${error.resource} não encontrado.`;
      case 'VALIDATION_ERROR':
        return 'Verifique os campos e tente novamente.';
      case 'SERVER_ERROR':
        return 'Erro no servidor. Tente novamente em alguns instantes.';
      case 'UNKNOWN':
      default:
        return error.message ?? 'Ocorreu um erro inesperado.';
    }
  }

  function showErrorAlert(error: ApiError, title = 'Erro') {
    Alert.alert(title, getErrorMessage(error), [{ text: 'OK' }]);
  }

  return { getErrorMessage, showErrorAlert };
}
```

### 6.3 Componente de Erro Inline (Formulários)

```tsx
// src/components/form-error.tsx
import { Text, StyleSheet } from 'react-native';

interface FormErrorProps {
  message?: string | null;
}

export function FormError({ message }: FormErrorProps) {
  if (!message) return null;

  return <Text style={styles.error}>{message}</Text>;
}

const styles = StyleSheet.create({
  error: {
    color: '#CC0000',
    fontSize: 12,
    marginTop: 4,
    marginLeft: 2,
  },
});
```

```tsx
// Uso em formulário
import { useLogin } from '@features/auth';
import { FormError } from '@components/form-error';

export function LoginForm() {
  const { handleLogin, isLoading, error } = useLogin();
  const { getErrorMessage } = useErrorFeedback();

  return (
    <View>
      <TextInput placeholder="E-mail" onChangeText={setEmail} />
      <TextInput placeholder="Senha" secureTextEntry onChangeText={setPassword} />

      {/* Exibe erro de validação inline */}
      {error?.type === 'VALIDATION_ERROR' && (
        <FormError message={error.fields.email?.[0]} />
      )}

      {/* Exibe erro geral abaixo do formulário */}
      {error && error.type !== 'VALIDATION_ERROR' && (
        <FormError message={getErrorMessage(error)} />
      )}

      <Button onPress={() => handleLogin(email, password)} loading={isLoading}>
        Entrar
      </Button>
    </View>
  );
}
```

### 6.4 Componente de Estado Vazio com Retry

```tsx
// src/components/error-state.tsx
import { View, Text, Pressable, StyleSheet } from 'react-native';
import type { ApiError } from '@services/api/errors';
import { useErrorFeedback } from '@hooks/use-error-feedback';

interface ErrorStateProps {
  error: ApiError;
  onRetry?: () => void;
}

export function ErrorState({ error, onRetry }: ErrorStateProps) {
  const { getErrorMessage } = useErrorFeedback();

  return (
    <View style={styles.container}>
      <Text style={styles.emoji}>⚠️</Text>
      <Text style={styles.message}>{getErrorMessage(error)}</Text>
      {onRetry && (
        <Pressable style={styles.button} onPress={onRetry}>
          <Text style={styles.buttonText}>Tentar novamente</Text>
        </Pressable>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 32,
  },
  emoji: { fontSize: 48, marginBottom: 16 },
  message: {
    fontSize: 16,
    color: '#555',
    textAlign: 'center',
    marginBottom: 24,
    lineHeight: 24,
  },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  buttonText: { color: '#fff', fontWeight: '600' },
});
```

```tsx
// Uso em uma tela com carregamento de dados
import { useProducts } from '@features/products';
import { ErrorState } from '@components/error-state';

export default function ProductsScreen() {
  const { products, isLoading, error, refetch } = useProducts();

  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorState error={error} onRetry={refetch} />;

  return <ProductList products={products} />;
}
```

---

## 7. Organização dos Arquivos

```text
src/
├── services/
│   └── api/
│       ├── client.ts          ← Instância Axios + interceptors
│       ├── errors.ts          ← Tipos ApiError + helpers ApiErrors
│       └── result.ts          ← fromAxiosPromise + tipos ApiResult
├── hooks/
│   └── use-error-feedback.ts  ← Hook de mensagens de erro
├── components/
│   ├── feature-error-boundary.tsx  ← Wrapper de Error Boundary reutilizável
│   ├── error-state.tsx             ← Estado de erro com retry
│   └── form-error.tsx              ← Erro inline para formulários
├── features/
│   └── auth/
│       ├── services/
│       │   └── auth.service.ts     ← Usa fromAxiosPromise → Result<T, E>
│       └── hooks/
│           └── use-login.ts        ← Consome Result com .match()
└── app/
    ├── _layout.tsx                 ← export ErrorBoundary (global)
    └── (home)/
        └── products/
            └── index.tsx           ← export ErrorBoundary (por rota)
```

---

## 8. Padrões e Anti-Padrões

### ✅ Faça

```ts
// ✅ Funções de serviço retornam Result — nunca lançam exceção
export function login(payload: LoginPayload): ApiResultAsync<LoginResponse> {
  return fromAxiosPromise(api.post('/auth/login', payload).then((r) => r.data));
}

// ✅ Consuma Result com .match() para tratar sucesso e erro explicitamente
result.match(
  (data) => setData(data),
  (error) => setError(error), // error é ApiError tipado
);

// ✅ Use ErrorBoundary por rota no Expo Router para isolar falhas
export function ErrorBoundary({ error, retry }: ErrorBoundaryProps) {
  return <ErrorScreen error={error} onRetry={retry} />;
}

// ✅ Use FeatureErrorBoundary para isolar seções críticas de uma tela
<FeatureErrorBoundary>
  <ProductList />
</FeatureErrorBoundary>

// ✅ Centralize a instância Axios — nunca crie axios.create() em múltiplos lugares
import { api } from '@services/api/client';
```

### ❌ Evite

```ts
// ❌ Não lance exceções em funções de serviço
async function login(payload) {
  const { data } = await api.post('/auth/login', payload); // pode lançar!
  return data;
}

// ❌ Não use catch genérico sem tipar o erro
try {
  await login(payload);
} catch (e) {
  console.error(e); // e é `unknown` — sem tipo, sem tratamento adequado
}

// ❌ Não crie múltiplas instâncias Axios com configurações diferentes
const api1 = axios.create({ baseURL: '...' });
const api2 = axios.create({ baseURL: '...' }); // interceptors não compartilhados!

// ❌ Não exiba mensagens técnicas ao usuário
Alert.alert('Erro', error.message); // "Request failed with status code 401"

// ❌ Não ignore o resultado de uma operação que pode falhar
await login(payload); // e se falhar? o usuário não sabe
```

---

## 9. Integração com Monitoramento (Sentry)

Para produção, integre o Sentry para capturar erros automaticamente:

```bash
npx expo install @sentry/react-native
```

```ts
// src/app/_layout.tsx
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  environment: process.env.EXPO_PUBLIC_ENV, // 'development' | 'staging' | 'production'
  enabled: process.env.EXPO_PUBLIC_ENV !== 'development',
});

// No ErrorBoundary global — loga o erro no Sentry
export function ErrorBoundary({ error, retry }: ErrorBoundaryProps) {
  // Sentry captura automaticamente via seu próprio ErrorBoundary,
  // mas você pode logar manualmente também:
  Sentry.captureException(error);

  return <ErrorScreen error={error} onRetry={retry} />;
}
```

```ts
// No FeatureErrorBoundary — loga com contexto adicional
<ErrorBoundary
  FallbackComponent={FeatureFallback}
  onError={(error, info) => {
    Sentry.withScope((scope) => {
      scope.setExtra('componentStack', info.componentStack);
      Sentry.captureException(error);
    });
  }}
>
  {children}
</ErrorBoundary>
```

---

## 10. TanStack Query v5 — Integração com Tratamento de Erros

O TanStack Query v5 tem suporte nativo a erros tipados e integra-se perfeitamente com o padrão `Result<T, E>` e os Error Boundaries desta skill.

### 10.1 Conversão de `Result<T, E>` para TanStack Query

O `queryFn` deve retornar os dados em caso de sucesso ou **lançar** o erro para o TanStack Query capturar. A conversão é simples:

```ts
// src/features/products/hooks/use-products.ts
import { useQuery } from '@tanstack/react-query';
import { productKeys } from '@constants/query-keys';
import { getProducts } from '../services/products.service';
import type { ApiError } from '@services/api/errors';
import type { Product } from '../types';

export function useProducts() {
  return useQuery<Product[], ApiError>({
    queryKey: productKeys.lists(),
    queryFn: async () => {
      const result = await getProducts();
      // Result.isErr() → throw → TanStack Query captura como `error`
      if (result.isErr()) throw result.error;
      return result.value; // Result.isOk() → retorna os dados
    },
    // Retry inteligente por tipo de ApiError
    retry: (failureCount, error) => {
      // Não faz retry em erros de autenticação/permissão
      if (error.type === 'UNAUTHORIZED' || error.type === 'FORBIDDEN') return false;
      // Não faz retry em erros de validação (4xx client errors)
      if (error.type === 'VALIDATION_ERROR' || error.type === 'NOT_FOUND') return false;
      return failureCount < 2;
    },
  });
}
```

```tsx
// Uso na tela — error é ApiError tipado, não `unknown`
export default function ProductsScreen() {
  const { data: products, isLoading, error, refetch } = useProducts();

  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorState error={error} onRetry={refetch} />;

  return <ProductList products={products} />;
}
```

### 10.2 `throwOnError` — Propagar Erros para Error Boundaries

Use `throwOnError` para propagar erros de servidor (5xx) para o Error Boundary mais próximo, enquanto erros de cliente (4xx) são tratados localmente:

```ts
export function useProducts() {
  return useQuery<Product[], ApiError>({
    queryKey: productKeys.lists(),
    queryFn: async () => {
      const result = await getProducts();
      if (result.isErr()) throw result.error;
      return result.value;
    },
    // Erros 5xx vão para o Error Boundary — erros 4xx são tratados localmente
    throwOnError: (error) => error.type === 'SERVER_ERROR',
  });
}
```

### 10.3 `useMutation` com Erros Tipados

```ts
// src/features/auth/hooks/use-login.ts
import { useMutation } from '@tanstack/react-query';
import { useRouter } from 'expo-router';
import { useAuthStore } from '@features/auth';
import { login } from '../services/auth.service';
import type { ApiError } from '@services/api/errors';
import type { LoginPayload, LoginResponse } from '../types';

export function useLogin() {
  const router = useRouter();
  const setAuth = useAuthStore((s) => s.setAuth);

  return useMutation<LoginResponse, ApiError, LoginPayload>({
    mutationFn: async (payload) => {
      const result = await login(payload);
      if (result.isErr()) throw result.error; // ApiError tipado
      return result.value;
    },
    onSuccess: ({ user, token }) => {
      setAuth(user, token);
      router.replace('/(home)');
    },
    // error é ApiError — sem `unknown`, sem cast
    onError: (error) => {
      // Erros de validação são tratados no componente via mutation.error
      // Erros inesperados podem ser logados aqui
      if (error.type === 'UNKNOWN' || error.type === 'SERVER_ERROR') {
        console.error('[useLogin] Erro inesperado:', error);
      }
    },
  });
}
```

```tsx
// Uso no componente — mutation.error é ApiError tipado
export function LoginForm() {
  const { mutate: login, isPending, error } = useLogin();
  const { getErrorMessage } = useErrorFeedback();

  return (
    <View>
      <TextInput placeholder="E-mail" onChangeText={setEmail} />

      {/* Erro de validação inline por campo */}
      {error?.type === 'VALIDATION_ERROR' && (
        <FormError message={error.fields.email?.[0]} />
      )}

      {/* Erro geral (rede, servidor, etc.) */}
      {error && error.type !== 'VALIDATION_ERROR' && (
        <FormError message={getErrorMessage(error)} />
      )}

      <Button
        onPress={() => login({ email, password })}
        loading={isPending}
      >
        Entrar
      </Button>
    </View>
  );
}
```

### 10.4 `onError` Global no `QueryClient`

Configure um handler global para logar todos os erros de query/mutation automaticamente:

```ts
// src/services/query-client.ts
import { QueryClient, MutationCache, QueryCache } from '@tanstack/react-query';
import type { ApiError } from './api/errors';

export const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Loga todos os erros de query no Sentry (ou console em dev)
      const apiError = error as ApiError;
      console.error(`[QueryCache] Erro em ${String(query.queryKey[0])}:`, apiError);
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      // Loga todos os erros de mutation
      const apiError = error as ApiError;
      console.error('[MutationCache] Erro:', apiError);
    },
  }),
  defaultOptions: {
    queries: {
      staleTime: 2 * 60 * 1000,
      gcTime: 5 * 60 * 1000,
      retry: 2,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30_000),
      refetchOnReconnect: true,
      refetchOnWindowFocus: false,
    },
    mutations: { retry: false },
  },
});
```

### 10.5 Tabela de Decisão: Onde Tratar o Erro

| Tipo de Erro | Onde tratar | Mecanismo |
|---|---|---|
| Erro de render (JSX) | Error Boundary | `export ErrorBoundary` / `FeatureErrorBoundary` |
| Erro de servidor (5xx) | Error Boundary | `throwOnError: (e) => e.type === 'SERVER_ERROR'` |
| Erro de autenticação (401) | Axios interceptor | Redirect automático para login |
| Erro de permissão (403) | Axios interceptor | Redirect para tela de acesso negado |
| Erro de validação (422) | Componente local | `mutation.error?.fields` inline no formulário |
| Erro de rede | Componente local | `<ErrorState error={error} onRetry={refetch} />` |
| Erro inesperado | `onError` global | Log no Sentry + `<ErrorState>` |
