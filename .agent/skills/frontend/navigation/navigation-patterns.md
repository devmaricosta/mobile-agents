---
name: navigation-patterns
description: Padrões avançados com Expo Router SDK 54+: deep linking com scheme customizado e +native-intent, navegação autenticada com Stack.Protected e redirect automático via Zustand, parâmetros de rota tipados com typed routes, e como estruturar layouts aninhados sem poluir a pasta app/.
---

# Padrões de Navegação — Expo Router SDK 54+

Você é um Engenheiro de Software Senior especialista em navegação com Expo Router. Sua responsabilidade é implementar padrões de roteamento robustos, type-safe e alinhados à arquitetura definida nas skills `arquiteto-react-native-features` e `setup-react-native`, respeitando a separação entre a camada de roteamento (`src/app/`) e a camada de features (`src/features/`).

---

## Quando usar esta skill

- **Proteção de rotas:** Implementar fluxo de autenticação com redirect automático.
- **Deep linking:** Configurar scheme personalizado, Universal Links e reescrita de URLs nativas.
- **Parâmetros tipados:** Passar dados entre telas com type safety completo.
- **Layouts aninhados:** Criar grupos de rota sem poluir a estrutura do `app/`.
- **Dúvidas de roteamento:** Onde colocar lógica de navegação, guards e redirecionamentos.

---

## Estrutura de Arquivos de Navegação

A pasta `src/app/` é uma **camada fina de roteamento**. Toda a lógica real vive em `src/features/`. A estrutura abaixo é o padrão obrigatório:

```text
src/
├── app/
│   ├── _layout.tsx                  # Layout raiz — providers globais (QueryClient, etc.)
│   ├── +native-intent.tsx           # Reescritor de deep links nativos (opcional)
│   │
│   ├── (auth)/                      # Grupo: rotas públicas (não autenticadas)
│   │   ├── _layout.tsx              # Guard: redireciona autenticados para (home)
│   │   ├── login.tsx                # → export { default } from '@features/auth/...'
│   │   └── register.tsx
│   │
│   ├── (home)/                      # Grupo: rotas protegidas (autenticadas)
│   │   ├── _layout.tsx              # Guard: redireciona não-autenticados para (auth)
│   │   ├── index.tsx
│   │   ├── (tabs)/                  # Sub-grupo: tab navigator
│   │   │   ├── _layout.tsx          # Tab navigator (Tabs component)
│   │   │   ├── feed.tsx
│   │   │   └── profile.tsx
│   │   └── product/
│   │       └── [id].tsx             # Rota dinâmica
│   │
│   └── +not-found.tsx               # Fallback 404
│
├── constants/
│   └── routes.ts                    # Mapa centralizado de ROUTES (obrigatório)
│
└── features/
    └── auth/
        └── store/
            └── auth.store.ts        # isAuthenticated — fonte da verdade para guards
```

> **Regra:** Grupos de rota (pastas com parênteses) **não afetam a URL**. Use-os para organizar layouts e aplicar guards sem criar segmentos de URL sem sentido.

---

## 1. Constantes de Rotas — `src/constants/routes.ts`

> **Regra obrigatória (skill `arquiteto-react-native-features`):** Features **não devem usar strings hardcoded** para navegar. Use o mapa centralizado `ROUTES`. Importe via alias `@constants/routes`.

```ts
// src/constants/routes.ts
import type { Href } from 'expo-router';

/**
 * Mapa central de todas as rotas da aplicação.
 * Tipado com `Href` do Expo Router para garantir type safety na navegação.
 *
 * REGRA: Toda navegação programática usa ROUTES.
 * NUNCA hardcode strings de rota em hooks, componentes ou screens.
 */
export const ROUTES = {
  // Grupo auth (público)
  LOGIN: '/(auth)/login' as Href,
  REGISTER: '/(auth)/register' as Href,

  // Grupo home (protegido)
  HOME: '/(home)' as Href,

  // Tabs (dentro de home)
  FEED: '/(home)/(tabs)/feed' as Href,
  PROFILE: '/(home)/(tabs)/profile' as Href,

  // Rotas dinâmicas (usar helper para tipagem)
  PRODUCT: (id: string) => `/(home)/product/${id}` as Href,
  SETTINGS: '/(home)/settings' as Href,
} as const;
```

