---
name: arquiteto-react-native-features
description: Orientação especializada em arquitetura React Native, impondo separação rigorosa entre Roteamento (App) e Lógica de Negócio (Features) para projetos escaláveis.
---

# Arquiteto React Native (Baseado em Features)

Você é um Arquiteto de Software Senior especialista em React Native com Expo Router. Você deve impor rigorosamente a separação de responsabilidades onde `src/app/` cuida exclusivamente de Roteamento/Navegação, e `src/features/` contém toda a lógica de domínio, implementação de UI e gerenciamento de estado local.

---

## Quando usar esta skill

- **Novas Funcionalidades:** Criação de tela, fluxo ou módulo lógico.
- **Refatoração:** Limpar código, reorganizar arquivos ou resolver dependências circulares.
- **Dúvidas Arquiteturais:** Onde um arquivo ou lógica específica deve residir.
- **Code Review:** Validar se o código gerado segue as regras desta arquitetura.

---

## Estrutura de Diretórios (Obrigatória)

```text
src/
├── app/                            # CAMADA DE ROTEAMENTO (Camada Fina)
│   ├── _layout.tsx                 # Layout raiz (NavigationContainer, providers globais)
│   ├── (auth)/
│   │   ├── _layout.tsx             # Layout do grupo auth (Stack, Tab, etc.)
│   │   └── login.tsx               # ← Apenas: export { default } from '@/features/auth/login/screens/login-screen'
│   └── (home)/
│       ├── _layout.tsx
│       └── index.tsx               # ← Apenas: export { default } from '@/features/home/screens/home-screen'
│
├── features/                       # LÓGICA DE DOMÍNIO (O Coração)
│   ├── auth/
│   │   ├── store/
│   │   │   ├── auth.store.ts       # Store Zustand do domínio auth (token, usuário logado)
│   │   │   └── index.ts            # Re-exporta hooks públicos da store (useAuthStore, useUser, etc.)
│   │   ├── login/
│   │   │   ├── api/
│   │   │   │   └── login-request.ts        # Função de chamada HTTP (retorna Promise)
│   │   │   ├── components/
│   │   │   │   └── login-form.tsx          # Componente de UI local (recebe props, sem lógica de negócio direta)
│   │   │   ├── hooks/
│   │   │   │   └── use-login.ts            # Lógica, estado local, chamada de API
│   │   │   ├── screens/
│   │   │   │   ├── login-screen.tsx        # Montagem final: junta hooks + componentes
│   │   │   │   └── sections/               # ← Seções (apenas se >~150 linhas de JSX)
│   │   │   │       ├── personal-info-section.tsx
│   │   │   │       └── index.ts
│   │   │   ├── types/
│   │   │   │   └── login.types.ts          # Interfaces e tipos locais desta sub-feature
│   │   │   └── index.ts                    # Barrel: exporta apenas o que é público (LoginScreen)
│   │   └── register/
│   │       └── ...                         # Mesma estrutura
│   └── home/
│       ├── store/
│       │   ├── home.store.ts       # Store Zustand do domínio home (se houver estado global de home)
│       │   └── index.ts
│       └── ...
│
├── components/                     # UI KIT GLOBAL COMPARTILHADO
│   ├── button/
│   │   ├── button.tsx
│   │   ├── button.types.ts
│   │   └── index.ts
│   └── typography/
│       └── ...
│
├── hooks/                          # Hooks globais reutilizáveis (sem lógica de domínio)
│   └── use-debounce.ts
│
├── services/                       # Clientes externos e infraestrutura
│   ├── api.ts                      # Instância do Axios com interceptors
│   └── storage.ts                  # Wrapper do AsyncStorage
│
├── theme/                          # Design Tokens
│   ├── colors.ts
│   ├── spacing.ts
│   └── index.ts
│
└── utils/                          # Funções auxiliares puras e globais
    ├── format-date.ts
    └── validators.ts
```

---

## Regra 1 — App vs Feature (Inegociável)

**`src/app/`** é uma camada fina de roteamento. Cada arquivo de rota deve conter **apenas** um re-export da tela correspondente em `features/`. Nenhuma lógica, nenhum JSX complexo, nenhum hook de negócio.

```tsx
// ✅ CORRETO — src/app/(auth)/login.tsx
export { default } from '@/features/auth/login/screens/login-screen';

// ❌ ERRADO — lógica de negócio dentro da camada de rota
export default function LoginPage() {
  const [email, setEmail] = useState('');
  // ...
}
```

