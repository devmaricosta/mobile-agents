---
name: offline-first
description: Estratégia offline-first para React Native com Expo SDK 54 — persistência do cache do TanStack Query v5 com AsyncStorage, fila automática de mutations pendentes com resumePausedMutations, detecção de conectividade com @react-native-community/netinfo integrado ao onlineManager do TanStack Query, e feedback visual de status de rede.
---

# Estratégia Offline-First — React Native com Expo (SDK 54+)

Você é um Engenheiro de Software Senior especialista em experiências offline-first em React Native. Sua responsabilidade é garantir que o app funcione corretamente sem internet, sincronize automaticamente quando a conexão for restaurada, e forneça feedback visual claro ao usuário sobre o status da rede — tudo em harmonia com os padrões de **TanStack Query v5**, **Zustand** e **AsyncStorage** já definidos no projeto, compatíveis com **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

---

## Quando usar esta skill

- **Configuração inicial:** Instalar e configurar persistência do cache e detecção de rede.
- **Mutations offline:** Garantir que ações do usuário (criar, editar, deletar) sejam enfileiradas offline e sincronizadas ao reconectar.
- **Queries offline:** Exibir dados em cache quando não há conexão, sem tela em branco.
- **Feedback visual:** Implementar banner/indicador de status de rede.
- **Code review:** Validar se o fluxo offline segue as convenções do projeto.

---

## 1. Visão Geral da Arquitetura Offline-First

```text
┌─────────────────────────────────────────────────────────┐
│                     CAMADA DE REDE                       │
│  @react-native-community/netinfo                         │
│  └── onlineManager (TanStack Query) ← sincronizado      │
└────────────────────────┬────────────────────────────────┘
                         │ isOnline / isOffline
┌────────────────────────▼────────────────────────────────┐
│                  TANSTACK QUERY v5                        │
│  PersistQueryClientProvider                              │
│  ├── QueryCache (em memória) ←→ AsyncStorage (disco)    │
│  └── MutationQueue (pausada offline → resume online)    │
└────────────────────────┬────────────────────────────────┘
                         │ dados / status
┌────────────────────────▼────────────────────────────────┐
│               FEATURES (hooks + telas)                   │
│  useQuery → exibe cache local enquanto offline           │
│  useMutation → enfileira, sincroniza ao reconectar       │
│  useNetworkStatus → fornece estado para feedback visual  │
└─────────────────────────────────────────────────────────┘
```

**Princípio fundamental:** O TanStack Query gerencia queries e mutations. O NetInfo é a única fonte de verdade sobre conectividade. O AsyncStorage persiste o cache entre sessões do app. O Zustand **não** é usado para cachear dados de API.

---

## 2. Instalação

```bash
# Persister: pontes entre TanStack Query e AsyncStorage
npx expo install @tanstack/query-async-storage-persister
npx expo install @tanstack/react-query-persist-client

# AsyncStorage (provavelmente já instalado — usado também pelo Zustand persist)
npx expo install @react-native-async-storage/async-storage

# Detecção de conectividade
npx expo install @react-native-community/netinfo
```

> **SDK 54:** O `@react-native-community/netinfo` é compatível com a Nova Arquitetura (Fabric + Hermes) habilitada por padrão no SDK 54. Use sempre `npx expo install` para garantir a versão compatível com o SDK.

---

## 3. Configuração do QueryClient com suporte offline

O `QueryClient` configurado em `src/services/query-client.ts` (já definido na skill `react-query-patterns`) **precisa** de dois ajustes para suportar offline:

```ts
// src/services/query-client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 2 * 60 * 1000,      // 2 min — igual ao padrão do projeto
      gcTime: 24 * 60 * 60 * 1000,   // ⚠️ AUMENTAR para 24h — cache sobrevive offline

      retry: 2,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30_000),

      refetchOnReconnect: true,       // ✅ refaz fetch ao reconectar
      refetchOnWindowFocus: false,    // irrelevante em mobile
    },
    mutations: {
      retry: false,                   // mutations não fazem retry automático
    },
  },
});
```

> **Por que `gcTime: 24h`?** Com o persister do AsyncStorage, os dados persistidos expiram de acordo com o `gcTime` do QueryClient. Em cenários offline, o usuário pode ficar horas sem internet — aumentar o `gcTime` garante que os dados em cache ainda estejam disponíveis quando ele abrir o app novamente.

---

## 4. Configuração do Persister (AsyncStorage)