**Uso nos hooks:**

```ts
// src/features/auth/login/hooks/use-login.ts
import { useRouter } from 'expo-router';
import { ROUTES } from '@constants/routes';

export function useLogin() {
  const router = useRouter();

  return useMutation({
    // ...
    onSuccess: () => {
      router.replace(ROUTES.HOME); // ✅ Usa a constante, não string hardcoded
    },
  });
}
```

---

## 2. Navegação Autenticada com Redirect Automático

### 2.1 Fonte da Verdade: `useIsAuthenticated`

O estado de autenticação vive na `auth.store.ts` (skill `zustand-architecture`). A store é a **única fonte da verdade** para guards de rota.

```ts
// src/features/auth/store/auth.store.ts (trecho — ver zustand-architecture para store completa)
export const useIsAuthenticated = () => useAuthStore((s) => s.isAuthenticated);
export const useAuthHydrated = () => useAuthStore((s) => s._hasHydrated);
```

```ts
// src/features/auth/store/index.ts (barrel público da feature auth)
export { useAuthStore, useUser, useIsAuthenticated, useAuthActions, useAuthHydrated } from './auth.store';
```

### 2.2 Layout Raiz — `src/app/_layout.tsx`

O layout raiz configura os providers globais. **Não faz guard aqui** — o guard fica nos layouts de grupo.

```tsx
// src/app/_layout.tsx
import { Stack } from 'expo-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@services/query-client';

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <Stack screenOptions={{ headerShown: false }} />
    </QueryClientProvider>
  );
}
```

### 2.3 Guard de Rotas Públicas — `src/app/(auth)/_layout.tsx`

Redireciona usuários **já autenticados** para fora do fluxo de login.

```tsx
// src/app/(auth)/_layout.tsx
import { Redirect, Stack } from 'expo-router';
// ✅ Importa via barrel público da feature auth — nunca por caminho direto
import { useIsAuthenticated, useAuthHydrated } from '@features/auth';
import { LoadingScreen } from '@components/loading-screen';

export default function AuthLayout() {
  const isAuthenticated = useIsAuthenticated();
  const hasHydrated = useAuthHydrated();

  // Aguarda hidratação do persist para evitar flash de redirect incorreto
  if (!hasHydrated) {
    return <LoadingScreen />;
  }

  // Usuário já logado → redireciona para home
  if (isAuthenticated) {
    return <Redirect href="/(home)" />;
  }

  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Screen name="login" />
      <Stack.Screen name="register" />
    </Stack>
  );
}
```

### 2.4 Guard de Rotas Protegidas — `src/app/(home)/_layout.tsx`

Redireciona usuários **não autenticados** para o login. Usa `Stack.Protected` (SDK 53+).

```tsx
// src/app/(home)/_layout.tsx
import { Redirect, Stack } from 'expo-router';
import { useIsAuthenticated, useAuthHydrated } from '@features/auth';
import { LoadingScreen } from '@components/loading-screen';

export default function HomeLayout() {
  const isAuthenticated = useIsAuthenticated();
  const hasHydrated = useAuthHydrated();

  if (!hasHydrated) {
    return <LoadingScreen />;
  }

  // Usuário não autenticado → redireciona para login
  if (!isAuthenticated) {
    return <Redirect href="/(auth)/login" />;
  }

  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Screen name="index" />
      <Stack.Screen name="(tabs)" />
      <Stack.Screen
        name="product/[id]"
        options={{ presentation: 'modal' }}
      />
      <Stack.Screen name="settings" />
    </Stack>
  );
}
```