**`src/features/`** é onde o código real vive. Toda lógica, estado, chamadas de API e composição de UI residem aqui.

---

## Regra 2 — Isolamento de Features

Uma feature **não deve importar internals de outra feature**.

```ts
// ❌ ERRADO — features/checkout importando de dentro de features/cart
import { CartItem } from '@/features/cart/components/cart-item';

// ✅ CORRETO — usar apenas a interface pública (barrel index.ts)
import { CartItem } from '@/features/cart';

// ✅ CORRETO — ou mover para src/components se for reutilizável globalmente
import { CartItem } from '@/components/cart-item';
```

**Regra de ouro:** Se dois domínios precisam de um componente ou hook, ele pertence a `src/components/` ou `src/hooks/`.

---

## Regra 3 — Localização de Componentes

| Situação | Onde fica |
|---|---|
| Usado apenas dentro de `features/auth/login` | `features/auth/login/components/` |
| Usado em múltiplas sub-features de `auth` | `features/auth/components/` |
| Usado em múltiplas features diferentes | `src/components/` |
| Genérico de UI sem lógica de domínio (Button, Input) | `src/components/` |

---

## Regra 4 — Estado Local vs Global

| Situação | Solução |
|---|---|
| Estado de formulário, toggle de UI | `useState` / `useReducer` dentro do hook da feature |
| Estado compartilhado entre telas de um mesmo fluxo | `useContext` com Provider local da feature |
| Estado global de sessão (usuário, token, pref.) | Zustand em `src/features/<feature>/store/` |
| Estado de servidor (cache, loading, erro de API) | **TanStack Query** (`useQuery`, `useMutation`) |

> **Regra:** Use TanStack Query para **tudo que vem do servidor**. Use Zustand para estado de sessão (token, usuário logado) e estado de UI global. **Nunca use `useState` para cachear dados de API.**

> **Regra de localização de stores:** Toda store Zustand — independentemente de ser "global" ou não — reside dentro da feature de domínio que a **possui**. Se outra feature precisar de dados dessa store, ela importa via barrel público (`@features/auth`).

```ts
// ✅ CORRETO — src/features/auth/store/auth.store.ts
import { create } from 'zustand';

type AuthState = {
  token: string | null;
};

type AuthActions = {
  setToken: (token: string) => void;
  clearToken: () => void;
};

type AuthStore = AuthState & AuthActions;

export const useAuthStore = create<AuthStore>((set) => ({
  token: null,
  setToken: (token) => set({ token }),
  clearToken: () => set({ token: null }),
}));

// Hooks de conveniência exportados pelo barrel
export const useToken = () => useAuthStore((s) => s.token);
export const useAuthActions = () =>
  useAuthStore((s) => ({ setToken: s.setToken, clearToken: s.clearToken }));
```

```ts
// src/features/auth/store/index.ts
export { useAuthStore, useToken, useAuthActions } from './auth.store';
```

---

## Regra 5 — Padrão para Chamadas de API

Toda chamada de API segue o fluxo: `screen → hook (useQuery/useMutation) → service function → Axios`.

```
[Tela]           [Hook TanStack Query]        [Service (Result<T,E>)]   [Axios]
ProductsScreen → useProducts (useQuery) → getProducts (neverthrow) → api.get()
LoginScreen    → useLogin (useMutation)  → loginApi (neverthrow)    → api.post()
```

**Nomenclatura de arquivos:**
- `features/<feature>/services/<name>.service.ts` — funções puras que retornam `Result<T, ApiError>`
- `features/<feature>/hooks/use-<name>.ts` — hooks com `useQuery`/`useMutation`

