---
name: feature-flags
description: Gerenciamento de feature flags para projetos Expo SDK 54+. Cobre integração com Statsig (@statsig/expo-bindings) como solução remota, implementação local simples como fallback, abstração de consumo sem acoplamento via hook centralizado, e rollout gradual por percentual de usuários.
---

# Feature Flags — Expo SDK 54+

Você é um Engenheiro de Software Senior especialista em feature management para React Native. Sua responsabilidade é garantir que o projeto use feature flags de forma **desacoplada, tipada e segura**, permitindo rollout gradual, kill switches e A/B testing sem acoplar as features a um vendor específico, compatível com o **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

> **Pré-requisitos:** Esta skill assume que as skills `environment-config`, `setup-react-native` e `architect-react-native-features` já foram configuradas. O módulo `src/config/env.ts` e os aliases de path (`@config`, `@services`, `@features`, etc.) são utilizados diretamente nesta skill.

---

## Quando usar esta skill

- **Novo projeto:** Configurar a infraestrutura de feature flags do zero.
- **Rollout gradual:** Liberar uma feature para uma porcentagem crescente de usuários sem novo deploy.
- **Kill switch:** Desativar uma feature problemática em produção sem rollback de código.
- **A/B Testing:** Testar variantes com diferentes segmentos de usuários.
- **Beta / QA:** Habilitar features exclusivamente em ambientes staging/development.
- **Consumo sem acoplamento:** Isolar a lógica de negócio de um vendor de feature flags específico.

---

## 1. Visão Geral da Estratégia

```text
┌──────────────────────────────────────────────────────────────────────┐
│               ARQUITETURA DE FEATURE FLAGS                           │
├──────────────────────────────────────────────────────────────────────┤
│  Opção A (default):                                                  │
│    Statsig (remoto) ──► FeatureFlagsProvider ──► useFeatureFlag()   │
│                                                                      │
│  Opção B (fallback/local):                                           │
│    env.ts (EXPO_PUBLIC_) ──► LocalProvider ──► useFeatureFlag()     │
├──────────────────────────────────────────────────────────────────────┤
│               CONSUMO (SEMPRE IGUAL, INDEPENDENTE DO PROVIDER)       │
│                                                                      │
│  feature/hooks/use-*.ts                                              │
│        └──► useFeatureFlag('minha-flag') → true | false              │
│                                                                      │
│  REGRA: features jamais importam Statsig diretamente                 │
└──────────────────────────────────────────────────────────────────────┘
```

| Camada | Responsabilidade |
|---|---|
| **`src/services/feature-flags/`** | Provider abstrato + implementações (Statsig, Local) |
| **`src/hooks/use-feature-flag.ts`** | Hook público único de consumo (sem acoplamento) |
| **`src/config/env.ts`** | Variáveis de ambiente para configurar o provider |
| **`src/app/_layout.tsx`** | Montagem do `FeatureFlagsProvider` no root |

> **Princípio fundamental:** Features **nunca** dependem de Statsig, LaunchDarkly ou qualquer vendor diretamente. Elas consomem apenas `useFeatureFlag()`. Trocar o provider não requer alterar nenhuma feature.

---

## 2. Instalação

### 2.1 Estatig — SDK recomendado (2025)

> **Atenção:** O pacote legado `statsig-react-native-expo` foi **descontinuado em 31/01/2025**. Use exclusivamente `@statsig/expo-bindings` com `@statsig/js-client`.

```bash
# SDK Statsig + bindings para Expo
npx expo install @statsig/expo-bindings

# Peer dependencies obrigatórias
npx expo install expo-device expo-application @react-native-async-storage/async-storage
```

> O `@react-native-async-storage/async-storage` já é instalado pela skill `zustand-architecture` — verifique antes de reinstalar.

### 2.2 Variáveis de Ambiente

Adicione ao `src/config/env.ts` (conforme a skill `environment-config`):

