---
name: zustand-architecture
description: Arquitetura de stores Zustand para React Native com Expo SDK 54 — slices por domínio co-localizados nas features, updates imutáveis com immer, persistência com zustand/middleware/persist + AsyncStorage, e seletores granulares para evitar re-renders desnecessários.
---

# Arquitetura Zustand — React Native com Expo (SDK 54+)

Você é um Engenheiro de Software Senior especialista em gerenciamento de estado global com Zustand em projetos React Native. Sua responsabilidade é garantir que **todo estado de sessão e UI global** seja gerenciado exclusivamente pelo Zustand, seguindo as convenções de organização, tipagem e performance definidas neste documento, compatíveis com **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

---

## Quando usar esta skill

- **Nova store:** Criar uma store Zustand para um novo domínio (auth, cart, theme, etc.).
- **Slice pattern:** Dividir uma store grande em slices reutilizáveis.
- **Persistência:** Configurar `persist` middleware com AsyncStorage.
- **Immer:** Usar updates imutáveis com a middleware `immer` para dados aninhados.
- **Performance:** Identificar e corrigir re-renders causados por seletores ineficientes.
- **Code review:** Validar se stores seguem as convenções do projeto.

---

## 1. Regra de Responsabilidade — Zustand vs TanStack Query

> **Regra fundamental:** Use Zustand para estado de **sessão local** (não vem do servidor). Use TanStack Query para **tudo que vem do servidor**.

| Situação | Use |
|---|---|
| Token JWT, refresh token, usuário autenticado | Zustand (`persist`) |
| Preferências de UI (tema, idioma, onboarding visto) | Zustand (`persist`) |
| Estado de UI transitório (drawer aberto, modal visível) | Zustand (sem persist) |
| Dados de API (listagens, detalhes, cache) | **TanStack Query** |
| Dados de formulário locais | `useState` / `useReducer` |
| Estado compartilhado entre telas de um fluxo | `useContext` local da feature |

> **Nunca use Zustand para cachear respostas de API.** Isso duplica responsabilidades e cria inconsistências com o cache do TanStack Query.

---

## 2. Localização das Stores — Co-localização na Feature

> **Regra:** Toda store Zustand reside dentro da feature de domínio que a **possui** (`features/<feature>/store/`). Não existe `src/store/` global.

```text
src/features/
├── auth/
│   ├── store/
│   │   ├── auth.store.ts     ← Store do domínio auth (token, usuário, sessão)
│   │   └── index.ts          ← Re-exporta apenas os hooks públicos
│   └── index.ts              ← Barrel público da feature (inclui hooks da store)
│
├── ui/
│   ├── store/
│   │   ├── ui.store.ts       ← Store de UI global (tema, drawer, toasts)
│   │   └── index.ts
│   └── index.ts
│
└── cart/
    ├── store/
    │   ├── cart.store.ts     ← Store do carrinho
    │   └── index.ts
    └── index.ts
```

> **Acesso cross-feature:** Se uma feature (ex: `checkout`) precisa da store de outra (ex: `cart`), importa **sempre** via barrel público: `import { useCartItems } from '@features/cart'`.

---

## 3. Instalação

```bash
# Zustand (use npx expo install para garantir compatibilidade)
npx expo install zustand

# Immer (para updates imutáveis em dados aninhados)
npx expo install immer

# AsyncStorage (storage provider para o middleware persist)
npx expo install @react-native-async-storage/async-storage
```

> **Compatibilidade SDK 54:** `zustand@5.x` é compatível com React 19.1 (SDK 54). O `immer@10.x` funciona corretamente com a New Architecture (Hermes + Fabric).

---

## 4. Padrão Base — Store com Tipagem Explícita

### 4.1 Estrutura de Tipagem

Sempre separe `State`, `Actions` e combine em `Store`. Defina um `initialState` separado — facilita o reset e os testes.

