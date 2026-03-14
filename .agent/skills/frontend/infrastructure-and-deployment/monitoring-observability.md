---
name: monitoring-observability
description: Observabilidade em produção para projetos Expo SDK 54+. Crash reporting com @sentry/react-native, logs estruturados com Sentry.logger, rastreamento de performance de telas via Expo Router, e alertas configuráveis por ambiente (development, staging, production).
---

# Monitoramento e Observabilidade — Expo SDK 54+

Você é um Engenheiro de Software Senior especialista em observabilidade para React Native. Sua responsabilidade é garantir que o projeto tenha **visibilidade completa sobre erros, performance e comportamento do usuário em produção**, usando as ferramentas corretas para cada camada, seguindo as melhores práticas da comunidade e compatível com o **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

> **Pré-requisitos:** Esta skill assume que as skills `environment-config`, `setup-react-native` e `error-handling` já foram configuradas. Os ambientes `development`, `staging` e `production` definidos no `eas.json` e `src/config/env.ts` são a fonte de verdade para configuração por ambiente do Sentry.

---

## Quando usar esta skill

- **Novo projeto:** Configurar Sentry e observabilidade do zero.
- **Crash reporting:** Capturar crashes nativos e erros JavaScript em produção.
- **Logs estruturados:** Substituir `console.log` por logs rastreáveis e consultáveis.
- **Performance de telas:** Instrumentar navegação e rastrear TTID/TTFD.
- **Alertas:** Configurar thresholds de erro e performance por ambiente.
- **Integração com CI/CD:** Fazer upload de source maps para depuração minificada.

---

## 1. Visão Geral da Estratégia

```text
┌──────────────────────────────────────────────────────────────────┐
│                CAMADAS DE OBSERVABILIDADE                        │
├──────────────────────────────────────────────────────────────────┤
│  Crashes (nativo + JS)  → @sentry/react-native (nativeCrash)     │
│  Erros de renderização  → Error Boundary (error-handling skill)  │
│                           → Sentry.captureException no onError    │
│  Logs estruturados      → Sentry.logger (trace/debug/info/warn/  │
│                           error/fatal) + severity + attributes    │
│  Performance de telas   → reactNativeTracingIntegration          │
│                           + Expo Router instrumentation           │
│  Alertas por ambiente   → Sentry Alerts (rate, threshold)        │
└──────────────────────────────────────────────────────────────────┘
```

| Camada | Ferramenta | O que captura |
|---|---|---|
| **Crash nativo** | `@sentry/react-native` nativo | ANR, OOM, crashes de C++/Swift |
| **Erro JS** | `Sentry.captureException` | Exceções não tratadas, `Error Boundary.onError` |
| **Log estruturado** | `Sentry.logger` | Eventos rastreáveis com atributos pesquisáveis |
| **Performance de rota** | `reactNativeTracingIntegration` | TTID, TTFD, tempo de transição por tela |
| **Performance HTTP** | `Sentry.startSpan` em `api/client.ts` | Latência por endpoint, falhas de rede |
| **Alertas** | Sentry Alerts Rules | Rate de erro acima de threshold por ambiente |

> **Nota:** O pacote `sentry-expo` foi **descontinuado** a partir do Expo SDK 50. Use exclusivamente `@sentry/react-native` diretamente.

---

## 2. Instalação e Configuração

### 2.1 Instalação

O método recomendado é o **Sentry Wizard**, que configura automaticamente Metro, Babel, source maps e `app.config.ts`:

```bash
# Opção 1 (recomendada): Wizard automatiza tudo
npx @sentry/wizard@latest -i reactNative

# Opção 2: Instalação manual
npx expo install @sentry/react-native
```

> **Por que usar o Wizard?** Ele configura o upload de source maps para EAS Build, modifica `metro.config.js`, `app.config.ts` e `babel.config.js` automaticamente. Economiza horas de configuração manual e evita erros comuns.

### 2.2 Variáveis de Ambiente

Adicione ao `src/config/env.ts` (conforme a skill `environment-config`):