> **Por que `Redirect` em vez de `router.replace`?** O componente `<Redirect>` é declarativo e renderizado durante o ciclo de render do layout. Garante que o redirect acontece **antes** de qualquer tela ser montada. `router.replace` é imperativo e deve ser evitado em `useEffect` para redirects de auth (timing imprevisível).

### 2.5 Hidratação do Persist e o Flag `_hasHydrated`

A store Zustand com `persist` é **assíncrona**. Sem aguardar a hidratação, o app pode exibir a tela de login por um frame antes de perceber que o usuário está logado.

```ts
// src/features/auth/store/auth.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

type AuthState = {
  accessToken: string | null;
  user: UserProfile | null;
  isAuthenticated: boolean;
  _hasHydrated: boolean; // Flag de hidratação
};

type AuthActions = {
  setAuth: (user: UserProfile, accessToken: string) => void;
  logout: () => void;
  setHasHydrated: (value: boolean) => void;
};

type AuthStore = AuthState & AuthActions;

const initialState: AuthState = {
  accessToken: null,
  user: null,
  isAuthenticated: false,
  _hasHydrated: false,
};

export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      ...initialState,
      setAuth: (user, accessToken) =>
        set({ user, accessToken, isAuthenticated: true }),
      logout: () => set({ ...initialState, _hasHydrated: true }),
      setHasHydrated: (value) => set({ _hasHydrated: value }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({
        accessToken: state.accessToken,
        user: state.user,
        isAuthenticated: state.isAuthenticated,
      }),
      // Marca como hidratado após carregar do AsyncStorage
      onRehydrateStorage: () => (state) => {
        state?.setHasHydrated(true);
      },
      version: 1,
    },
  ),
);

// Hooks de conveniência — exportados via barrel
export const useUser = () => useAuthStore((s) => s.user);
export const useIsAuthenticated = () => useAuthStore((s) => s.isAuthenticated);
export const useAuthHydrated = () => useAuthStore((s) => s._hasHydrated);
export const useAuthActions = () =>
  useAuthStore(
    useShallow((s) => ({
      setAuth: s.setAuth,
      logout: s.logout,
    })),
  );
```

### 2.6 Logout e Limpeza de Estado

Ao fazer logout, invalide o cache do TanStack Query e limpe a store:

```ts
// src/features/auth/logout/hooks/use-logout.ts
import { useRouter } from 'expo-router';
import { useQueryClient } from '@tanstack/react-query';
import { useAuthActions } from '@features/auth';
import { ROUTES } from '@constants/routes';

export function useLogout() {
  const router = useRouter();
  const queryClient = useQueryClient();
  const { logout } = useAuthActions();

  return () => {
    logout();                             // Limpa Zustand store
    queryClient.clear();                  // Remove todo o cache do TanStack Query
    router.replace(ROUTES.LOGIN);         // Redireciona para login
  };
}
```

---

## 3. Parâmetros de Rota Tipados

### 3.1 Typed Routes — Geração Automática de Tipos

Expo Router suporta geração automática de tipos para rotas. Habilite no `app.json`:

```json
// app.json
{
  "expo": {
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

Execute `npx expo start` para gerar os tipos em `.expo/types/router.d.ts`. Com isso, `<Link href="...">` e `router.push(...)` verificam a existência da rota em compile time.

### 3.2 Parâmetros Obrigatórios — `useLocalSearchParams`

```tsx
// src/app/(home)/product/[id].tsx — arquivo de rota (camada fina)
export { default } from '@features/product/detail/screens/product-detail-screen';
```

```tsx
// src/features/product/detail/screens/product-detail-screen.tsx
import { useLocalSearchParams } from 'expo-router';

// Tipagem dos parâmetros da rota
type ProductDetailParams = {
  id: string;          // Parâmetro obrigatório — vem da URL
  from?: string;       // Parâmetro opcional — passado programaticamente
};