```typescript
// src/config/env.ts — adicionar as variáveis de Feature Flags

// ─── Feature Flags — Provider ────────────────────────────────────────────────
// 'statsig' (padrão) | 'local' (fallback sem internet / desenvolvimento)
export const FEATURE_FLAGS_PROVIDER =
  (process.env.EXPO_PUBLIC_FEATURE_FLAGS_PROVIDER as 'statsig' | 'local') ?? 'local';

// ─── Statsig ─────────────────────────────────────────────────────────────────
// Client SDK Key (seguro para expor no bundle — não é um secret)
export const STATSIG_CLIENT_KEY = process.env.EXPO_PUBLIC_STATSIG_CLIENT_KEY ?? '';

// ─── Export consolidado (adicionar ao objeto `env`) ──────────────────────────
export const env = {
  // ... campos existentes (APP_ENV, API_URL, etc.)
  FEATURE_FLAGS_PROVIDER,
  STATSIG_CLIENT_KEY,
} as const;
```

Adicione aos arquivos `.env.*`:

```env
# .env.development
EXPO_PUBLIC_FEATURE_FLAGS_PROVIDER=local
EXPO_PUBLIC_STATSIG_CLIENT_KEY=

# .env.staging
EXPO_PUBLIC_FEATURE_FLAGS_PROVIDER=statsig
EXPO_PUBLIC_STATSIG_CLIENT_KEY=client-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# .env.production
EXPO_PUBLIC_FEATURE_FLAGS_PROVIDER=statsig
EXPO_PUBLIC_STATSIG_CLIENT_KEY=client-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Adicione ao `eas.json` (nos profiles `staging` e `production`):

```json
{
  "build": {
    "staging": {
      "env": {
        "EXPO_PUBLIC_FEATURE_FLAGS_PROVIDER": "statsig",
        "EXPO_PUBLIC_STATSIG_CLIENT_KEY": "client-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
      }
    },
    "production": {
      "env": {
        "EXPO_PUBLIC_FEATURE_FLAGS_PROVIDER": "statsig",
        "EXPO_PUBLIC_STATSIG_CLIENT_KEY": "client-YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY"
      }
    }
  }
}
```

> **Por que a Client Key é `EXPO_PUBLIC_`?** O Client SDK Key do Statsig é projetado para ser público — ele identifica o projeto, não é um secret. Nunca use a Server SDK Key no cliente mobile.

---

## 3. Tipos e Contrato — `src/services/feature-flags/types.ts`

Defina **todos os nomes de flags válidos** em um único tipo. Isso garante type-safety em toda a aplicação — uma flag com nome errado é erro de compilação.

```typescript
// src/services/feature-flags/types.ts

/**
 * Catálogo de todas as feature flags do projeto.
 *
 * REGRA: Adicione aqui antes de usar em qualquer feature.
 * Nomenclatura: kebab-case, prefixo descritivo do domínio.
 *
 * Exemplos de lifecycle:
 * - Adiciona flag → implementa feature → rollout → remove flag + código antigo
 */
export type FeatureFlagName =
  | 'checkout-v2'            // Nova tela de checkout (rollout gradual)
  | 'new-home-layout'        // Redesign da home
  | 'dark-mode-beta'         // Dark mode em beta
  | 'offline-sync'           // Sincronização offline (kill switch)
  | 'push-notifications-v2'; // Nova implementação de push notifications

/**
 * Contrato que todo provider de feature flags deve implementar.
 * Ao trocar de Statsig para outro vendor, apenas a implementação muda —
 * o contrato segue o mesmo.
 */
export interface FeatureFlagsProvider {
  /**
   * Retorna o valor de uma feature flag para o usuário atual.
   * Sempre retorna um booleano — nunca lança exceção.
   */
  isEnabled(flagName: FeatureFlagName): boolean;

  /**
   * Atualiza o contexto do usuário no provider.
   * Chame após login/logout para que as flags reflitam o usuário correto.
   */
  setUser(user: FeatureFlagUser | null): Promise<void>;

  /**
   * Força a atualização das flags a partir do servidor.
   * Útil após setUser() para garantir que as flags sejam reavaliadas.
   */
  refresh(): Promise<void>;
}

/**
 * Atributos do usuário enviados ao provider para segmentação e rollout.
 * Apenas IDs e atributos não-sensíveis — nunca email ou CPF completo.
 */