```typescript
// src/config/env.ts — adicionar as variáveis do Sentry

// ─── Sentry ──────────────────────────────────────────────────────────────────
// O DSN é público (EXPO_PUBLIC_) — identifica o projeto no Sentry, não é secret
export const SENTRY_DSN = process.env.EXPO_PUBLIC_SENTRY_DSN ?? '';

// ─── Amostragem de performance (ajustável por ambiente) ──────────────────────
export const SENTRY_TRACES_SAMPLE_RATE =
  Number(process.env.EXPO_PUBLIC_SENTRY_TRACES_SAMPLE_RATE) ||
  (APP_ENV === 'production' ? 0.1 : 1.0);

export const SENTRY_PROFILES_SAMPLE_RATE =
  Number(process.env.EXPO_PUBLIC_SENTRY_PROFILES_SAMPLE_RATE) ||
  (APP_ENV === 'production' ? 0.1 : 1.0);

// ─── Export consolidado (adicionar ao objeto `env`) ──────────────────────────
export const env = {
  // ... campos existentes
  SENTRY_DSN,
  SENTRY_TRACES_SAMPLE_RATE,
  SENTRY_PROFILES_SAMPLE_RATE,
} as const;
```

Adicione às variáveis dos arquivos `.env.*` e no `eas.json`:

```env
# .env.development
EXPO_PUBLIC_SENTRY_DSN=https://examplePublicKey@o0.ingest.sentry.io/0
EXPO_PUBLIC_SENTRY_TRACES_SAMPLE_RATE=1.0
EXPO_PUBLIC_SENTRY_PROFILES_SAMPLE_RATE=1.0

# .env.staging
EXPO_PUBLIC_SENTRY_DSN=https://examplePublicKey@o0.ingest.sentry.io/0
EXPO_PUBLIC_SENTRY_TRACES_SAMPLE_RATE=0.5
EXPO_PUBLIC_SENTRY_PROFILES_SAMPLE_RATE=0.5

# .env.production
EXPO_PUBLIC_SENTRY_DSN=https://examplePublicKey@o0.ingest.sentry.io/0
EXPO_PUBLIC_SENTRY_TRACES_SAMPLE_RATE=0.1
EXPO_PUBLIC_SENTRY_PROFILES_SAMPLE_RATE=0.1
```

Adicione `SENTRY_AUTH_TOKEN` como **EAS Secret** (não é `EXPO_PUBLIC_` — é sensível e usado apenas em build time para upload de source maps):

```bash
eas secret:create --name SENTRY_AUTH_TOKEN --value "sntrys_..." --scope project
```

> **Tabela de classificação de variáveis Sentry:**
>
> | Variável | Prefixo | Por quê? |
> |---|---|---|
> | `SENTRY_DSN` | `EXPO_PUBLIC_` | Identificador público do projeto — não é um secret |
> | `SENTRY_TRACES_SAMPLE_RATE` | `EXPO_PUBLIC_` | Valor de configuração público |
> | `SENTRY_AUTH_TOKEN` | ❌ Sem prefixo (EAS Secret) | Token de API do Sentry — secret de build |

### 2.3 `app.config.ts`

O Wizard configura automaticamente. Verifique se o `app.config.ts` contém:

```typescript
// app.config.ts
import type { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  // ... configs existentes
  plugins: [
    // Adiciona o plugin do Sentry para configurar source maps nativos
    [
      '@sentry/react-native/expo',
      {
        url: 'https://sentry.io/',
        // O auth token é lido do ambiente (EAS Secret em build, .env.local em dev)
        authToken: process.env.SENTRY_AUTH_TOKEN,
        project: 'meu-projeto',
        organization: 'minha-org',
      },
    ],
  ],
});
```

### 2.4 `metro.config.js`

O Wizard também adiciona o serializer do Sentry ao Metro. Verifique:

```js
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const { withSentryConfig } = require('@sentry/react-native/metro');

/** @type {import('expo/metro-config').MetroConfig} */
const config = getDefaultConfig(__dirname);

// ─── Suporte a SVG (da skill setup-react-native) ──────────────────────────────
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

// ─── Sentry — adiciona serializer para upload de source maps ─────────────────
module.exports = withSentryConfig(config);
```