export default function ProductDetailScreen() {
  // useLocalSearchParams retorna string | string[] — sempre valide
  const { id, from } = useLocalSearchParams<ProductDetailParams>();

  if (!id) return null; // Nunca deve acontecer se a rota é válida

  return <ProductDetailView productId={id} sourcePage={from} />;
}
```

### 3.3 Navegação com Parâmetros Tipados

```ts
// src/features/home/feed/hooks/use-navigate-to-product.ts
import { useRouter } from 'expo-router';

export function useNavigateToProduct() {
  const router = useRouter();

  return (productId: string, from?: string) => {
    router.push({
      pathname: '/(home)/product/[id]',
      params: { id: productId, from },
    });
  };
}
```

```tsx
// Uso no componente
const navigateToProduct = useNavigateToProduct();

<ProductCard
  onPress={() => navigateToProduct(product.id, 'feed')}
/>
```

### 3.4 Parâmetros Globais vs Locais

| Hook | Escopo | Quando usar |
|---|---|---|
| `useLocalSearchParams()` | Apenas a tela atual | Parâmetros da rota atual (ex: `[id]`) |
| `useGlobalSearchParams()` | Todas as telas no stack | Parâmetros propagados para modais/nested |

> **Regra:** Use `useLocalSearchParams` por padrão. Use `useGlobalSearchParams` apenas quando precisar ler parâmetros de uma tela pai dentro de uma tela filho aninhada (ex: tab dentro de stack com parâmetro).

### 3.5 Parâmetros Complexos — Serialização

Expo Router passa parâmetros como **strings** na URL. Para dados complexos (arrays, objetos), serialize:

```ts
// Enviando
router.push({
  pathname: '/(home)/search',
  params: {
    filters: JSON.stringify({ category: 'electronics', minPrice: 100 }),
    tags: ['react', 'expo'].join(','), // Arrays como CSV
  },
});

// Recebendo
const { filters: filtersRaw, tags: tagsRaw } = useLocalSearchParams<{
  filters?: string;
  tags?: string;
}>();

const filters = filtersRaw ? JSON.parse(filtersRaw) : undefined;
const tags = tagsRaw ? tagsRaw.split(',') : [];
```

> **Prefira parâmetros simples.** Se a tela precisar de dados complexos, considere buscar via TanStack Query usando apenas o `id` como parâmetro, em vez de serializar objetos inteiros.

---

## 4. Deep Linking

### 4.1 Configuração Básica no `app.json`

```json
// app.json
{
  "expo": {
    "scheme": "meuapp",
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [
            { "scheme": "https", "host": "meuapp.com.br", "pathPrefix": "/" }
          ],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    },
    "ios": {
      "associatedDomains": ["applinks:meuapp.com.br"]
    }
  }
}
```

Com isso, o Expo Router automaticamente mapeia:
- `meuapp://product/123` → `app/(home)/product/[id].tsx` com `id = "123"`
- `https://meuapp.com.br/product/123` → mesmo destino (Universal Link)

### 4.2 Reescrita de Deep Links — `+native-intent.tsx`

Use o arquivo especial `+native-intent.tsx` para **interceptar e reescrever** URLs nativas antes de o Expo Router processá-las. Útil para:
- Links legados (ex: `meuapp://app/legacy-path` → `/new-path`)
- Integração com serviços de attribution (Branch.io, AppsFlyer)
- Normalização de parâmetros vindos de campanhas de marketing