```ts
// src/services/query-persister.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';

export const asyncStoragePersister = createAsyncStoragePersister({
  storage: AsyncStorage,

  // Chave única no AsyncStorage — não conflita com as stores do Zustand
  key: 'TANSTACK_QUERY_OFFLINE_CACHE',

  // Throttle: serializa para AsyncStorage no máximo a cada 1s (evita I/O excessivo)
  throttleTime: 1000,
});
```

> **Separação de responsabilidades no AsyncStorage:**
>
> | Chave | Dono | Conteúdo |
> |---|---|---|
> | `auth-storage` | Zustand (`zustand-architecture`) | Token, user profile |
> | `cart-storage` | Zustand (`zustand-architecture`) | Itens do carrinho |
> | `TANSTACK_QUERY_OFFLINE_CACHE` | TanStack Query (esta skill) | Cache de queries |
>
> Cada sistema usa sua própria chave — zero conflito.

---

## 5. Configuração do `onlineManager` com NetInfo

O TanStack Query tem um `onlineManager` interno que controla se mutations são pausadas ou executadas. Por padrão, ele usa `navigator.onLine` (API web). Em React Native, é necessário substituí-lo pelo NetInfo:

```ts
// src/services/online-manager.ts
import { onlineManager } from '@tanstack/react-query';
import NetInfo from '@react-native-community/netinfo';

/**
 * Conecta o onlineManager do TanStack Query ao NetInfo.
 * Deve ser chamado UMA VEZ, antes do PersistQueryClientProvider.
 *
 * Efeito: mutations são automaticamente pausadas offline e
 * retomadas (resumePausedMutations) ao reconectar.
 */
export function setupOnlineManager(): () => void {
  return onlineManager.setEventListener((setOnline) => {
    // addEventListener retorna uma função de cleanup
    return NetInfo.addEventListener((state) => {
      const isConnected = state.isConnected ?? false;
      const isInternetReachable = state.isInternetReachable ?? false;

      // Considera online apenas se conectado E com internet alcançável
      setOnline(isConnected && isInternetReachable);
    });
  });
}
```

> **Por que `isConnected && isInternetReachable`?** Em redes Wi-Fi sem saída para a internet (ex: roteador sem conexão externa), `isConnected` é `true` mas `isInternetReachable` é `false`. Usar ambos evita falsas detecções de "online".

---

## 6. Provider no Layout Raiz

Substitui o `QueryClientProvider` padrão pelo `PersistQueryClientProvider`:

```tsx
// src/app/_layout.tsx
import { useEffect } from 'react';
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { queryClient } from '@services/query-client';
import { asyncStoragePersister } from '@services/query-persister';
import { setupOnlineManager } from '@services/online-manager';

export default function RootLayout() {
  // Inicializa o onlineManager UMA VEZ, na montagem do layout raiz
  useEffect(() => {
    const cleanup = setupOnlineManager();
    return cleanup; // desregistra o listener ao desmontar (dev/HMR)
  }, []);

  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: asyncStoragePersister,

        // Descarta o cache persistido se tiver mais de 24h
        maxAge: 24 * 60 * 60 * 1000,

        // Desserializa / serializa o cache (padrão JSON é suficiente)
        dehydrateOptions: {
          shouldDehydrateMutation: (mutation) =>
            // Persiste apenas mutations que ainda não foram sincronizadas
            mutation.state.status === 'pending',
        },
      }}
      onSuccess={() => {
        // Chamado após o cache ser restaurado do AsyncStorage.
        // Resume imediatamente qualquer mutation enfileirada da sessão anterior.
        queryClient.resumePausedMutations().then(() => {
          // Invalida todas as queries para garantir dados frescos após sync
          queryClient.invalidateQueries();
        });
      }}
    >
      {/* Zustand não precisa de Provider */}
      <Stack />
    </PersistQueryClientProvider>
  );
}
```

> **Por que `onSuccess` + `resumePausedMutations`?** O TanStack Query serializa mutations pendentes no AsyncStorage. Quando o app abre novamente, o `onSuccess` é o ponto certo para reprocessar essas mutations — após ter certeza que o cache foi reidratado e que há conexão (controlada pelo `onlineManager`).

---

## 7. Fila de Mutations Pendentes — `mutationDefaults`

O TanStack Query serializa mutations pendentes no AsyncStorage, mas **não serializa funções**. Por isso, é obrigatório registrar `mutationDefaults` no `QueryClient` para que o `mutationFn` seja re-associado ao reabrir o app.

### 7.1 Padrão `mutationDefaults` por Feature