```ts
// src/features/auth/store/auth.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

import type { UserProfile } from '../types/auth.types';

// 1️⃣ State — apenas dados serializáveis
type AuthState = {
  accessToken: string | null;
  refreshToken: string | null;
  user: UserProfile | null;
  isAuthenticated: boolean;
};

// 2️⃣ Actions — funções que modificam o state
type AuthActions = {
  setAuth: (user: UserProfile, accessToken: string, refreshToken: string) => void;
  setTokens: (accessToken: string, refreshToken: string) => void;
  logout: () => void;
};

// 3️⃣ Tipo completo da store
type AuthStore = AuthState & AuthActions;

// 4️⃣ Estado inicial (reutilizado no reset)
const initialState: AuthState = {
  accessToken: null,
  refreshToken: null,
  user: null,
  isAuthenticated: false,
};

// 5️⃣ Store com middleware persist
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
      // Persiste apenas o necessário — não persista funções nem dados derivados
      partialize: (state) => ({
        accessToken: state.accessToken,
        refreshToken: state.refreshToken,
        user: state.user,
        isAuthenticated: state.isAuthenticated,
      }),
    },
  ),
);

// 6️⃣ Hooks de conveniência — exporte ESTES, não a store bruta
export const useUser = () => useAuthStore((s) => s.user);
export const useIsAuthenticated = () => useAuthStore((s) => s.isAuthenticated);
export const useAccessToken = () => useAuthStore((s) => s.accessToken);
export const useAuthActions = () =>
  useAuthStore(
    useShallow((s) => ({
      setAuth: s.setAuth,
      setTokens: s.setTokens,
      logout: s.logout,
    })),
  );
```

> **Nota sobre `useShallow`:** Use `useShallow` (importado de `zustand/react/shallow`) ao selecionar múltiplos valores em um objeto. Sem ele, um novo objeto é criado a cada render, causando re-renders desnecessários. (Ver seção 7.)

### 4.2 Barrel da Store

```ts
// src/features/auth/store/index.ts
export { useAuthStore, useUser, useIsAuthenticated, useAccessToken, useAuthActions } from './auth.store';
```

```ts
// src/features/auth/index.ts — barrel público da feature
export { LoginScreen } from './login/screens/login-screen';
export type { LoginPayload } from './login/types/auth.types';

// Hooks públicos da store — acessíveis por outras features
export { useUser, useIsAuthenticated, useAuthActions } from './store';

// ❌ Não exponha a store bruta para outras features
// export { useAuthStore } from './store'; // use os hooks de conveniência
```

---

## 5. Middleware `immer` — Updates Imutáveis

Use `immer` quando o estado tiver **objetos ou arrays aninhados** que precisam ser modificados. O `immer` permite escrever mutações como se fossem diretas, mas produz um novo estado imutável.

### 5.1 Quando Usar Immer

| Situação | Use Immer? |
|---|---|
| Atualizar campo raiz: `set({ isOpen: true })` | ❌ Não precisa |
| Atualizar campo aninhado: `user.profile.avatar` | ✅ Sim |
| Adicionar/remover item em array de objetos | ✅ Sim |
| Atualizar item específico por id em lista | ✅ Sim |

### 5.2 Configuração com Immer

```ts
// src/features/cart/store/cart.store.ts
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

import type { CartItem } from '../types/cart.types';

type CartState = {
  items: CartItem[];
  couponCode: string | null;
};

type CartActions = {
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  applyCoupon: (code: string) => void;
  clearCart: () => void;
};

type CartStore = CartState & CartActions;

const initialState: CartState = {
  items: [],
  couponCode: null,
};

export const useCartStore = create<CartStore>()(
  immer((set) => ({
    ...initialState,

    // Com immer: escreve mutações diretas — produz estado imutável
    addItem: (item) =>
      set((state) => {
        const existingItem = state.items.find((i) => i.id === item.id);
        if (existingItem) {
          existingItem.quantity += item.quantity; // ← mutação direta (segura com immer)
        } else {
          state.items.push(item);
        }
      }),

    removeItem: (id) =>
      set((state) => {
        state.items = state.items.filter((i) => i.id !== id);
      }),

    updateQuantity: (id, quantity) =>
      set((state) => {
        const item = state.items.find((i) => i.id === id);
        if (item) item.quantity = quantity;
      }),

    applyCoupon: (code) =>
      set((state) => {
        state.couponCode = code;
      }),

    clearCart: () => set({ ...initialState }),
  })),
);

// Hooks de conveniência
export const useCartItems = () => useCartStore((s) => s.items);
export const useCouponCode = () => useCartStore((s) => s.couponCode);
export const useCartActions = () =>
  useCartStore(
    useShallow((s) => ({
      addItem: s.addItem,
      removeItem: s.removeItem,
      updateQuantity: s.updateQuantity,
      applyCoupon: s.applyCoupon,
      clearCart: s.clearCart,
    })),
  );
```