```tsx
// src/app/+native-intent.tsx
import * as Linking from 'expo-linking';

/**
 * Reescreve URLs nativas antes do roteamento do Expo Router.
 * Chamado quando o app é aberto via deep link (app fechado ou em background).
 *
 * @param url - URL nativa recebida (ex: "meuapp://product/123")
 * @returns URL reescrita (ex: "/product/123") ou a URL original
 */
export function redirectSystemPath({
  path,
  initial,
}: {
  path: string;
  initial: boolean;
}): string {
  try {
    // Normaliza paths legados
    if (path.startsWith('/app/legacy-product/')) {
      const productId = path.replace('/app/legacy-product/', '');
      return `/(home)/product/${productId}`;
    }

    // Remove parâmetros de rastreamento de marketing
    const url = new URL(path, 'https://meuapp.com.br');
    url.searchParams.delete('utm_source');
    url.searchParams.delete('utm_medium');
    url.searchParams.delete('utm_campaign');
    url.searchParams.delete('fbclid');

    return url.pathname + url.search;
  } catch {
    return path; // Retorna o path original em caso de erro
  }
}
```

> **Importante:** `+native-intent.tsx` só é invocado quando o app é **aberto via link externo** (cold start ou background). Links navegados programaticamente dentro do app (via `router.push`) não passam por este arquivo.

### 4.3 Testando Deep Links

```bash
# Android (via adb)
adb shell am start -a android.intent.action.VIEW \
  -d "meuapp://product/123" com.meuapp

# iOS (via xcrun)
xcrun simctl openurl booted "meuapp://product/123"

# Expo Go (em desenvolvimento)
npx uri-scheme open "meuapp://product/123" --ios
npx uri-scheme open "meuapp://product/123" --android
```

---

## 5. Layouts Aninhados Sem Poluir `app/`

### 5.1 Princípio: Grupos de Rota

Grupos de rota (pastas com parênteses `(nome)`) **não geram segmento de URL**. Use-os para:
- Aplicar layouts diferentes a conjuntos de telas
- Agrupar telas que compartilham um guard de autenticação
- Criar sub-navegadores (tabs, drawers) dentro de uma rota

```text
app/
├── (auth)/               # URL: /login (não /(auth)/login)
│   ├── _layout.tsx       # Guard + Stack para rotas públicas
│   ├── login.tsx         # URL: /login
│   └── register.tsx      # URL: /register
│
└── (home)/               # URL: / (não /(home)/)
    ├── _layout.tsx       # Guard + Stack para rotas protegidas
    ├── index.tsx         # URL: /
    ├── settings.tsx      # URL: /settings
    │
    └── (tabs)/           # URL: /feed (não /(home)/(tabs)/feed)
        ├── _layout.tsx   # Tab navigator
        ├── feed.tsx      # URL: /feed
        └── profile.tsx   # URL: /profile
```

### 5.2 Tab Navigator Dentro de Stack

```tsx
// src/app/(home)/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { HomeIcon, UserIcon } from '@components/icons';

/**
 * Tab navigator — somente estrutura de navegação.
 * Toda lógica de negócio fica nas screens dentro de src/features/.
 */
export default function TabsLayout() {
  return (
    <Tabs
      screenOptions={{
        headerShown: false,
        tabBarActiveTintColor: '#6200EE',
      }}
    >
      <Tabs.Screen
        name="feed"
        options={{
          title: 'Feed',
          tabBarIcon: ({ color }) => <HomeIcon color={color} />,
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Perfil',
          tabBarIcon: ({ color }) => <UserIcon color={color} />,
        }}
      />
    </Tabs>
  );
}
```

### 5.3 Modal Dentro de Stack

```tsx
// src/app/(home)/_layout.tsx — adicionando uma rota modal
<Stack screenOptions={{ headerShown: false }}>
  <Stack.Screen name="index" />
  <Stack.Screen name="(tabs)" />
  <Stack.Screen
    name="product/[id]"
    options={{
      presentation: 'modal',           // Apresenta como modal
      animation: 'slide_from_bottom',
    }}
  />
  <Stack.Screen
    name="settings"
    options={{
      presentation: 'card',            // Navegação padrão
      animation: 'slide_from_right',
    }}
  />
</Stack>
```

### 5.4 Drawer Navigator (Alternativa ao Tab)