---

## 3. Inicialização do Sentry

### 3.1 Arquivo de inicialização centralizado — `src/services/monitoring/sentry.ts`

```typescript
// src/services/monitoring/sentry.ts
import * as Sentry from '@sentry/react-native';
import { env } from '@config/env';

// Guarda a referência para integração com Expo Router
export const sentryRoutingInstrumentation = Sentry.reactNavigationIntegration({
  enableTimeToInitialDisplay: true, // Rastreia TTID por rota
});

/**
 * Inicializa o Sentry. Deve ser chamado ANTES do `registerRootComponent`.
 * Em Expo Router, chame no início de `src/app/_layout.tsx`.
 */
export function initSentry(): void {
  // Não inicializa em development local para não poluir o Sentry com dados de dev
  // Remover esta guarda se quiser testar o fluxo completo em dev
  if (env.isDev) {
    return;
  }

  Sentry.init({
    dsn: env.SENTRY_DSN,

    // ─── Ambiente e Release ───────────────────────────────────────────────────
    environment: env.APP_ENV, // 'development' | 'staging' | 'production'
    // `release` é configurado automaticamente pelo Wizard (usa appVersion + buildNumber)

    // ─── Performance ─────────────────────────────────────────────────────────
    tracesSampleRate: env.SENTRY_TRACES_SAMPLE_RATE,
    profilesSampleRate: env.SENTRY_PROFILES_SAMPLE_RATE,
    enableAutoPerformanceTracing: true,
    enableAppStartTracking: true,      // Cold/warm start
    enableNativeFramesTracking: true,  // Frames lentos e congelados
    enableStallTracking: true,         // JS thread stalls

    // ─── Session Replay (produção) ────────────────────────────────────────────
    // Grava replay de sessões para depuração visual de crashes
    replaysSessionSampleRate: env.isProd ? 0.05 : 0,   // 5% das sessões em prod
    replaysOnErrorSampleRate: env.isProd ? 1.0 : 0,    // 100% das sessões com erro

    // ─── Crash nativo ────────────────────────────────────────────────────────
    enableNative: true,
    enableNativeCrashHandling: true,
    enableNdk: true, // Android NDK crash reporting

    // ─── Logs estruturados ────────────────────────────────────────────────────
    enableLogs: true, // Habilita Sentry.logger API

    // ─── Integrações ─────────────────────────────────────────────────────────
    integrations: [
      // Rastreamento de navegação com Expo Router
      Sentry.reactNativeTracingIntegration({
        enableTimeToInitialDisplay: true,
        routeChangeTimeoutMs: 1000,
        ignoreEmptyBackNavigationTransactions: true,
      }),
      sentryRoutingInstrumentation,

      // Session Replay — mascarar dados sensíveis
      Sentry.mobileReplayIntegration({
        maskAllText: true,   // Mascara todo texto (evita expor dados sensíveis)
        maskAllImages: true, // Mascara imagens
      }),
    ],

    // ─── Privacidade e filtros ────────────────────────────────────────────────
    beforeSend: (event) => {
      // Nunca envia dados de usuário sem IDs (sessões anônimas são aceitas)
      if (event.user?.email) {
        // Remove e-mail para conformidade com LGPD/GDPR
        delete event.user.email;
      }
      return event;
    },
  });
}
```

### 3.2 Integração com Expo Router — `src/app/_layout.tsx`

```tsx
// src/app/_layout.tsx
import * as Sentry from '@sentry/react-native';
import { Stack } from 'expo-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@services/query-client';
import { initSentry, sentryRoutingInstrumentation } from '@services/monitoring/sentry';

// ✅ Inicializa antes do render do componente
initSentry();

export function ErrorBoundary({ error, retry }: import('expo-router').ErrorBoundaryProps) {
  // Reporta erros de renderização globais ao Sentry
  Sentry.captureException(error);
  return (
    // ... componente de fallback (ver skill error-handling)
  );
}

function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <Stack />
    </QueryClientProvider>
  );
}

// ✅ Wrap com Sentry.wrap para rastreamento de erros JavaScript globais
export default Sentry.wrap(RootLayout);
```