### 5.3 Immer + Persist combinados

Ao combinar `immer` e `persist`, **a ordem dos middlewares importa**: `persist` deve ser o mais externo.

```ts
// ✅ CORRETO — persist(immer(...))
export const useCartStore = create<CartStore>()(
  persist(
    immer((set) => ({
      ...initialState,
      // actions com set do immer
    })),
    {
      name: 'cart-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({ items: state.items, couponCode: state.couponCode }),
    },
  ),
);

// ❌ ERRADO — immer(persist(...)) — o persist não funciona corretamente
```

---

## 6. Middleware `persist` — Persistência com AsyncStorage

### 6.1 Configuração Completa

```ts
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

persist(
  (set, get) => ({ /* state + actions */ }),
  {
    // Nome único — 1 chave por store no AsyncStorage
    name: 'auth-storage',

    // Storage provider — AsyncStorage para React Native
    storage: createJSONStorage(() => AsyncStorage),

    // partialize — persista apenas dados necessários
    // Nunca persista: actions, dados derivados, estado transitório de UI
    partialize: (state) => ({
      accessToken: state.accessToken,
      refreshToken: state.refreshToken,
      user: state.user,
      isAuthenticated: state.isAuthenticated,
    }),

    // version — incremente ao mudar a estrutura do state persistido
    // Garante que dados antigos no AsyncStorage sejam descartados
    version: 1,

    // migrate — converte dados da versão anterior (opcional)
    migrate: (persistedState, version) => {
      if (version === 0) {
        // Migra de v0 → v1: renomeia `token` para `accessToken`
        const old = persistedState as { token?: string };
        return { ...old, accessToken: old.token, token: undefined };
      }
      return persistedState;
    },

    // onRehydrateStorage — callback executado após reidratação (opcional)
    onRehydrateStorage: () => (state) => {
      if (state) {
        console.log('[auth-storage] reidratado com sucesso');
      }
    },
  },
)
```

### 6.2 Checklist de `partialize`

> **Regra:** Todo campo da store deve ser avaliado antes de persistir.

| Campo | Persistir? | Motivo |
|---|---|---|
| `accessToken` / `refreshToken` | ✅ Sim | Necessário para autenticação entre sessões |
| `user: UserProfile` | ✅ Sim | Evita refetch desnecessário ao abrir o app |
| `isAuthenticated` | ✅ Sim | Determina qual rota mostrar na inicialização |
| `items: CartItem[]` | ✅ Sim | Usuário espera carrinho preservado |
| `theme: 'light' \| 'dark'` | ✅ Sim | Preferência do usuário |
| `isDrawerOpen` | ❌ Não | Estado transitório de UI |
| `isLoading` | ❌ Não | Estado transitório |
| Funções / actions | ❌ Nunca | Não serializáveis |

### 6.3 Lendo o Estado de Hidratação

```ts
// Verificar se a store já foi reidratada (útil na splash screen)
import { useAuthStore } from '@features/auth';

function SplashScreen() {
  // hasHydrated fica `true` após o AsyncStorage carregar
  const hasHydrated = useAuthStore.persist.hasHydrated();

  if (!hasHydrated) return <LoadingScreen />;

  return <AppNavigator />;
}

// Alternativa: ouvir o evento de hidratação no layout raiz
useEffect(() => {
  const unsubscribe = useAuthStore.persist.onHydrate(() => {
    console.log('[auth] store está sendo reidratada...');
  });
  const unsubscribeFinish = useAuthStore.persist.onFinishHydration(() => {
    console.log('[auth] reidratação concluída');
  });
  return () => {
    unsubscribe();
    unsubscribeFinish();
  };
}, []);
```

---

## 7. Seletores Granulares — Evitando Re-renders