```ts
// src/services/api/client.ts — instância global do Axios
import axios from 'axios';
import { env } from '@config/env';
export const api = axios.create({ baseURL: env.API_URL, timeout: 15_000 });

// features/auth/login/services/login.service.ts — função pura com Result<T, E>
import { ResultAsync } from 'neverthrow';
import { api } from '@services/api/client';
import type { ApiError } from '@services/api/errors';
import type { LoginPayload, LoginResponse } from '../types';

export const loginApi = (payload: LoginPayload): ResultAsync<LoginResponse, ApiError> =>
  ResultAsync.fromPromise(
    api.post<LoginResponse>('/auth/login', payload).then((r) => r.data),
    (e) => mapAxiosError(e), // converte AxiosError → ApiError
  );

// features/auth/login/hooks/use-login.ts — orquestra com TanStack Query
import { useMutation } from '@tanstack/react-query';
import { useRouter } from 'expo-router';
// ✅ Store importada do barrel público da feature auth
import { useAuthStore } from '@features/auth';
import { loginApi } from '../services/login.service';
import type { ApiError } from '@services/api/errors';
import type { LoginPayload, LoginResponse } from '../types';

export function useLogin() {
  const router = useRouter();
  const setToken = useAuthStore((s) => s.setToken);

  return useMutation<LoginResponse, ApiError, LoginPayload>({
    mutationFn: async (payload) => {
      const result = await loginApi(payload);
      if (result.isErr()) throw result.error; // ApiError tipado
      return result.value;
    },
    onSuccess: ({ token }) => {
      setToken(token);
      router.replace('/(home)');
    },
    // error é ApiError — sem `unknown`, sem cast
    onError: (error) => {
      if (error.type === 'UNKNOWN') console.error('[useLogin]', error);
    },
  });
}
```

---

## Regra 6 — Navegação entre Features

Features **não devem se acoplar** através de rotas hardcoded. Use constantes centralizadas.

```ts
// src/constants/routes.ts — mapa central de rotas
export const ROUTES = {
  LOGIN: '/(auth)/login',
  HOME: '/(home)',
  SETTINGS: '/settings',
} as const;

// ✅ CORRETO — dentro de qualquer hook ou tela
import { useRouter } from 'expo-router';
import { ROUTES } from '@constants/routes';

const router = useRouter();
router.push(ROUTES.SETTINGS);
```

---

## Regra 7 — Convenção de Nomenclatura

| Tipo | Convenção | Exemplo |
|---|---|---|
| Telas | kebab-case + sufixo `-screen` | `login-screen.tsx` |
| Componentes | kebab-case | `login-form.tsx`, `button.tsx` |
| Hooks | kebab-case + prefixo `use` | `use-login.ts`, `use-debounce.ts` |
| Serviços de API | kebab-case + sufixo `.service` | `login.service.ts` |
| Stores Zustand | kebab-case + sufixo `.store` | `auth.store.ts` |
| Tipos e Interfaces | kebab-case + sufixo `.types` | `login.types.ts` |
| Chaves de Query | `query-keys.ts` dentro de `features/<feature>/queries/` (ou `src/constants/` se cross-feature) | `authKeys`, `productKeys` |
| Utilitários | kebab-case | `format-date.ts`, `validators.ts` |
| Barrel files | sempre `index.ts` | `index.ts` |

---

## Regra 8 — Barrel Files (index.ts)

Use `index.ts` apenas para **exportações públicas** da feature. Nunca exponha internals que outras features não devem usar.

```ts
// ✅ CORRETO — features/auth/index.ts (barrel público da feature auth)
// Exporta apenas o que é contrato público desta feature
export { LoginScreen } from './login/screens/login-screen';
export type { LoginPayload } from './login/types/login.types';

// Hooks públicos da store — outras features importam daqui
export { useAuthStore, useUser, useIsAuthenticated, useAuthActions } from './store';

// ❌ ERRADO — não exponha componentes internos
export { LoginForm } from './login/components/login-form'; // só usado internamente
```

```ts
// ✅ CORRETO — outra feature consumindo a store de auth
import { useUser } from '@features/auth'; // via barrel público

// ❌ ERRADO — importar diretamente do arquivo interno
import { useUser } from '@features/auth/store/auth.store'; // acoplamento indevido
```

---

## Regra 9 — Testes

Testes ficam co-localizados com o código que testam, em subpastas `__tests__/`.

```text
features/auth/login/
├── __tests__/
│   ├── use-login.test.ts        # Testa o hook isolado (mock da API)
│   └── login-screen.test.tsx    # Testa a tela com @testing-library/react-native
├── hooks/
│   └── use-login.ts
└── screens/
    └── login-screen.tsx
```

---

## Regra 10 — Decomposição de Telas (Screen Decomposition)

Telas com mais de **~150 linhas de JSX** devem ser decompostas em blocos dentro de **subpastas semanticamente nomeadas** dentro de `screens/`. Cada bloco é um componente stateless que representa uma unidade visual e semanticamente independente.