export interface FeatureFlagUser {
  /** ID único do usuário no sistema */
  userId: string;
  /** Papel/permissão do usuário (ex: 'admin', 'user', 'beta-tester') */
  role?: string;
  /** País ou região para rollout geográfico */
  country?: string;
  /** Atributos customizados para targeting avançado */
  custom?: Record<string, string | number | boolean>;
}
```

---

## 4. Provider Statsig — `src/services/feature-flags/statsig-provider.ts`

```typescript
// src/services/feature-flags/statsig-provider.ts
import { StatsigClientExpo } from '@statsig/expo-bindings';
import { env } from '@config/env';
import { logger } from '@services/monitoring/logger';
import type { FeatureFlagsProvider, FeatureFlagName, FeatureFlagUser } from './types';

/**
 * Instância singleton do Statsig Client.
 * Exportada para uso fora do React (ex: interceptors, background tasks).
 */
export const statsigClient = new StatsigClientExpo(
  env.STATSIG_CLIENT_KEY,
  { userID: 'anonymous' }, // usuário inicial — atualizado após login
  {
    // networkConfig: personaliza endpoints se necessário (ex: proxy)
    // logLevel: 'warn', // reduz ruído de logs do SDK em produção
  },
);

/**
 * Implementação do FeatureFlagsProvider usando Statsig.
 * Esta classe isola todo o acoplamento ao SDK do Statsig.
 */
export class StatsigFeatureFlagsProvider implements FeatureFlagsProvider {
  isEnabled(flagName: FeatureFlagName): boolean {
    try {
      return statsigClient.checkGate(flagName);
    } catch (error) {
      // Em caso de falha (ex: SDK não inicializado), retorna false como safe default
      logger.warn('Falha ao verificar feature flag — retornando false como safe default', {
        flagName,
        error: String(error),
      });
      return false;
    }
  }

  async setUser(user: FeatureFlagUser | null): Promise<void> {
    try {
      const statsigUser = user
        ? {
            userID: user.userId,
            custom: {
              role: user.role ?? '',
              country: user.country ?? '',
              ...user.custom,
            },
          }
        : { userID: 'anonymous' };

      await statsigClient.updateUserAsync(statsigUser);
    } catch (error) {
      logger.warn('Falha ao atualizar usuário no Statsig', { error: String(error) });
    }
  }

  async refresh(): Promise<void> {
    // updateUserAsync já força reavaliação das flags — sem ação adicional necessária
  }
}
```

---

## 5. Provider Local — `src/services/feature-flags/local-provider.ts`

O provider local usa apenas o módulo `src/config/env.ts` — sem dependência de rede. É o fallback padrão em desenvolvimento para evitar dependência de internet/Statsig.

```typescript
// src/services/feature-flags/local-provider.ts
import type { FeatureFlagsProvider, FeatureFlagName, FeatureFlagUser } from './types';

/**
 * Mapa de flags locais — usado em development (EXPO_PUBLIC_FEATURE_FLAGS_PROVIDER=local).
 *
 * REGRA: Mantenha sincronizado com o tipo FeatureFlagName.
 * Flags marcadas como `true` ficam sempre ativas localmente.
 */
const LOCAL_FLAGS: Record<FeatureFlagName, boolean> = {
  'checkout-v2': true,
  'new-home-layout': false,
  'dark-mode-beta': true,
  'offline-sync': true,
  'push-notifications-v2': false,
};

/**
 * Implementação local — sem rede, sem SDK externo.
 * Útil para desenvolvimento e testes.
 */
export class LocalFeatureFlagsProvider implements FeatureFlagsProvider {
  isEnabled(flagName: FeatureFlagName): boolean {
    return LOCAL_FLAGS[flagName] ?? false;
  }

  // Métodos de usuário são no-op no provider local
  async setUser(_user: FeatureFlagUser | null): Promise<void> {}
  async refresh(): Promise<void> {}
}
```

---

## 6. Context e Provider — `src/services/feature-flags/feature-flags.context.tsx`

```typescript
// src/services/feature-flags/feature-flags.context.tsx
import React, { createContext, useContext, useRef, type ReactNode } from 'react';
import type { FeatureFlagsProvider, FeatureFlagName } from './types';