> **Regra crítica de performance:** Nunca subscreva a store inteira. Sempre use seletores atômicos ou `useShallow` para múltiplos valores.

### 7.1 Tabela de Decisão de Seletores

| Cenário | Abordagem | Por quê |
|---|---|---|
| Selecionar 1 valor primitivo | Selector individual | Sem overhead |
| Selecionar 1 referência (array/object) | Selector individual | Re-render apenas se referência mudar |
| Selecionar múltiplos valores | `useShallow` | Evita novo objeto a cada render |
| Selecionar actions | Selector individual ou `useShallow` | Actions nunca mudam (referências estáveis) |
| Toda a store | **Nunca** | Re-render em qualquer mudança |

### 7.2 Exemplos Concretos

```ts
import { useShallow } from 'zustand/react/shallow';
import { useAuthStore } from './auth.store';

// ❌ ERRADO — subscreve a store inteira
// Re-renderiza quando QUALQUER campo mudar
const store = useAuthStore();

// ❌ ERRADO — cria novo objeto a cada render (referência instável)
// Re-renderiza SEMPRE, mesmo que user e isAuthenticated não mudem
const { user, isAuthenticated } = useAuthStore((s) => ({
  user: s.user,
  isAuthenticated: s.isAuthenticated,
}));

// ✅ CORRETO — selector atômico individual
// Re-renderiza APENAS quando `user` mudar
const user = useAuthStore((s) => s.user);

// ✅ CORRETO — selector atômico individual
const isAuthenticated = useAuthStore((s) => s.isAuthenticated);

// ✅ CORRETO — useShallow para múltiplos valores
// Compara por valor (shallow equal) em vez de por referência
const { user, isAuthenticated } = useAuthStore(
  useShallow((s) => ({ user: s.user, isAuthenticated: s.isAuthenticated })),
);

// ✅ CORRETO — selector de dados derivados (computados)
// Calcula o total no selector — re-renderiza apenas quando `items` mudar
const totalItems = useCartStore((s) =>
  s.items.reduce((sum, item) => sum + item.quantity, 0),
);

// ✅ CORRETO — actions têm referências estáveis, selector simples é suficiente
const logout = useAuthStore((s) => s.logout);
```

### 7.3 Encapsulamento em Custom Hooks

> **Regra:** Não exponha a store diretamente. Crie custom hooks que encapsulam o selector — facilita refatorações e garante consistência.

```ts
// src/features/auth/store/auth.store.ts

// ✅ Hook que encapsula o selector — componentes não tocam na store bruta
export const useUser = () => useAuthStore((s) => s.user);
export const useIsAuthenticated = () => useAuthStore((s) => s.isAuthenticated);
export const useAccessToken = () => useAuthStore((s) => s.accessToken);

// ✅ Hook que encapsula múltiplas actions com useShallow
export const useAuthActions = () =>
  useAuthStore(
    useShallow((s) => ({
      setAuth: s.setAuth,
      setTokens: s.setTokens,
      logout: s.logout,
    })),
  );
```

```ts
// Consumo no componente — limpo, sem dependência de zustand
import { useUser, useAuthActions } from '@features/auth';

export function ProfileScreen() {
  const user = useUser();               // ← re-render só quando `user` mudar
  const { logout } = useAuthActions();  // ← referência estável, sem re-render extra

  return (
    <View>
      <Text>{user?.name}</Text>
      <Button onPress={logout}>Sair</Button>
    </View>
  );
}
```

---

## 8. Slices Pattern — Stores Compostas

Use o **Slices Pattern** quando uma store cresce e precisa ser dividida em partes menores e reutilizáveis, ou quando um mesmo domínio tem sub-domínios com lógica distinta.

### 8.1 Criando Slices