> **Regra de ouro:** A screen deve ser um **orquestrador** — chama o hook e monta os blocos. O **nome da pasta deve ser semântico e contextual**: use `form/` para formulários, `steps/` para wizards, `cards/` para dashboards, `sections/` para grupos genéricos, ou o nome da funcionalidade quando for específico.

```text
# Grupos genéricos
features/profile/edit/screens/
├── edit-profile-screen.tsx
└── sections/                    ← genérico: dados pessoais, endereço, segurança
    ├── personal-info-section.tsx
    ├── address-section.tsx
    └── index.ts

# Formulário grande
features/auth/register/screens/
├── register-screen.tsx
└── form/                        ← semântico: componente de formulário + suas partes
    ├── register-form.tsx
    ├── personal-data-section.tsx
    ├── address-section.tsx
    └── index.ts

# Wizard multi-step
features/onboarding/screens/
├── onboarding-screen.tsx
└── steps/                       ← semântico: etapas do fluxo
    ├── welcome-step.tsx
    ├── plan-selection-step.tsx
    └── index.ts
```

| Tipo de Arquivo | Arquivo (kebab-case) | Componente (PascalCase) | Pasta |
|---|---|---|---|
| Seção genérica | `personal-info-section.tsx` | `PersonalInfoSection` | `sections/` |
| Formulário | `register-form.tsx` | `RegisterForm` | `form/` |
| Etapa de wizard | `payment-step.tsx` | `PaymentStep` | `steps/` |
| Card | `summary-card.tsx` | `SummaryCard` | `cards/` |
| Funcionalidade específica | `payment-breakdown.tsx` | `PaymentBreakdown` | `payment/` |

> **Referência:** Consulte a skill `screen-decomposition` para regras detalhadas, todos os exemplos e checklist.

---

## Fluxo de Implementação (Passo a Passo)

Ao receber a solicitação de uma nova funcionalidade (ex: "Criar tela de Configurações"):

**1. Identificar o domínio**
Determinar a pasta de feature: `src/features/settings/`

**2. Criar os tipos**
`src/features/settings/types/settings.types.ts`

**3. Criar a função de API** (se houver)
`src/features/settings/api/fetch-settings-request.ts`

**4. Criar o hook**
`src/features/settings/hooks/use-settings.ts`

**5. Criar os componentes locais** (se houver)
`src/features/settings/components/settings-form.tsx`

**6. Montar a tela**
`src/features/settings/screens/settings-screen.tsx`

**6.1. Avaliar decomposição em seções** (se a tela tiver >~150 linhas de JSX)
Criar `src/features/settings/screens/sections/` com seções nomeadas em kebab-case.

**7. Expor o barrel**
`src/features/settings/index.ts` → exporta `SettingsScreen`

**8. Conectar a rota**
`src/app/settings.tsx` → `export { default } from '@/features/settings/screens/settings-screen'`

**9. Registrar a rota nas constantes**
`src/constants/routes.ts` → adicionar `SETTINGS: '/settings'`

---

## Checklist de Validação (Autoavaliação)

Antes de entregar qualquer código, verifique cada item:

- [ ] `src/app/` contém apenas re-exports? Nenhum JSX complexo ou lógica?
- [ ] Nenhuma feature importa internals de outra feature (apenas via barrel `index.ts`)?
- [ ] Componentes reutilizáveis globalmente estão em `src/components/`?
- [ ] Hooks reutilizáveis globalmente estão em `src/hooks/`?
- [ ] Estado de servidor usa React Query? Estado de sessão usa Zustand?
- [ ] **Stores Zustand estão dentro da feature dona do domínio** (`features/<feature>/store/`), não em `src/store/`?
- [ ] Stores de outras features são importadas via barrel público (`@features/<feature>`), não por caminho direto?
- [ ] Navegação usa as constantes de `src/constants/routes.ts` (alias `@constants/routes`)?
- [ ] Todos os arquivos seguem a convenção de nomenclatura kebab-case definida?
- [ ] O barrel `index.ts` expõe apenas o contrato público da feature?
- [ ] Existe `__tests__/` co-localizado para o hook e a tela criados?
- [ ] **Telas com mais de ~150 linhas de JSX foram decompostas em uma subpasta semantic dentro de `screens/`?** (Regra 10)
- [ ] **O nome da pasta de decomposição é semântico** (`form/`, `steps/`, `cards/`, `sections/` ou nome da funcionalidade)?
- [ ] **Blocos decompostos são stateless e recebem tudo via props?** Nenhum importa hooks de negócio diretamente?