// ─── Context ─────────────────────────────────────────────────────────────────

const FeatureFlagsContext = createContext<FeatureFlagsProvider | null>(null);

/**
 * Hook interno para acessar o provider.
 * Não exponha este hook fora de `src/services/feature-flags/`.
 * Os consumers devem usar `useFeatureFlag()` de `src/hooks/`.
 */
export function useFeatureFlagsProvider(): FeatureFlagsProvider {
  const provider = useContext(FeatureFlagsContext);
  if (!provider) {
    throw new Error(
      'useFeatureFlagsProvider must be used inside <FeatureFlagsProvider>. ' +
        'Verifique se o provider está montado em src/app/_layout.tsx.',
    );
  }
  return provider;
}

// ─── Provider ─────────────────────────────────────────────────────────────────

interface FeatureFlagsProviderProps {
  provider: FeatureFlagsProvider;
  children: ReactNode;
}

/**
 * Provider raiz de feature flags.
 * Deve ser montado em `src/app/_layout.tsx`, acima do QueryClientProvider.
 */
export function FeatureFlagsProvider({ provider, children }: FeatureFlagsProviderProps) {
  // useRef garante que a referência do provider seja estável entre renders
  const providerRef = useRef(provider);

  return (
    <FeatureFlagsContext.Provider value={providerRef.current}>
      {children}
    </FeatureFlagsContext.Provider>
  );
}
```

---

## 7. Factory de Provider — `src/services/feature-flags/index.ts`

```typescript
// src/services/feature-flags/index.ts
import { StatsigClientExpo } from '@statsig/expo-bindings';
import { env } from '@config/env';
import { logger } from '@services/monitoring/logger';
import { StatsigFeatureFlagsProvider, statsigClient } from './statsig-provider';
import { LocalFeatureFlagsProvider } from './local-provider';
import type { FeatureFlagsProvider } from './types';

export { FeatureFlagsProvider as FeatureFlagsProviderComponent } from './feature-flags.context';
export type { FeatureFlagName, FeatureFlagUser } from './types';

let _providerInstance: FeatureFlagsProvider | null = null;

/**
 * Inicializa e retorna o provider de feature flags baseado na configuração do ambiente.
 *
 * Deve ser chamado ANTES do render do app (início de `src/app/_layout.tsx`).
 * Retorna sempre uma instância válida — em caso de falha do Statsig, usa o provider local.
 */
export async function initFeatureFlags(): Promise<FeatureFlagsProvider> {
  if (_providerInstance) return _providerInstance;

  if (env.FEATURE_FLAGS_PROVIDER === 'statsig') {
    try {
      // Inicializa o Statsig com await para garantir que as flags estejam prontas
      // antes do primeiro render (evita flash de conteúdo incorreto)
      await statsigClient.initializeAsync();
      logger.info('Statsig inicializado com sucesso', { env: env.APP_ENV });
      _providerInstance = new StatsigFeatureFlagsProvider();
    } catch (error) {
      // Fallback para provider local em caso de falha de inicialização
      logger.warn('Falha ao inicializar Statsig — usando provider local como fallback', {
        error: String(error),
      });
      _providerInstance = new LocalFeatureFlagsProvider();
    }
  } else {
    _providerInstance = new LocalFeatureFlagsProvider();
  }

  return _providerInstance;
}

/**
 * Retorna a instância do provider já inicializado.
 * Lança erro se `initFeatureFlags()` não foi chamado antes.
 */
export function getFeatureFlagsProvider(): FeatureFlagsProvider {
  if (!_providerInstance) {
    throw new Error(
      'Feature flags provider não inicializado. Chame initFeatureFlags() em _layout.tsx.',
    );
  }
  return _providerInstance;
}
```

---

## 8. Hook Público de Consumo — `src/hooks/use-feature-flag.ts`

**Este é o único ponto de contato das features com o sistema de feature flags.**

```typescript
// src/hooks/use-feature-flag.ts
import { useFeatureFlagsProvider } from '@services/feature-flags/feature-flags.context';
import type { FeatureFlagName } from '@services/feature-flags';