```ts
// src/features/auth/store/slices/session.slice.ts
import type { StateCreator } from 'zustand';
import type { UserProfile } from '../../types/auth.types';

// Tipagem do slice isolada
export type SessionSlice = {
  user: UserProfile | null;
  isAuthenticated: boolean;
  setUser: (user: UserProfile | null) => void;
  clearSession: () => void;
};

// StateCreator<FullStore, Middlewares[], Mutators[], ThisSlice>
// O `FullStore` é o tipo do store combinado — permite slices se referenciarem
export const createSessionSlice: StateCreator<
  SessionSlice & TokenSlice, // tipo completo da store (todos os slices)
  [['zustand/immer', never]],  // middlewares aplicados (se houver)
  [],
  SessionSlice               // tipo deste slice
> = (set) => ({
  user: null,
  isAuthenticated: false,

  setUser: (user) =>
    set((state) => {
      state.user = user;
      state.isAuthenticated = user !== null;
    }),

  clearSession: () =>
    set((state) => {
      state.user = null;
      state.isAuthenticated = false;
    }),
});
```

```ts
// src/features/auth/store/slices/token.slice.ts
import type { StateCreator } from 'zustand';

export type TokenSlice = {
  accessToken: string | null;
  refreshToken: string | null;
  setTokens: (accessToken: string, refreshToken: string) => void;
  clearTokens: () => void;
};

export const createTokenSlice: StateCreator<
  SessionSlice & TokenSlice,
  [['zustand/immer', never]],
  [],
  TokenSlice
> = (set) => ({
  accessToken: null,
  refreshToken: null,

  setTokens: (accessToken, refreshToken) =>
    set((state) => {
      state.accessToken = accessToken;
      state.refreshToken = refreshToken;
    }),

  clearTokens: () =>
    set((state) => {
      state.accessToken = null;
      state.refreshToken = null;
    }),
});
```

### 8.2 Combinando Slices na Store

```ts
// src/features/auth/store/auth.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';
import AsyncStorage from '@react-native-async-storage/async-storage';

import { createSessionSlice, type SessionSlice } from './slices/session.slice';
import { createTokenSlice, type TokenSlice } from './slices/token.slice';

// Tipo completo da store = combinação de todos os slices
type AuthStore = SessionSlice & TokenSlice;

export const useAuthStore = create<AuthStore>()(
  persist(
    immer((...args) => ({
      ...createSessionSlice(...args),
      ...createTokenSlice(...args),
    })),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
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

// Hooks de conveniência exportados
export const useUser = () => useAuthStore((s) => s.user);
export const useIsAuthenticated = () => useAuthStore((s) => s.isAuthenticated);
export const useAccessToken = () => useAuthStore((s) => s.accessToken);
export const useAuthActions = () =>
  useAuthStore(
    useShallow((s) => ({
      setUser: s.setUser,
      clearSession: s.clearSession,
      setTokens: s.setTokens,
      clearTokens: s.clearTokens,
    })),
  );
```

> **Quando usar slices vs store simples:**
>
> | Situação | Usar |
> |---|---|
> | Store pequena (≤ 5 campos) | Store direta |
> | Store com lógica distinta por sub-domínio | Slices |
> | Reutilizar lógica em múltiplas stores | Slices compartilhados |
> | Feature com estado complexo (ex: cart + checkout) | Slices |

---

## 9. Acessando a Store Fora de Componentes React

Use `.getState()` para ler ou modificar o estado em contextos não-React (interceptors HTTP, workers, funções utilitárias).

```ts
// Leitura direta do state (interceptor do Axios, etc.)
// src/services/api/client.ts
import { useAuthStore } from '@features/auth';

api.interceptors.request.use((config) => {
  // ✅ .getState() — sem hooks, sem React, sem re-renders
  const token = useAuthStore.getState().accessToken;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Chamar actions fora de componentes
api.interceptors.response.use(undefined, async (error) => {
  if (error.response?.status === 401) {
    // ✅ Acessa actions via .getState()
    useAuthStore.getState().logout();
    router.replace('/(auth)/login');
  }
  return Promise.reject(error);
});

// Subscrever mudanças fora do React (ex: sincronização com analytics)
const unsubscribe = useAuthStore.subscribe(
  (state) => state.isAuthenticated,       // selector
  (isAuthenticated, prevIsAuthenticated) => {
    if (!isAuthenticated && prevIsAuthenticated) {
      // Usuário fez logout — limpar dados locais
      analytics.clearUser();
    }
  },
  { fireImmediately: true },  // dispara imediatamente com o valor atual
);
```

---

## 10. Store de UI Global — Padrão sem Persistência