```ts
// src/services/query-client.ts — após criar o queryClient
import { createProduct } from '@features/products/services/products.service';

// Cada mutation que precisa sobreviver offline DEVE ter um mutationKey
// e um mutationDefault com o mutationFn registrado.
queryClient.setMutationDefaults(['products', 'create'], {
  mutationFn: async (payload: CreateProductPayload) => {
    const result = await createProduct(payload);
    if (result.isErr()) throw result.error;
    return result.value;
  },
});

queryClient.setMutationDefaults(['products', 'update'], {
  mutationFn: async ({ id, payload }: { id: string; payload: UpdateProductPayload }) => {
    const result = await updateProduct(id, payload);
    if (result.isErr()) throw result.error;
    return result.value;
  },
});
```

### 7.2 Mutation com `mutationKey` (obrigatório para offline)

Toda mutation que deve ser enfileirada offline **precisa declarar `mutationKey`**:

```ts
// src/features/products/mutations/use-create-product.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import type { ApiError } from '@services/api/errors';
import { createProduct } from '../services/products.service';
import type { CreateProductPayload, Product } from '../types/products.types';
import { productKeys } from '../queries/query-keys';

export function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation<Product, ApiError, CreateProductPayload>({
    // ✅ mutationKey obrigatório para mutations offline
    // Deve coincidir com o registrado em setMutationDefaults
    mutationKey: ['products', 'create'],

    // mutationFn aqui é usado online (sessão ativa)
    // O setMutationDefaults registra o mesmo fn para reuso após restart
    mutationFn: async (payload) => {
      const result = await createProduct(payload);
      if (result.isErr()) throw result.error;
      return result.value;
    },

    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: productKeys.lists() });
    },

    onError: (error) => {
      if (error.type === 'UNKNOWN') {
        console.error('[useCreateProduct]', error);
      }
    },
  });
}
```

### 7.3 Fluxo da Fila Offline

```text
Usuário cria produto SEM INTERNET
       ↓
useMutation → TanStack Query detecta offline (onlineManager)
       ↓
mutation fica com status "paused" (não "error")
       ↓
mutation é serializada no AsyncStorage (via PersistQueryClientProvider)
       ↓
Usuário reconecta / reabre o app
       ↓
onlineManager sinaliza online → resumePausedMutations()
       ↓
mutation é re-executada automaticamente com o mesmo payload
       ↓
onSuccess → invalidateQueries → UI atualizada
```

> **Nota sobre `status: 'paused'`:** Quando offline, o TanStack Query pausa a mutation (não marca como erro). O hook `useMutation` expõe `isPaused: boolean` para detectar esse estado e exibir feedback adequado.

---

## 8. Hook de Status de Rede — `useNetworkStatus`

Hook global reutilizável que centraliza toda a lógica de detecção de rede:

```ts
// src/hooks/use-network-status.ts
import { useEffect, useState } from 'react';
import NetInfo, { type NetInfoState } from '@react-native-community/netinfo';

export type NetworkStatus = {
  isOnline: boolean;
  isOffline: boolean;
  connectionType: NetInfoState['type'] | null;
  isInternetReachable: boolean | null;
};

/**
 * Hook global para detecção de conectividade.
 * Usado por componentes de feedback visual (banner, ícone, etc.).
 *
 * NÃO use este hook para pausar mutations — isso é feito
 * automaticamente pelo onlineManager configurado em setupOnlineManager().
 */
export function useNetworkStatus(): NetworkStatus {
  const [state, setState] = useState<NetworkStatus>({
    isOnline: true,           // assume online até primeira leitura
    isOffline: false,
    connectionType: null,
    isInternetReachable: null,
  });

  useEffect(() => {
    // Lê o estado atual imediatamente (sem esperar por mudança)
    NetInfo.fetch().then((netState) => {
      const isOnline = (netState.isConnected ?? false) && (netState.isInternetReachable ?? false);
      setState({
        isOnline,
        isOffline: !isOnline,
        connectionType: netState.type,
        isInternetReachable: netState.isInternetReachable,
      });
    });

    // Escuta mudanças subsequentes
    const unsubscribe = NetInfo.addEventListener((netState) => {
      const isOnline = (netState.isConnected ?? false) && (netState.isInternetReachable ?? false);
      setState({
        isOnline,
        isOffline: !isOnline,
        connectionType: netState.type,
        isInternetReachable: netState.isInternetReachable,
      });
    });

    return unsubscribe; // cleanup: remove o listener ao desmontar
  }, []);

  return state;
}
```

---

## 9. Feedback Visual — Componentes de Status de Rede

### 9.1 Banner Offline Global