```tsx
// src/app/(home)/_layout.tsx (variação com Drawer)
import { Drawer } from 'expo-router/drawer';

export default function HomeLayout() {
  const isAuthenticated = useIsAuthenticated();
  const hasHydrated = useAuthHydrated();

  if (!hasHydrated) return <LoadingScreen />;
  if (!isAuthenticated) return <Redirect href="/(auth)/login" />;

  return (
    <Drawer
      screenOptions={{ headerShown: true }}
    >
      <Drawer.Screen name="index" options={{ title: 'Início' }} />
      <Drawer.Screen name="settings" options={{ title: 'Configurações' }} />
    </Drawer>
  );
}
```

---

## 6. Navegação Programática — Padrões Corretos

### 6.1 Hooks Disponíveis

| Hook | Uso | Retorna |
|---|---|---|
| `useRouter()` | Navegação imperativa (`push`, `replace`, `back`) | `Router` |
| `usePathname()` | Pathname atual | `string` |
| `useLocalSearchParams()` | Parâmetros da rota atual | `Record<string, string \| string[]>` |
| `useGlobalSearchParams()` | Parâmetros globais (útil em nested navigators) | `Record<string, string \| string[]>` |
| `useNavigation()` | Acesso ao React Navigation state (evitar) | `NavigationProp` |
| `useSegments()` | Segmentos da URL atual | `string[]` |

### 6.2 Métodos do Router

```ts
import { useRouter } from 'expo-router';
const router = useRouter();

// Empilha nova tela (pressione voltar para retornar)
router.push(ROUTES.PRODUCT('123'));

// Substitui a tela atual (sem botão voltar)
router.replace(ROUTES.HOME);

// Volta para a tela anterior
router.back();

// Verifica se pode voltar (use antes de router.back())
if (router.canGoBack()) {
  router.back();
}

// Navegação com parâmetros
router.push({
  pathname: '/(home)/product/[id]',
  params: { id: '123', from: 'search' },
});
```

### 6.3 Componente `<Link>`

Para links declarativos em JSX, use o componente `<Link>` do Expo Router:

```tsx
import { Link } from 'expo-router';
import { ROUTES } from '@constants/routes';

// Link simples
<Link href={ROUTES.HOME}>Ir para Home</Link>

// Link com parâmetros
<Link
  href={{
    pathname: '/(home)/product/[id]',
    params: { id: '123' },
  }}
>
  Ver Produto
</Link>

// Link que substitui (sem histórico)
<Link href={ROUTES.LOGIN} replace>
  Fazer Login
</Link>

// Link que renderiza um componente customizado
<Link href={ROUTES.HOME} asChild>
  <TouchableOpacity>
    <Text>Home</Text>
  </TouchableOpacity>
</Link>
```

---

## 7. Proteção de Rotas com `Stack.Protected` (SDK 53+)

O `Stack.Protected` é uma API declarativa para proteger rotas. É a alternativa moderna ao padrão `<Redirect>` — mais explícita e visualizável no devtools.

```tsx
// src/app/(home)/_layout.tsx — usando Stack.Protected
import { Stack } from 'expo-router';
import { useIsAuthenticated, useAuthHydrated } from '@features/auth';
import { LoadingScreen } from '@components/loading-screen';

export default function HomeLayout() {
  const isAuthenticated = useIsAuthenticated();
  const hasHydrated = useAuthHydrated();

  if (!hasHydrated) {
    return <LoadingScreen />;
  }

  return (
    <Stack screenOptions={{ headerShown: false }}>
      {/*
        Stack.Protected — protege telas contra acesso não autorizado.
        Quando `guard={false}`, o Expo Router redireciona automaticamente
        para a primeira rota não-protegida do stack pai ou para a rota
        especificada em `redirectTo`.
      */}
      <Stack.Protected guard={isAuthenticated} redirectTo="/(auth)/login">
        <Stack.Screen name="index" />
        <Stack.Screen name="(tabs)" />
        <Stack.Screen name="product/[id]" options={{ presentation: 'modal' }} />
        <Stack.Screen name="settings" />
      </Stack.Protected>
    </Stack>
  );
}
```