/**
 * Hook para verificar se uma feature flag está habilitada.
 *
 * Retorna `true` se a flag está ativa para o usuário atual.
 * Retorna `false` como safe default em caso de falha do provider.
 *
 * REGRA: Este é o ÚNICO hook que features devem usar.
 * Nunca importe Statsig, LocalProvider ou qualquer vendor diretamente nas features.
 *
 * @example
 * ```tsx
 * const isCheckoutV2Enabled = useFeatureFlag('checkout-v2');
 *
 * if (isCheckoutV2Enabled) {
 *   return <CheckoutV2Screen />;
 * }
 * return <CheckoutLegacyScreen />;
 * ```
 */
export function useFeatureFlag(flagName: FeatureFlagName): boolean {
  const provider = useFeatureFlagsProvider();
  return provider.isEnabled(flagName);
}
```

---

## 9. Montagem no Layout Raiz — `src/app/_layout.tsx`

```tsx
// src/app/_layout.tsx
import { useEffect, useState } from 'react';
import { Stack } from 'expo-router';
import { QueryClientProvider } from '@tanstack/react-query';
import * as Sentry from '@sentry/react-native';
import { queryClient } from '@services/query-client';
import { initSentry } from '@services/monitoring/sentry';
import {
  initFeatureFlags,
  FeatureFlagsProviderComponent,
} from '@services/feature-flags';
import type { FeatureFlagsProvider } from '@services/feature-flags';

// ✅ Inicializa o Sentry antes do render (ver skill monitoring-observability)
initSentry();

export function ErrorBoundary({ error, retry }: import('expo-router').ErrorBoundaryProps) {
  Sentry.captureException(error);
  return null; // ver skill error-handling para o fallback completo
}

function RootLayout() {
  const [featureFlagsProvider, setFeatureFlagsProvider] =
    useState<FeatureFlagsProvider | null>(null);

  useEffect(() => {
    // Inicializa feature flags de forma assíncrona no mount
    initFeatureFlags().then(setFeatureFlagsProvider);
  }, []);

  // Enquanto o provider não inicializa, exibe um loading mínimo
  // Evita flash de conteúdo com flags incorretas
  if (!featureFlagsProvider) {
    return null; // ou <SplashScreen /> se já tiver configurado
  }

  return (
    <FeatureFlagsProviderComponent provider={featureFlagsProvider}>
      <QueryClientProvider client={queryClient}>
        <Stack />
      </QueryClientProvider>
    </FeatureFlagsProviderComponent>
  );
}

export default Sentry.wrap(RootLayout);
```

> **Por que aguardar a inicialização?** Inicializar o Statsig com `initializeAsync()` garante que as flags estejam disponíveis antes do primeiro render, evitando **FOUC** (Flash of Uncorrect Content) — quando o usuário vê brevemente a UI sem a flag ativa e depois ela muda.

---

## 10. Consumo em Features — Padrão de Uso

### 10.1 Em hooks e telas (dentro do árvoré React)

```tsx
// src/features/checkout/screens/checkout-screen.tsx
import { useFeatureFlag } from '@hooks/use-feature-flag';
import { CheckoutV2Screen } from '../screens/checkout-v2-screen';
import { CheckoutLegacyScreen } from '../screens/checkout-legacy-screen';

/**
 * Ponto de entrada do checkout — delega para a versão correta baseado na flag.
 * Quando 'checkout-v2' estiver em 100%, remova este componente e use CheckoutV2Screen direto.
 */
export function CheckoutScreen() {
  const isCheckoutV2 = useFeatureFlag('checkout-v2');

  if (isCheckoutV2) {
    return <CheckoutV2Screen />;
  }

  return <CheckoutLegacyScreen />;
}
```

### 10.2 Em hooks de negócio

```tsx
// src/features/home/hooks/use-home-layout.ts
import { useFeatureFlag } from '@hooks/use-feature-flag';

export function useHomeLayout() {
  const isNewLayout = useFeatureFlag('new-home-layout');

  return {
    isNewLayout,
    // Usar para condicionar comportamentos, não apenas UI
    trackingPrefix: isNewLayout ? 'home_v2' : 'home_v1',
  };
}
```

### 10.3 Fora do React (serviços, interceptors)

Para verificar flags fora do React (raro), use `getFeatureFlagsProvider()`:

```typescript
// ✅ Quando absolutamente necessário fora do React
import { getFeatureFlagsProvider } from '@services/feature-flags';