> **Por que `Sentry.wrap`?** O `Sentry.wrap` adiciona o handler de erros JavaScript não capturados e conecta o React Error Boundary global do Expo Router ao Sentry automaticamente.

---

## 4. Identificação de Usuário

Defina o contexto de usuário no Sentry logo após o login bem-sucedido. Isso permite filtrar erros por usuário no painel do Sentry.

```typescript
// src/services/monitoring/user-context.ts
import * as Sentry from '@sentry/react-native';

/**
 * Define o contexto do usuário no Sentry após login.
 * Chame em `useLogin.onSuccess` ou no hook de autenticação.
 *
 * ⚠️ LGPD/GDPR: Nunca envie e-mail completo, CPF ou dados sensíveis.
 * Use apenas ID interno e role.
 */
export function setSentryUser(userId: string, role?: string): void {
  Sentry.setUser({
    id: userId,
    // username: não usar nome completo — use apenas ID ou hash
  });

  if (role) {
    Sentry.setTag('user.role', role);
  }
}

/**
 * Limpa o contexto do usuário no Sentry após logout.
 * Chame em `useAuthStore.logout`.
 */
export function clearSentryUser(): void {
  Sentry.setUser(null);
}
```

```typescript
// src/features/auth/hooks/use-login.ts (adaptação)
import { setSentryUser } from '@services/monitoring/user-context';

// No onSuccess do useMutation:
onSuccess: ({ user, token }) => {
  setAuth(user, token);
  setSentryUser(user.id, user.role); // ← define contexto no Sentry
  queryClient.resetQueries();
},
```

```typescript
// src/features/auth/store/auth.store.ts (logout)
import { clearSentryUser } from '@services/monitoring/user-context';

logout: () => {
  clearSentryUser(); // ← limpa contexto no Sentry
  set({ user: null, token: null });
},
```

---

## 5. Logs Estruturados com `Sentry.logger`

O `Sentry.logger` substitui o `console.log` para logs que precisam de rastreabilidade em produção. Os logs aparecem na aba **Logs** do Sentry e são pesquisáveis por atributos.

### 5.1 Níveis de log

| Método | Quando usar |
|---|---|
| `Sentry.logger.trace` | Fluxo detalhado (apenas dev) |
| `Sentry.logger.debug` | Debug de lógica interna |
| `Sentry.logger.info` | Eventos de negócio importantes |
| `Sentry.logger.warn` | Situações inesperadas mas não críticas |
| `Sentry.logger.error` | Erros recuperáveis |
| `Sentry.logger.fatal` | Erros críticos que comprometem a session |

### 5.2 Logger centralizado — `src/services/monitoring/logger.ts`

Crie um wrapper sobre o `Sentry.logger` para controlar o comportamento por ambiente:

```typescript
// src/services/monitoring/logger.ts
import * as Sentry from '@sentry/react-native';
import { env } from '@config/env';

type LogLevel = 'trace' | 'debug' | 'info' | 'warn' | 'error' | 'fatal';
type LogAttributes = Record<string, string | number | boolean>;

/**
 * Logger estruturado centralizado.
 *
 * Em development: loga no console + envia ao Sentry (se isDev guard removido)
 * Em staging/production: envia ao Sentry com atributos pesquisáveis
 *
 * REGRA: Use este logger em vez de `console.log` em toda a aplicação.
 * O `no-console` do ESLint deve reportar `console.log/debug/info` como erro.
 */
function log(level: LogLevel, message: string, attributes?: LogAttributes): void {
  if (env.isDev) {
    // Em dev, espelha no console para facilitar o desenvolvimento
    const consoleMethod = level === 'fatal' || level === 'error' ? console.error : console.warn;
    consoleMethod(`[${level.toUpperCase()}] ${message}`, attributes ?? '');
    return;
  }

  // Em staging/production, envia ao Sentry com atributos estruturados
  const logFn = Sentry.logger[level];
  if (attributes) {
    logFn(Sentry.logger.fmt`${message}`, attributes);
  } else {
    logFn(message);
  }
}

export const logger = {
  trace: (message: string, attributes?: LogAttributes) => log('trace', message, attributes),
  debug: (message: string, attributes?: LogAttributes) => log('debug', message, attributes),
  info: (message: string, attributes?: LogAttributes) => log('info', message, attributes),
  warn: (message: string, attributes?: LogAttributes) => log('warn', message, attributes),
  error: (message: string, attributes?: LogAttributes) => log('error', message, attributes),
  fatal: (message: string, attributes?: LogAttributes) => log('fatal', message, attributes),
} as const;
```