```tsx
// src/components/network-status-banner/network-status-banner.tsx
import { useEffect, useRef } from 'react';
import { View, Text, StyleSheet, Animated } from 'react-native';
import { useNetworkStatus } from '@hooks/use-network-status';

/**
 * Banner que aparece no topo do app quando offline.
 * Deve ser inserido no layout raiz, dentro do PersistQueryClientProvider.
 */
export function NetworkStatusBanner() {
  const { isOffline } = useNetworkStatus();
  const translateY = useRef(new Animated.Value(-60)).current;

  useEffect(() => {
    Animated.timing(translateY, {
      toValue: isOffline ? 0 : -60,
      duration: 300,
      useNativeDriver: true,
    }).start();
  }, [isOffline, translateY]);

  return (
    <Animated.View style={[styles.banner, { transform: [{ translateY }] }]}>
      <Text style={styles.text}>📡 Sem conexão — exibindo dados salvos</Text>
    </Animated.View>
  );
}

const styles = StyleSheet.create({
  banner: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    backgroundColor: '#F59E0B',  // amber-500
    paddingTop: 48,              // safe area top
    paddingBottom: 12,
    paddingHorizontal: 16,
    zIndex: 999,
  },
  text: {
    color: '#1C1917',
    fontWeight: '600',
    fontSize: 14,
    textAlign: 'center',
  },
});
```

### 9.2 Indicador de Mutation Pausada

```tsx
// src/components/sync-indicator/sync-indicator.tsx
import { View, Text, ActivityIndicator, StyleSheet } from 'react-native';
import { useIsMutating } from '@tanstack/react-query';
import { useNetworkStatus } from '@hooks/use-network-status';

/**
 * Indicador pequeno que aparece quando há mutations pendentes offline.
 * Exibe "Salvando..." quando online processando, e "Pendente sync" quando offline.
 */
export function SyncIndicator() {
  const isMutating = useIsMutating();
  const { isOffline } = useNetworkStatus();

  if (!isMutating) return null;

  return (
    <View style={styles.container}>
      <ActivityIndicator size="small" color="#6B7280" />
      <Text style={styles.text}>
        {isOffline ? '⏳ Pendente — sincronizará ao reconectar' : '🔄 Salvando...'}
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
    paddingHorizontal: 16,
    paddingVertical: 8,
    backgroundColor: '#F3F4F6',
    borderRadius: 8,
  },
  text: {
    fontSize: 13,
    color: '#374151',
  },
});
```

### 9.3 Integração no Layout Raiz

```tsx
// src/app/_layout.tsx (atualizado)
import { useEffect } from 'react';
import { View } from 'react-native';
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { queryClient } from '@services/query-client';
import { asyncStoragePersister } from '@services/query-persister';
import { setupOnlineManager } from '@services/online-manager';
import { NetworkStatusBanner } from '@components/network-status-banner';

export default function RootLayout() {
  useEffect(() => {
    const cleanup = setupOnlineManager();
    return cleanup;
  }, []);

  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: asyncStoragePersister,
        maxAge: 24 * 60 * 60 * 1000,
        dehydrateOptions: {
          shouldDehydrateMutation: (mutation) =>
            mutation.state.status === 'pending',
        },
      }}
      onSuccess={() => {
        queryClient.resumePausedMutations().then(() => {
          queryClient.invalidateQueries();
        });
      }}
    >
      <View style={{ flex: 1 }}>
        {/* Banner aparece sobre todo o conteúdo */}
        <NetworkStatusBanner />
        <Stack />
      </View>
    </PersistQueryClientProvider>
  );
}
```

---

## 10. Padrão em Telas — Queries com Cache Offline

O cache persistido faz o `useQuery` exibir dados locais automaticamente quando offline. Na tela, basta indicar ao usuário que está vendo dados salvos:

```tsx
// src/features/products/screens/products-screen.tsx
import { useProducts } from '../queries/use-products';
import { useNetworkStatus } from '@hooks/use-network-status';
import { ErrorState } from '@components/error-state';

export function ProductsScreen() {
  const { data: products, isLoading, error, refetch } = useProducts();
  const { isOffline } = useNetworkStatus();

  if (isLoading && !products) return <LoadingSpinner />;

  // Apenas exibe o estado de erro se não há cache disponível
  if (error && !products) return <ErrorState error={error} onRetry={refetch} />;

  return (
    <View>
      {/* Aviso inline quando está offline E tem dados em cache */}
      {isOffline && products && (
        <OfflineBadge message="Dados salvos — atualizará ao reconectar" />
      )}
      <ProductList products={products ?? []} />
    </View>
  );
}
```