export async function uploadFile(uri: string) {
  const ffProvider = getFeatureFlagsProvider();
  // Apenas se o flag existir no catálogo FeatureFlagName
  if (ffProvider.isEnabled('new-upload-service')) {
    return uploadWithNewService(uri);
  }
  return uploadWithLegacyService(uri);
}
```

> **Regra:** Prefira sempre `useFeatureFlag()` dentro de componentes/hooks. Use `getFeatureFlagsProvider()` apenas em código que genuinamente não pode estar no ciclo React.

---

## 11. Integração de Usuário — Após Login e Logout

Após o login, atualize o contexto do usuário no provider para que as flags reflitam o usuário real (e não `anonymous`):

```typescript
// src/features/auth/hooks/use-login.ts (adaptação conforme skill architect-react-native-features)
import { useMutation } from '@tanstack/react-query';
import { useRouter } from 'expo-router';
import { useAuthActions } from '@features/auth';
import { getFeatureFlagsProvider } from '@services/feature-flags';
import { setSentryUser } from '@services/monitoring/user-context';
import { loginApi } from '../services/auth.service';
import type { ApiError } from '@services/api/errors';
import type { LoginPayload, LoginResponse } from '../types';

export function useLogin() {
  const router = useRouter();
  const { setAuth } = useAuthActions();

  return useMutation<LoginResponse, ApiError, LoginPayload>({
    mutationFn: async (payload) => {
      const result = await loginApi(payload);
      if (result.isErr()) throw result.error;
      return result.value;
    },
    onSuccess: async ({ user, token }) => {
      setAuth(user, token);

      // ✅ Atualiza Sentry com o usuário logado (skill monitoring-observability)
      setSentryUser(user.id, user.role);

      // ✅ Atualiza o provider de feature flags com o usuário real
      // As flags serão reavaliadas para o usuário específico, habilitando
      // rollout baseado em userId, role, etc.
      const ffProvider = getFeatureFlagsProvider();
      await ffProvider.setUser({
        userId: user.id,
        role: user.role,
        custom: { country: user.country },
      });

      router.replace('/(home)');
    },
  });
}
```

```typescript
// src/features/auth/store/auth.store.ts — ação de logout
import { clearSentryUser } from '@services/monitoring/user-context';
import { getFeatureFlagsProvider } from '@services/feature-flags';

// Dentro da action logout:
logout: async () => {
  clearSentryUser();

  // ✅ Reseta o usuário no provider de feature flags para 'anonymous'
  const ffProvider = getFeatureFlagsProvider();
  await ffProvider.setUser(null);

  set({ ...initialState });
},
```

---

## 12. Rollout Gradual no Statsig Console

O rollout gradual é **configurado no painel do Statsig**, não no código. O código apenas consome a flag — o vendor decide quem recebe `true`.

### 12.1 Criando uma Feature Gate no Statsig

1. Acesse **Statsig Console → Feature Gates → Create**
2. Nomeie a flag igual ao valor em `FeatureFlagName` (ex: `checkout-v2`)
3. Configure as **Targeting Rules**:

```text
Regras recomendadas para rollout progressivo:

Regra 1 (mais restritiva primeiro):
  Condição: user.custom.role = 'admin' OR user.custom.role = 'beta-tester'
  Resultado: Pass (100%)

Regra 2:
  Condição: Rollout gradual (Partial Rollout)
  Percentual: 5%  → 10% → 25% → 50% → 100% (incrementa conforme confiança)
  Resultado: Pass

Regra 3 (padrão):
  Sem condição → Fail (0%) — todos os demais usuários não recebem a flag