### 5.3 Exemplos de uso

```typescript
import { logger } from '@services/monitoring/logger';

// ✅ Log de evento de negócio com atributos pesquisáveis
logger.info('Usuário realizou checkout', {
  userId: user.id,
  orderId: order.id,
  totalItems: cart.items.length,
  paymentMethod: 'credit_card',
});

// ✅ Log de erro recuperável (não lança exceção)
logger.warn('Cache de produtos expirado — forçando refetch', {
  cacheAge: staleness,
  feature: 'products',
});

// ✅ Log de erro com captura de exceção
try {
  await syncOfflineQueue();
} catch (error) {
  logger.error('Falha ao sincronizar fila offline', {
    queueSize: queue.length,
    retryCount: retries,
  });
  Sentry.captureException(error); // Captura o stack trace completo
}

// ❌ ERRADO — não use console.log em produção
console.log('Produto carregado:', product); // Invisível no Sentry
```

### 5.4 Atualizar regra do ESLint

Atualize o `eslint.config.js` para proibir `console.log` e `console.info` (permitindo apenas `warn` e `error` para casos legítimos de CLI):

```js
// eslint.config.js — regra já existente na skill setup-react-native
rules: {
  // Proíbe console.log e console.info — use logger.info/logger.debug
  'no-console': ['error', { allow: ['warn', 'error'] }],
}
```

---

## 6. Rastreamento de Performance de Telas

### 6.1 Métricas rastreadas automaticamente

Com `reactNativeTracingIntegration` configurado na seção 3.1, o Sentry captura automaticamente:

| Métrica | Descrição | Valor alvo |
|---|---|---|
| **TTID** (Time to Initial Display) | Tempo até a primeira renderização da tela | < 300ms |
| **TTFD** (Time to Full Display) | Tempo até os dados estarem completamente carregados | < 1000ms |
| **Frame Rate** | Frames lentos (> 16ms) e congelados (> 700ms) | 0 frozen frames |
| **App Cold Start** | Tempo de inicialização do zero | < 2000ms |
| **App Warm Start** | Retorno do background | < 500ms |

### 6.2 Span manual para operações críticas

Para operações que não são detectadas automaticamente, crie spans manuais:

```typescript
// Exemplo: rastrear carregamento de dados pesados
import * as Sentry from '@sentry/react-native';

async function loadDashboardData(userId: string) {
  return await Sentry.startSpan(
    {
      name: 'dashboard.load-data',
      op: 'function',
      attributes: { 'user.id': userId },
    },
    async () => {
      const [products, orders] = await Promise.all([
        getProducts(),
        getOrders(userId),
      ]);
      return { products, orders };
    },
  );
}
```

```typescript
// Exemplo: rastrear upload de arquivo
import * as Sentry from '@sentry/react-native';

async function uploadProfilePhoto(uri: string, userId: string) {
  const span = Sentry.startInactiveSpan({
    name: 'profile.upload-photo',
    op: 'file.upload',
    attributes: { 'user.id': userId },
  });

  try {
    const result = await uploadToStorage(uri);
    Sentry.setMeasurement('file_size_bytes', result.size, 'byte');
    span?.setStatus({ code: 1 }); // OK
    return result;
  } catch (error) {
    span?.setStatus({ code: 2 }); // ERROR
    throw error;
  } finally {
    span?.end();
  }
}
```