> **Regra:** Nunca mostre tela de loading pura quando já há dados em cache (`products !== undefined`). O `isLoading` será `true` no background para dados stale — use `isLoading && !data` para o estado inicial sem cache.

---

## 11. Organização de Arquivos

```text
src/
├── services/
│   ├── query-client.ts          ← QueryClient + gcTime aumentado + mutationDefaults
│   ├── query-persister.ts       ← createAsyncStoragePersister
│   └── online-manager.ts        ← setupOnlineManager (NetInfo → onlineManager)
│
├── hooks/
│   └── use-network-status.ts    ← Hook global de status de rede (feedback visual)
│
├── components/
│   ├── network-status-banner/
│   │   ├── network-status-banner.tsx   ← Banner offline animado
│   │   └── index.ts
│   └── sync-indicator/
│       ├── sync-indicator.tsx          ← Indicador de mutations pendentes
│       └── index.ts
│
└── features/
    └── products/
        ├── mutations/
        │   └── use-create-product.ts   ← useMutation com mutationKey obrigatório
        └── ...
```

---

## 12. Integração com `Result<T, E>` e `ApiError`

O padrão de `Result<T, E>` (definido na skill `error-handling`) é totalmente compatível com o fluxo offline. O `mutationFn` desempacota o `Result` normalmente — se offline, o TanStack Query pausa antes mesmo de chamar o `mutationFn`:

```ts
// ✅ CORRETO — desempacota Result dentro do mutationFn (como sempre)
// Offline: o TanStack Query pausa ANTES de chamar este fn
// Online: o Result<T, E> é desempacotado normalmente
mutationFn: async (payload) => {
  const result = await createProduct(payload);
  if (result.isErr()) throw result.error; // ApiError tipado
  return result.value;
},
```

---

## 13. Checklist de Validação

Antes de entregar qualquer feature com suporte offline, verifique:

- [ ] `gcTime` no `QueryClient` está definido como 24h (ou mais) para queries que precisam de cache longo?
- [ ] O `setupOnlineManager()` é chamado no `useEffect` do layout raiz?
- [ ] O `PersistQueryClientProvider` substitui o `QueryClientProvider` no layout raiz?
- [ ] O `onSuccess` do `PersistQueryClientProvider` chama `resumePausedMutations()`?
- [ ] Mutations offline têm `mutationKey` declarado no `useMutation`?
- [ ] O `setMutationDefaults` está registrado para cada mutation offline em `query-client.ts`?
- [ ] O `NetworkStatusBanner` está incluído no layout raiz?
- [ ] Telas que exibem dados usam `isLoading && !data` (não `isLoading` puro) para o loading inicial?
- [ ] O `useNetworkStatus` detecta tanto `isConnected` quanto `isInternetReachable`?
- [ ] As chaves do AsyncStorage não conflitam com as das stores Zustand (`auth-storage`, `cart-storage`)?

---

## 14. Padrões e Anti-Padrões

### ✅ Faça

```ts
// ✅ Use PersistQueryClientProvider (não QueryClientProvider) para offline
<PersistQueryClientProvider persistOptions={{ persister }} onSuccess={...}>

// ✅ Declare mutationKey em toda mutation que precisa sobreviver offline
useMutation({ mutationKey: ['products', 'create'], mutationFn: ... })

// ✅ Registre mutationDefaults para re-associar mutationFn após restart
queryClient.setMutationDefaults(['products', 'create'], { mutationFn: ... })

// ✅ Use isConnected && isInternetReachable (não só isConnected)
setOnline(state.isConnected && state.isInternetReachable)

// ✅ Exiba dados do cache antes de exibir loading
if (isLoading && !data) return <Loading />;
```

### ❌ Evite

```ts
// ❌ Não use setTimeout para "tentar de novo" manualmente — use resumePausedMutations
setTimeout(() => mutation.mutate(payload), 5000);

// ❌ Não use Zustand para cachear respostas de API — duplica com TanStack Query
const [products, setProducts] = useProductStore(); // ← errado para dados de servidor

// ❌ Não use isConnected isolado — rede sem internet engana o sensor
setOnline(state.isConnected); // pode ser true em Wi-Fi sem saída para internet

// ❌ Não declare mutationKey sem registrar mutationDefaults (mutation não renasce após restart)
useMutation({ mutationKey: ['products', 'create'] }); // sem setMutationDefaults → fila perdida

// ❌ Não esconda dados em cache por causa do loading
if (isLoading) return <Loading />; // mostra loading mesmo com dados disponíveis
```