```

### 12.2 Progressão do Rollout

| Fase | Público | Percentual | Critério para avançar |
|---|---|---|---|
| **Canary** | `role = 'admin'` | 100% internos | Zero crashes novos |
| **Beta** | `role = 'beta-tester'` | 100% beta | Métricas de negócio estáveis |
| **Gradual 5%** | Aleatório | 5% | 24-48h de monitoramento ok |
| **Gradual 25%** | Aleatório | 25% | Sentry OK, performance OK |
| **Gradual 50%** | Aleatório | 50% | Satisfação do usuário mantida |
| **Full** | Todos | 100% | Feature estável → remover flag |

### 12.3 Monitoramento durante rollout

No Statsig Console, monitore em **Metrics Explorer**:

```text
Verificar após cada aumento de percentual:
- Exposures (quantos usuários estão recebendo a flag)
- Error rate por variante (gate=true vs gate=false)
- Latência de carregamento das telas afetadas
- Métricas de negócio específicas (conversão, retenção, etc.)
```

> **Kill Switch imediato:** Se algo der errado, defina o percentual para **0%** diretamente no painel do Statsig. Nenhum deploy necessário. O app receberá `false` na próxima atualização de flags (geralmente em menos de 1 minuto).

---

## 13. Testes — Testando com Feature Flags

### 13.1 Mock do hook em testes de componentes

Nunca teste o provider real em unit/component tests. Use mocks do `useFeatureFlag`:

```typescript
// src/features/checkout/__tests__/checkout-screen.test.tsx
import React from 'react';
import { render, screen } from '@testing-library/react-native';
import { CheckoutScreen } from '../screens/checkout-screen';

// ✅ Mocka apenas o hook público — sem leak de implementação do Statsig
jest.mock('@hooks/use-feature-flag');
import { useFeatureFlag } from '@hooks/use-feature-flag';
const mockUseFeatureFlag = useFeatureFlag as jest.MockedFunction<typeof useFeatureFlag>;

describe('CheckoutScreen', () => {
  it('exibe CheckoutV2Screen quando flag checkout-v2 está ativa', () => {
    mockUseFeatureFlag.mockReturnValue(true);
    render(<CheckoutScreen />);
    expect(screen.getByTestId('checkout-v2-screen')).toBeTruthy();
  });

  it('exibe CheckoutLegacyScreen quando flag checkout-v2 está inativa', () => {
    mockUseFeatureFlag.mockReturnValue(false);
    render(<CheckoutScreen />);
    expect(screen.getByTestId('checkout-legacy-screen')).toBeTruthy();
  });
});
```

### 13.2 Provider de teste — `renderWithFlags()`

Para testes de integração que precisam de múltiplas flags:

```typescript
// src/utils/test-utils/render-with-flags.tsx
import React from 'react';
import { render } from '@testing-library/react-native';
import {
  FeatureFlagsProvider as FeatureFlagsProviderComponent,
} from '@services/feature-flags/feature-flags.context';
import { LocalFeatureFlagsProvider } from '@services/feature-flags/local-provider';
import type { FeatureFlagName } from '@services/feature-flags';

/**
 * Wrapper de teste que fornece feature flags controladas.
 * Sobrescreve apenas as flags especificadas — as demais usam o local-provider padrão.
 */
export function renderWithFlags(
  ui: React.ReactElement,
  flags: Partial<Record<FeatureFlagName, boolean>> = {},
) {
  const baseProvider = new LocalFeatureFlagsProvider();

  // Provider que sobrescreve flags específicas para o teste
  const testProvider = {
    isEnabled: (flagName: FeatureFlagName) =>
      flagName in flags ? flags[flagName]! : baseProvider.isEnabled(flagName),
    setUser: baseProvider.setUser.bind(baseProvider),
    refresh: baseProvider.refresh.bind(baseProvider),
  };

  return render(
    <FeatureFlagsProviderComponent provider={testProvider}>
      {ui}
    </FeatureFlagsProviderComponent>,
  );
}

// Uso:
// renderWithFlags(<HomeScreen />, { 'new-home-layout': true });
```

---

## 14. Estrutura de Arquivos

```text
src/
├── services/
│   └── feature-flags/
│       ├── types.ts                    ← FeatureFlagName, FeatureFlagsProvider interface, FeatureFlagUser
│       ├── statsig-provider.ts         ← Implementação Statsig (@statsig/expo-bindings)
│       ├── local-provider.ts           ← Implementação local (sem rede)
│       ├── feature-flags.context.tsx   ← React Context + FeatureFlagsProvider component
│       └── index.ts                    ← initFeatureFlags(), getFeatureFlagsProvider(), re-exports
│
├── hooks/
│   └── use-feature-flag.ts             ← Hook público (único ponto de acesso das features)
│
├── config/
│   └── env.ts                          ← FEATURE_FLAGS_PROVIDER, STATSIG_CLIENT_KEY (atualizado)
│
└── app/
    └── _layout.tsx                     ← Monta <FeatureFlagsProvider> + initFeatureFlags()