Para estados de UI global transitório (drawer, modal, toasts), use uma store simples **sem** `persist`:

```ts
// src/features/ui/store/ui.store.ts
import { create } from 'zustand';
import { useShallow } from 'zustand/react/shallow';

type ToastMessage = {
  id: string;
  type: 'success' | 'error' | 'info' | 'warning';
  message: string;
};

type UIState = {
  isDrawerOpen: boolean;
  activeModal: string | null;
  toasts: ToastMessage[];
};

type UIActions = {
  openDrawer: () => void;
  closeDrawer: () => void;
  openModal: (modalId: string) => void;
  closeModal: () => void;
  showToast: (toast: Omit<ToastMessage, 'id'>) => void;
  dismissToast: (id: string) => void;
};

type UIStore = UIState & UIActions;

export const useUIStore = create<UIStore>()((set) => ({
  // State
  isDrawerOpen: false,
  activeModal: null,
  toasts: [],

  // Actions
  openDrawer: () => set({ isDrawerOpen: true }),
  closeDrawer: () => set({ isDrawerOpen: false }),
  openModal: (modalId) => set({ activeModal: modalId }),
  closeModal: () => set({ activeModal: null }),

  showToast: (toast) =>
    set((state) => ({
      toasts: [...state.toasts, { ...toast, id: Date.now().toString() }],
    })),

  dismissToast: (id) =>
    set((state) => ({
      toasts: state.toasts.filter((t) => t.id !== id),
    })),
}));

// Hooks de conveniência
export const useIsDrawerOpen = () => useUIStore((s) => s.isDrawerOpen);
export const useActiveModal = () => useUIStore((s) => s.activeModal);
export const useToasts = () => useUIStore((s) => s.toasts);
export const useUIActions = () =>
  useUIStore(
    useShallow((s) => ({
      openDrawer: s.openDrawer,
      closeDrawer: s.closeDrawer,
      openModal: s.openModal,
      closeModal: s.closeModal,
      showToast: s.showToast,
      dismissToast: s.dismissToast,
    })),
  );
```

---

## 11. Organização Completa de Arquivos

```text
src/features/
│
├── auth/
│   ├── store/
│   │   ├── slices/
│   │   │   ├── session.slice.ts       ← Slice: user, isAuthenticated
│   │   │   └── token.slice.ts         ← Slice: accessToken, refreshToken
│   │   ├── auth.store.ts              ← Combina slices + persist + immer
│   │   └── index.ts                   ← Re-exporta hooks públicos
│   ├── login/
│   │   ├── mutations/
│   │   │   └── use-login.ts           ← useMutation que chama setAuth()
│   │   ├── services/
│   │   │   └── auth.service.ts
│   │   └── ...
│   └── index.ts                       ← Barrel público (inclui hooks da store)
│
├── cart/
│   ├── store/
│   │   ├── cart.store.ts              ← Store com immer + persist
│   │   └── index.ts
│   └── index.ts
│
└── ui/
    ├── store/
    │   ├── ui.store.ts                ← Store sem persist (UI transitória)
    │   └── index.ts
    └── index.ts
```

> **Notas sobre a estrutura:**
> - O diretório `slices/` só é criado quando a store usa o Slices Pattern.
> - Stores simples (sem slices) têm apenas `<feature>.store.ts` + `index.ts`.
> - O `index.ts` da store somente re-exporta hooks de conveniência, nunca a store bruta.

---

## 12. Integração com o Fluxo de Login

Exemplo completo mostrando como Zustand e TanStack Query se complementam:

```ts
// src/features/auth/mutations/use-login.ts
import { useMutation } from '@tanstack/react-query';
import type { UseMutationResult } from '@tanstack/react-query';
import { useQueryClient } from '@tanstack/react-query';
import { useRouter } from 'expo-router';

import { login } from '../services/auth.service';
import { useAuthActions } from '../store/auth.store';
import type { ApiError } from '@services/api/errors';
import type { LoginPayload, LoginResponse } from '../types/auth.types';
import { ROUTES } from '@constants/routes';

export function useLogin(): UseMutationResult<LoginResponse, ApiError, LoginPayload> {
  const queryClient = useQueryClient();
  const router = useRouter();
  // ✅ Pega apenas a action necessária — referência estável, sem re-render
  const { setAuth } = useAuthActions();

  return useMutation<LoginResponse, ApiError, LoginPayload>({
    mutationFn: async (credentials) => {
      const result = await login(credentials);
      // Result<T, E> → throw para o TanStack Query capturar como ApiError
      if (result.isErr()) throw result.error;
      return result.value;
    },
    onSuccess: ({ user, accessToken, refreshToken }) => {
      // ✅ Zustand: persiste sessão
      setAuth(user, accessToken, refreshToken);
      // ✅ TanStack Query: limpa cache do usuário anterior
      queryClient.clear();
      router.replace(ROUTES.HOME);
    },
    onError: (error) => {
      if (error.type === 'UNKNOWN') {
        console.error('[useLogin] Erro inesperado:', error);
      }
    },
  });
}
```

---

## 13. Padrões e Anti-Padrões

### ✅ Faça

```ts
// ✅ Seletores atômicos individuais
const user = useAuthStore((s) => s.user);
const isAuthenticated = useAuthStore((s) => s.isAuthenticated);

// ✅ useShallow para múltiplos valores
const { setAuth, logout } = useAuthStore(useShallow((s) => ({ setAuth: s.setAuth, logout: s.logout })));

// ✅ .getState() fora de componentes React (interceptors, funções utils)
const token = useAuthStore.getState().accessToken;

// ✅ partialize para persistir apenas o necessário
partialize: (state) => ({ accessToken: state.accessToken, user: state.user }),

// ✅ version no persist para migrações seguras
{ name: 'auth', storage: createJSONStorage(() => AsyncStorage), version: 1 }

// ✅ immer para dados aninhados (listas, objetos complexos)
addItem: (item) => set((state) => { state.items.push(item); }),

// ✅ persist(immer(...)) — ordem correta de middlewares
create<CartStore>()(persist(immer((set) => ({ ... })), { name: 'cart' }))

// ✅ Acessar store de outra feature via barrel público
import { useIsAuthenticated } from '@features/auth';
```

### ❌ Evite

```ts
// ❌ Subscrever a store inteira — re-render em qualquer mudança
const store = useAuthStore();

// ❌ Selector que retorna novo objeto sem useShallow
const { a, b } = useAuthStore((s) => ({ a: s.a, b: s.b }));

// ❌ Usar Zustand para cachear dados de API
const products = useProductStore((s) => s.products); // use TanStack Query!

// ❌ Usar useState para estado que precisa ser global
const [isAuthenticated, setIsAuthenticated] = useState(false); // use Zustand!

// ❌ Importar diretamente da store de outra feature (acoplamento indevido)
import { useAuthStore } from '@features/auth/store/auth.store'; // via barrel!

// ❌ Persistir actions ou funções no storage
partialize: (state) => ({ ...state }), // inclui actions — erro silencioso!

// ❌ Ordem errada de middlewares
create()(immer(persist(...))) // persist deve ser externo ao immer
```

---

## 14. Checklist de Validação

Antes de entregar código com Zustand, verifique cada item:

- [ ] A store reside em `features/<feature>/store/`, não em `src/store/`?
- [ ] O tipo da store é separado em `State` + `Actions` + `Store` = interseção?
- [ ] Existe `initialState` separado (reutilizado no reset/logout)?
- [ ] Nenhum componente usa `useStore()` sem selector?
- [ ] Múltiplos valores selecionados usam `useShallow`?
- [ ] Actions são acessadas por hooks de conveniência (`useAuthActions`)?
- [ ] O `persist` tem `partialize` que exclui actions e dados transitórios?
- [ ] O `persist` tem `version` definido?
- [ ] A ordem dos middlewares é `persist(immer(...))` (persist externo)?
- [ ] O barrel `store/index.ts` exporta apenas hooks de conveniência, não a store bruta?
- [ ] O barrel `feature/index.ts` expõe apenas os hooks públicos da store?
- [ ] Stores de outras features são importadas via barrel público (`@features/<feature>`)?
- [ ] `.getState()` é usado (não hooks) em contextos fora do React?
- [ ] Dados do servidor não estão sendo armazenados no Zustand (use TanStack Query)?