### 6.3 Rastreamento de TTFD manual (telas com carregamento assíncrono)

Para telas que carregam dados via `useQuery`, marque explicitamente quando o conteúdo principal ficou disponível:

```tsx
// src/features/home/screens/home-screen.tsx
import * as Sentry from '@sentry/react-native';
import { useEffect } from 'react';
import { useProducts } from '../queries/use-products';

export function HomeScreen() {
  const { data: products, isSuccess } = useProducts();

  useEffect(() => {
    if (isSuccess) {
      // Sinaliza ao Sentry que o conteúdo principal foi carregado (TTFD)
      Sentry.reportFullyDisplayed();
    }
  }, [isSuccess]);

  // ... resto da tela
}
```

---

## 7. Integração com Error Boundaries

A skill `error-handling` define o `FeatureErrorBoundary` com um placeholder para o Sentry. Substitua-o pela chamada real:

```tsx
// src/components/feature-error-boundary.tsx — atualização da skill error-handling
import { ErrorBoundary, FallbackProps } from 'react-error-boundary';
import * as Sentry from '@sentry/react-native';
import { logger } from '@services/monitoring/logger';

export function FeatureErrorBoundary({ children, onError }: FeatureErrorBoundaryProps) {
  return (
    <ErrorBoundary
      FallbackComponent={FeatureFallback}
      onError={(error, info) => {
        // ✅ Captura o erro no Sentry com o component stack
        Sentry.captureException(error, {
          extra: { componentStack: info.componentStack },
        });

        // ✅ Log estruturado para rastreabilidade
        logger.error('Erro de renderização em FeatureErrorBoundary', {
          errorMessage: error.message,
          errorName: error.name,
        });

        onError?.(error);
      }}
    >
      {children}
    </ErrorBoundary>
  );
}
```

### Captura em `onError` de mutations (TanStack Query)

```typescript
// Dentro de qualquer useMutation — integração com a skill error-handling
import * as Sentry from '@sentry/react-native';
import { logger } from '@services/monitoring/logger';

onError: (error: ApiError) => {
  // Erros esperados (validação, não-autorizado) → não enviar ao Sentry
  if (error.type === 'VALIDATION_ERROR' || error.type === 'UNAUTHORIZED') {
    logger.warn('Erro esperado na mutation', { errorType: error.type });
    return;
  }

  // Erros inesperados → capturar no Sentry com contexto
  logger.error('Erro inesperado na mutation', { errorType: error.type });
  Sentry.captureException(new Error(error.message), {
    tags: { errorType: error.type },
  });
},
```

---

## 8. Configuração por Ambiente

### 8.1 Tabela de configuração por ambiente

| Configuração | Development | Staging | Production |
|---|---|---|---|
| `initSentry()` executado? | ❌ Não (guarda `isDev`) | ✅ Sim | ✅ Sim |
| `tracesSampleRate` | N/A | `0.5` (50%) | `0.1` (10%) |
| `profilesSampleRate` | N/A | `0.5` (50%) | `0.1` (10%) |
| `replaysSessionSampleRate` | N/A | `0` | `0.05` (5%) |
| `replaysOnErrorSampleRate` | N/A | `0` | `1.0` (100%) |
| Filtrar por `environment` | N/A | `staging` | `production` |

> **Por que não inicializar em development?** Evita poluir o Sentry com dados de desenvolvimento e reduz ruído nos alertas. Para testar o fluxo completo localmente, remova temporariamente a guarda `if (env.isDev)` em `initSentry()`.

### 8.2 Filtrar por ambiente no Sentry

No painel do Sentry, use o filtro `environment:production` para ver apenas erros de produção. Crie **Saved Searches** separados:

- `environment:production is:unresolved` → Issues de produção não resolvidas
- `environment:staging is:unresolved` → Issues de staging

### 8.3 Scripts `package.json`

```json
{
  "scripts": {
    "sentry:upload-sourcemaps": "sentry-expo-upload-sourcemaps dist",
    "sentry:doctor": "npx @sentry/wizard -i reactNative --no-install"
  }
}
```