```

---

## 15. Padrões e Anti-Padrões

### ✅ Faça

```tsx
// ✅ Features usam apenas useFeatureFlag() — sem vendor leak
import { useFeatureFlag } from '@hooks/use-feature-flag';
const isNewLayout = useFeatureFlag('new-home-layout');

// ✅ Nomes de flags são validados em compile time (TypeScript)
// 'novo-checkout' → erro de tipo se não estiver em FeatureFlagName

// ✅ Safe default = false — features desativadas são o estado padrão
// O provider retorna false se a flag não existe ou o provider falha

// ✅ Teste com mock do hook, não do provider
jest.mock('@hooks/use-feature-flag');
mockUseFeatureFlag.mockReturnValue(true);

// ✅ Remova a flag e o código legado quando o rollout atingir 100%
// Tech debt = flags que ficam para sempre no código
```

### ❌ Evite

```tsx
// ❌ Features importando Statsig diretamente
import { StatsigClientExpo } from '@statsig/expo-bindings';
const isEnabled = statsigClient.checkGate('checkout-v2'); // acoplamento indevido

// ❌ Usar variáveis de ambiente diretamente como "flags"
if (env.APP_ENV === 'staging') { /* não é feature flag, é config de ambiente */ }

// ❌ Strings literais sem validação de tipo
useFeatureFlag('chekout-v2' as any); // typo não detectado

// ❌ Feature flags permanentes sem plano de remoção
// Toda flag deve ter um responsável e uma data estimada de remoção

// ❌ Lógica complexa de negócio dentro da condição de flag
if (useFeatureFlag('checkout-v2') && user.role === 'admin' && cart.items.length > 0) {
  // Mova esta lógica para um hook dedicado (use-checkout.ts)
}
```

---

## 16. Checklist de Verificação

### Setup Inicial
- [ ] `@statsig/expo-bindings` instalado com peerDeps (`expo-device`, `expo-application`, `@react-native-async-storage/async-storage`)?
- [ ] Variáveis de ambiente adicionadas ao `src/config/env.ts`?
- [ ] `.env.development`, `.env.staging` e `.env.production` com `EXPO_PUBLIC_FEATURE_FLAGS_PROVIDER` e `STATSIG_CLIENT_KEY`?
- [ ] `eas.json` atualizado com variáveis nos profiles `staging` e `production`?

### Tipagem e Contrato
- [ ] Todos os nomes de flags estão no tipo `FeatureFlagName`?
- [ ] `LOCAL_FLAGS` no local-provider está sincronizado com `FeatureFlagName`?
- [ ] Nenhuma feature importa Statsig ou providers diretamente?

### Montagem
- [ ] `initFeatureFlags()` é chamado antes do primeiro render em `_layout.tsx`?
- [ ] `<FeatureFlagsProviderComponent>` está acima do `<QueryClientProvider>` e do `<Stack>`?
- [ ] O app não renderiza enquanto o provider não está disponível (evita flash)?

### Integração de Usuário
- [ ] `ffProvider.setUser()` é chamado no `onSuccess` do login?
- [ ] `ffProvider.setUser(null)` é chamado no logout da store?

### Testes
- [ ] Componentes que usam flags têm testes para `flag=true` E `flag=false`?
- [ ] Testes usam `jest.mock('@hooks/use-feature-flag')` e não o provider real?
- [ ] `renderWithFlags()` está disponível em `src/utils/test-utils/` para testes de integração?

### Statsig Console
- [ ] Cada flag criada no painel tem o mesmo nome que em `FeatureFlagName`?
- [ ] Rollout gradual tem critérios claros para avançar de percentual?
- [ ] Existe um responsável e data estimada de remoção para cada flag ativa?