> **`Stack.Protected` vs `<Redirect>`:** Ambas as abordagens são válidas. Use `Stack.Protected` quando quiser proteção declarativa visível na estrutura do arquivo. Use `<Redirect>` quando precisar de lógica condicional mais complexa (múltiplos estados, branching).

---

## 8. Fluxo de Implementação (Novo Fluxo de Auth)

Ao receber a solicitação de implementar autenticação com redirect:

**1. Criar/verificar a auth store**
`src/features/auth/store/auth.store.ts` com `isAuthenticated`, `_hasHydrated` e `onRehydrateStorage`.

**2. Exportar hooks via barrel**
`src/features/auth/store/index.ts` → `useIsAuthenticated`, `useAuthHydrated`, `useAuthActions`.

**3. Criar os layouts de grupo**
`src/app/(auth)/_layout.tsx` — guard para autenticados (redirect para home).
`src/app/(home)/_layout.tsx` — guard para não-autenticados (redirect para login).

**4. Criar arquivos de rota (camada fina)**
`src/app/(auth)/login.tsx` → `export { default } from '@features/auth/login/screens/login-screen'`.

**5. Registrar as constantes**
`src/constants/routes.ts` → adicionar `LOGIN`, `REGISTER`, `HOME`.

**6. Implementar o hook de login**
`src/features/auth/login/hooks/use-login.ts` — usa `useMutation`, salva auth na store, navega para `ROUTES.HOME`.

**7. Implementar o hook de logout**
`src/features/auth/logout/hooks/use-logout.ts` — limpa store + TanStack Query cache + navega para `ROUTES.LOGIN`.

**8. Configurar deep linking (se necessário)**
Adicionar `scheme` e `intentFilters` / `associatedDomains` no `app.json`.
Criar `src/app/+native-intent.tsx` se precisar de reescrita de links.

---

## Checklist de Validação

Antes de entregar qualquer código de navegação, verifique:

### Estrutura e Arquitetura
- [ ] `src/app/` contém apenas re-exports e arquivos de layout (`_layout.tsx`)?
- [ ] Nenhum hook de negócio ou JSX complexo em arquivos de rota?
- [ ] Toda navegação usa `ROUTES` de `@constants/routes` — sem strings hardcoded?
- [ ] `@features/auth` é importado via barrel público — nunca por caminho direto?

### Autenticação
- [ ] `_hasHydrated` verificado antes de renderizar `<Redirect>`?
- [ ] `<LoadingScreen />` exibido enquanto awaita hidratação do persist?
- [ ] Layout `(auth)/_layout.tsx` redireciona autenticados para home?
- [ ] Layout `(home)/_layout.tsx` redireciona não-autenticados para login?
- [ ] Logout limpa store Zustand **e** cache do TanStack Query?

### Parâmetros e Tipagem
- [ ] `useLocalSearchParams` tipado com genérico `<ParamType>`?
- [ ] Typed Routes habilitado em `app.json` (`experiments.typedRoutes`)?
- [ ] Parâmetros complexos serializados como string (JSON/CSV)?
- [ ] Componente `<Link>` usado em vez de `router.push` em navegação declarativa?

### Deep Linking
- [ ] `scheme` definido no `app.json`?
- [ ] `intentFilters` (Android) e `associatedDomains` (iOS) configurados para Universal Links?
- [ ] `+native-intent.tsx` criado se houver links legados ou de attribution?
- [ ] Deep links testados via `adb` / `xcrun simctl` antes de entregar?

### Layouts e Grupos
- [ ] Grupos de rota (`(nome)`) usados para organizar sem poluir URLs?
- [ ] Tab/Drawer navigator em `_layout.tsx` — apenas estrutura de navegação, sem lógica?
- [ ] Modais configurados com `presentation: 'modal'` no `Stack.Screen`?