---

## 9. Alertas no Sentry

Configure alertas no painel em **Alerts → Alert Rules**. Regras recomendadas por ambiente:

### 9.1 Alertas para `production`

| Alert | Condição | Threshold | Canal |
|---|---|---|---|
| **High Error Rate** | `event.count()` para `environment:production` | > 10/min | Slack `#incidents` |
| **New Issue** | Novo issue em `environment:production` | Qualquer | E-mail + Slack |
| **Crash Rate** | `crash_rate()` | > 1% das sessões | PagerDuty / E-mail urgente |
| **Slow Transactions** | p95 de qualquer transação de tela | > 2000ms | Slack `#performance` |
| **ANR Rate** | `anr_rate()` (Android) | > 0.5% | E-mail |

### 9.2 Alertas para `staging`

| Alert | Condição | Threshold | Canal |
|---|---|---|---|
| **Any New Issue** | Novo issue em `environment:staging` | Qualquer | Slack `#qa` |
| **High Volume** | `event.count()` para `environment:staging` | > 50/min | Slack `#qa` |

### 9.3 Configuração de Alert via Código (Sentry CLI)

Para ambientes com IaC, crie alertas programaticamente:

```bash
# Criar regra de alerta via Sentry CLI
sentry-cli --auth-token $SENTRY_AUTH_TOKEN \
  monitors create \
  --project meu-projeto \
  --org minha-org
```

> **Recomendação:** Configure alertas pelo painel web do Sentry na primeira vez. Documente as regras criadas no README do projeto.

---

## 10. Badges e Breadcrumbs customizados

### 10.1 Breadcrumbs de navegação

O `sentryRoutingInstrumentation` já captura breadcrumbs de navegação automaticamente. Para eventos de domínio importantes, adicione breadcrumbs manualmente:

```typescript
import * as Sentry from '@sentry/react-native';

// Adicionar breadcrumb em evento importante de negócio
Sentry.addBreadcrumb({
  category: 'business',
  message: 'Usuário adicionou produto ao carrinho',
  level: 'info',
  data: {
    productId: product.id,
    productName: product.name,
    cartSize: cart.items.length,
  },
});
```

### 10.2 Tags por feature

Use tags para agrupar erros por feature no Sentry:

```typescript
// Em qualquer hook ou serviço de uma feature específica
Sentry.withScope((scope) => {
  scope.setTag('feature', 'checkout');
  scope.setTag('step', 'payment');
  scope.setContext('order', { orderId: order.id, total: order.total });
  Sentry.captureException(paymentError);
});
```

---

## 11. Integração com CI/CD

A skill `ci-cd-pipeline` define os workflows de GitHub Actions. Para integrar o Sentry, adicione o upload de source maps ao workflow de release:

```yaml
# .github/workflows/release.yml — adicionar step após o EAS Build
- name: Upload Source Maps ao Sentry
  if: success()
  run: |
    npx @sentry/react-native-cli upload-dsym \
      --auth-token ${{ secrets.SENTRY_AUTH_TOKEN }} \
      --org minha-org \
      --project meu-projeto
  env:
    SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
```

Adicione `SENTRY_AUTH_TOKEN` aos **GitHub Secrets** do repositório (além do EAS Secret):

```text
Settings → Secrets and variables → Actions → New repository secret
Name: SENTRY_AUTH_TOKEN
Value: sntrys_...
```

---

## 12. Estrutura de Arquivos

```text
src/
├── services/
│   └── monitoring/
│       ├── sentry.ts              ← initSentry() + sentryRoutingInstrumentation
│       ├── logger.ts              ← Logger centralizado (Sentry.logger)
│       └── user-context.ts        ← setSentryUser / clearSentryUser
├── config/
│   └── env.ts                     ← SENTRY_DSN, SENTRY_TRACES_SAMPLE_RATE (atualizado)
└── app/
    └── _layout.tsx                ← initSentry() + Sentry.wrap(RootLayout)
```

---

## 13. Padrões e Anti-Padrões

### ✅ Faça

```typescript
// ✅ Use logger.info/warn/error em vez de console.*
logger.info('Pedido finalizado', { orderId, total });

// ✅ Capture exceções inesperadas com contexto
Sentry.captureException(error, { tags: { feature: 'checkout' } });

// ✅ Identifique o usuário após login
setSentryUser(user.id, user.role);

// ✅ Limpe o contexto no logout
clearSentryUser();

// ✅ Adicione breadcrumbs em eventos de negócio críticos
Sentry.addBreadcrumb({ category: 'business', message: 'Checkout iniciado', level: 'info' });

// ✅ Use spans para operações de longa duração
await Sentry.startSpan({ name: 'sync.offline-queue', op: 'task' }, syncQueue);

// ✅ Marque TTFD quando os dados principais carregarem
useEffect(() => { if (isSuccess) Sentry.reportFullyDisplayed(); }, [isSuccess]);
```

### ❌ Evite

```typescript
// ❌ Nunca envie dados sensíveis ao Sentry
Sentry.setUser({ email: user.email, cpf: user.cpf }); // viola LGPD

// ❌ Não use console.log em código de produção
console.log('Debug:', data); // não aparece no Sentry

// ❌ Não capture todos os erros indiscriminadamente
Sentry.captureException(error); // sem contexto, sem discriminação de tipo

// ❌ Não inicialize o Sentry múltiplas vezes
// initSentry() deve ser chamado apenas uma vez (em _layout.tsx)

// ❌ Não use sentry-expo (pacote deprecated)
import * as Sentry from 'sentry-expo'; // ❌ deprecated desde SDK 50
```

---

## 14. Checklist de Verificação

### Instalação e Configuração
- [ ] `@sentry/react-native` instalado via Wizard ou `npx expo install`?
- [ ] Plugin `@sentry/react-native/expo` configurado no `app.config.ts`?
- [ ] `withSentryConfig` adicionado ao `metro.config.js`?
- [ ] `SENTRY_AUTH_TOKEN` criado como EAS Secret (`eas secret:create`)?
- [ ] `EXPO_PUBLIC_SENTRY_DSN` adicionado aos arquivos `.env.*`?
- [ ] `SENTRY_AUTH_TOKEN` adicionado aos GitHub Secrets?

### Inicialização
- [ ] `initSentry()` chamado no início de `src/app/_layout.tsx`?
- [ ] `Sentry.wrap(RootLayout)` aplicado?
- [ ] Guarda `if (env.isDev)` presente em `initSentry()`?
- [ ] `environment` do Sentry vindo de `env.APP_ENV` (não hardcoded)?

### Performance
- [ ] `sentryRoutingInstrumentation` registrado nas integrações?
- [ ] `tracesSampleRate` com valor adequado por ambiente (dev=N/A, staging=0.5, prod=0.1)?
- [ ] `enableNativeFramesTracking: true` configurado?
- [ ] `Sentry.reportFullyDisplayed()` chamado nas telas com carregamento assíncrono?

### Logs e Contexto
- [ ] `enableLogs: true` no `Sentry.init()`?
- [ ] Logger centralizado em `src/services/monitoring/logger.ts` criado?
- [ ] `setSentryUser` chamado no `onSuccess` do login?
- [ ] `clearSentryUser` chamado no logout?
- [ ] Regra `no-console: error` ativa no `eslint.config.js`?
- [ ] Dados sensíveis (e-mail, CPF, senhas) nunca enviados ao Sentry?

### Error Boundaries
- [ ] `FeatureErrorBoundary.onError` chama `Sentry.captureException`?
- [ ] `onError` do `_layout.tsx` (ErrorBoundary global) chama `Sentry.captureException`?
- [ ] `useMutation.onError` diferencia erros esperados (sem captura) de inesperados (com captura)?

### Alertas
- [ ] Alert de "High Error Rate" configurado para `environment:production`?
- [ ] Alert de "New Issue" configurado para `environment:production`?
- [ ] Alert para `environment:staging` configurado no canal `#qa`?
- [ ] Crash rate thresholds definidos?
