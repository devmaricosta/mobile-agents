---
name: security-patterns
description: Boas práticas de segurança para Expo SDK 54+. Armazenamento seguro de tokens com expo-secure-store, certificate pinning com react-native-ssl-public-key-pinning, prevenção de leaks em logs, e checklist de segurança antes do release.
---

# Padrões de Segurança — Expo SDK 54+

Você é um Engenheiro de Software Senior especialista em segurança mobile. Sua responsabilidade é garantir que o projeto adote uma postura **segura por padrão (Secure by Design)**, protegendo dados do usuário, comunicações de rede e o código em produção, seguindo as melhores práticas da comunidade e compatível com o **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

> **Pré-requisitos:** Esta skill assume que as skills `environment-config`, `setup-react-native`, `monitoring-observability` e `architect-react-native-features` já foram configuradas. O logger centralizado em `src/services/monitoring/logger.ts` e o módulo `src/config/env.ts` são usados como base nesta skill.

---

## Quando usar esta skill

- **Novo projeto:** Configurar armazenamento seguro, pinning e prevenção de leaks do zero.
- **Autenticação:** Armazenar tokens de acesso e refresh tokens de forma segura.
- **Comunicação segura:** Implementar certificate pinning para APIs críticas.
- **Code review:** Validar que logs e erros não expõem dados sensíveis.
- **Pré-release:** Executar o checklist de segurança antes de submeter às stores.

---

## 1. Visão Geral da Estratégia de Segurança

```text
┌─────────────────────────────────────────────────────────────────┐
│              CAMADAS DE SEGURANÇA                               │
├─────────────────────────────────────────────────────────────────┤
│  Armazenamento    → expo-secure-store (Keychain/Keystore)       │
│  Comunicação      → HTTPS + Certificate Pinning (public key)    │
│  Logs             → logger centralizado — sem dados sensíveis   │
│  Variáveis        → EAS Secrets + EXPO_PUBLIC_ bem delimitado   │
│  Build            → ProGuard (Android) + Hermes obfuscation     │
│  Dependências     → npm audit + atualizações periódicas         │
└─────────────────────────────────────────────────────────────────┘
```

| Camada | Ferramenta | O que protege |
|---|---|---|
| **Dados em repouso** | `expo-secure-store` | Tokens, refresh tokens, credenciais |
| **Dados em trânsito** | HTTPS + certificate pinning | Ataques MITM (Man-in-the-Middle) |
| **Logs** | `logger` centralizado + regras ESLint | Tokens, CPF, e-mail, senhas em logs |
| **Segredos de build** | EAS Secrets | API keys privadas, tokens de CI/CD |
| **Código** | Hermes + ProGuard | Engenharia reversa do bundle |

---

## 2. Armazenamento Seguro de Tokens — `expo-secure-store`

### 2.1 Por que não AsyncStorage

| Critério | `AsyncStorage` | `expo-secure-store` |
|---|---|---|
| Criptografia | ❌ Texto plano | ✅ AES-256 (Android) / Keychain (iOS) |
| Acessível por outros apps? | ⚠️ Sim (sem isolamento) | ✅ Isolado por app |
| Para dados sensíveis? | ❌ **Nunca** | ✅ **Sempre** |
| Limite de tamanho | Ilimitado | 2KB por valor |

> **Regra de ouro:** `expo-secure-store` é para tokens, credenciais e dados sensíveis pequenos. Para dados maiores ou bancos criptografados, use SQLite com extensão de criptografia e armazene a chave de criptografia no SecureStore.

### 2.2 Instalação

```bash
npx expo install expo-secure-store
```

> **Nota:** Não funciona na web — apenas iOS e Android. Adicione fallback para `Platform.OS === 'web'` se necessário.

### 2.3 Serviço centralizado — `src/services/storage/secure-storage.ts`

Crie um wrapper tipado e centralizado sobre o `expo-secure-store`. **Nunca use o `expo-secure-store` diretamente nos componentes ou hooks** — sempre use este serviço.

```typescript
// src/services/storage/secure-storage.ts
import * as SecureStore from 'expo-secure-store';
import type { SecureStoreOptions } from 'expo-secure-store';
import { logger } from '@services/monitoring/logger';

/**
 * Opções padrão de segurança para o SecureStore.
 *
 * WHEN_UNLOCKED: O item é acessível somente quando o dispositivo
 * está desbloqueado. Recomendado para a maioria dos casos.
 * Use WHEN_UNLOCKED_THIS_DEVICE_ONLY para tokens críticos
 * que não devem ser restaurados via backup iCloud/Google.
 */
const DEFAULT_OPTIONS: SecureStoreOptions = {
  keychainAccessible: SecureStore.WHEN_UNLOCKED,
};

/**
 * Opções para dados de alta sensibilidade (ex: refresh token).
 * THIS_DEVICE_ONLY impede que o dado seja restaurado em outro
 * dispositivo, mesmo a partir de um backup.
 */
const SENSITIVE_OPTIONS: SecureStoreOptions = {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
};

/**
 * Armazena um valor no SecureStore.
 * @param key Chave de armazenamento
 * @param value Valor a armazener (string). Serialize objetos com JSON.stringify.
 * @param sensitive Se true, usa opções que impedem backup cross-device.
 */
export async function secureSet(
  key: string,
  value: string,
  sensitive = false,
): Promise<void> {
  try {
    const options = sensitive ? SENSITIVE_OPTIONS : DEFAULT_OPTIONS;
    await SecureStore.setItemAsync(key, value, options);
  } catch (error) {
    // ⚠️ Não logue o `value` — pode ser um token
    logger.error('Falha ao armazenar dado seguro', { key });
    throw error;
  }
}

/**
 * Recupera um valor do SecureStore.
 * @returns O valor armazenado ou null se não existir.
 */
export async function secureGet(key: string): Promise<string | null> {
  try {
    return await SecureStore.getItemAsync(key, DEFAULT_OPTIONS);
  } catch (error) {
    logger.warn('Falha ao recuperar dado seguro', { key });
    return null;
  }
}

/**
 * Remove um valor do SecureStore.
 */
export async function secureDelete(key: string): Promise<void> {
  try {
    await SecureStore.deleteItemAsync(key);
  } catch (error) {
    logger.warn('Falha ao remover dado seguro', { key });
  }
}

/**
 * Remove todos os dados de autenticação do SecureStore.
 * Chame no logout para garantir limpeza completa.
 */
export async function clearAuthStorage(): Promise<void> {
  await Promise.all([
    secureDelete(STORAGE_KEYS.ACCESS_TOKEN),
    secureDelete(STORAGE_KEYS.REFRESH_TOKEN),
    secureDelete(STORAGE_KEYS.USER_ID),
  ]);
}
```

### 2.4 Chaves de armazenamento centralizadas — `src/constants/storage-keys.ts`

```typescript
// src/constants/storage-keys.ts
/**
 * Chaves únicas para o SecureStore.
 * Centralizar as chaves evita typos e colisões.
 */
export const STORAGE_KEYS = {
  ACCESS_TOKEN: 'auth.access_token',
  REFRESH_TOKEN: 'auth.refresh_token',
  USER_ID: 'auth.user_id',
} as const;

export type StorageKey = (typeof STORAGE_KEYS)[keyof typeof STORAGE_KEYS];
```

### 2.5 Integração com a Store de Auth

O token de acesso deve ser carregado do `SecureStore` ao inicializar o app e limpo no logout. A store Zustand mantém o token **em memória** para desempenho; o SecureStore é a fonte de verdade persistida.

```typescript
// src/features/auth/store/auth.store.ts
import { create } from 'zustand';
import { secureGet, secureSet, clearAuthStorage } from '@services/storage/secure-storage';
import { STORAGE_KEYS } from '@constants/storage-keys';
import { logger } from '@services/monitoring/logger';

type AuthState = {
  accessToken: string | null;
  userId: string | null;
  isHydrated: boolean;
};

type AuthActions = {
  setAuth: (accessToken: string, userId: string) => Promise<void>;
  clearAuth: () => Promise<void>;
  hydrateFromStorage: () => Promise<void>;
};

export const useAuthStore = create<AuthState & AuthActions>((set) => ({
  accessToken: null,
  userId: null,
  isHydrated: false,

  setAuth: async (accessToken, userId) => {
    // Persiste no SecureStore
    await Promise.all([
      secureSet(STORAGE_KEYS.ACCESS_TOKEN, accessToken),
      secureSet(STORAGE_KEYS.USER_ID, userId),
    ]);
    // Mantém em memória (sem logar o token)
    set({ accessToken, userId });
    logger.info('Auth definida com sucesso', { userId });
  },

  clearAuth: async () => {
    // Limpa persistência e memória
    await clearAuthStorage();
    set({ accessToken: null, userId: null });
    logger.info('Auth limpa no logout');
  },

  hydrateFromStorage: async () => {
    const [accessToken, userId] = await Promise.all([
      secureGet(STORAGE_KEYS.ACCESS_TOKEN),
      secureGet(STORAGE_KEYS.USER_ID),
    ]);
    set({ accessToken, userId, isHydrated: true });
  },
}));

// Hooks públicos exportados pelo barrel
export const useAccessToken = () => useAuthStore((s) => s.accessToken);
export const useIsAuthenticated = () => useAuthStore((s) => s.accessToken !== null);
export const useAuthActions = () =>
  useAuthStore((s) => ({ setAuth: s.setAuth, clearAuth: s.clearAuth }));
```

### 2.6 Hidratação no layout raiz

```tsx
// src/app/_layout.tsx
import { useEffect } from 'react';
import { useAuthStore } from '@features/auth';

export default function RootLayout() {
  const hydrateFromStorage = useAuthStore((s) => s.hydrateFromStorage);
  const isHydrated = useAuthStore((s) => s.isHydrated);

  useEffect(() => {
    hydrateFromStorage();
  }, [hydrateFromStorage]);

  if (!isHydrated) {
    return null; // Ou um SplashScreen enquanto hidrata
  }

  return <Stack />;
}
```

### 2.7 Opções de `keychainAccessible` — Quando usar cada uma

| Opção | Quando usar |
|---|---|
| `WHEN_UNLOCKED` | Tokens de acesso de curta duração. **Padrão recomendado.** |
| `WHEN_UNLOCKED_THIS_DEVICE_ONLY` | Refresh tokens ou dados que não devem ser restaurados em outro device. |
| `AFTER_FIRST_UNLOCK` | Dados que precisam estar acessíveis em background (ex: notificações push que precisam renovar token). |
| `WHEN_PASSCODE_SET_THIS_DEVICE_ONLY` | Dados de alta segurança que devem ser perdidos se o passcode for removido. |

---

## 3. Certificate Pinning — Proteção Contra MITM

### 3.1 Visão Geral

O certificate pinning impede que um atacante intercepte e modifique o tráfego entre o app e a API, mesmo que ele consiga instalar um certificado CA falso no dispositivo.

> **Preço:** Certificate pinning requer um **Custom Development Build** (não funciona no Expo Go). Não é possível testar via Expo Go.

### 3.2 Biblioteca recomendada

Use `react-native-ssl-public-key-pinning` — compatível com Expo Managed Workflow via Config Plugin (não requer ejetar):

```bash
npx expo install react-native-ssl-public-key-pinning
```

> **Por que Public Key Pinning?** Pinar a chave pública (e não o certificado inteiro) permite que o certificado seja renovado sem exigir atualização do app, desde que o par de chaves permaneça o mesmo.

### 3.3 Configurar no `app.config.ts`

O plugin configura automaticamente o pinning para iOS (via NSURLSession) e Android (via OkHttp):

```typescript
// app.config.ts
import type { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  plugins: [
    // ... outros plugins
    [
      'react-native-ssl-public-key-pinning',
      {
        // Desabilitar o Network Inspector do expo-dev-client no iOS
        // para não interferir com o pinning durante o desenvolvimento
        disableDefaultChecks: true,
      },
    ],
  ],
});
```

> **Nota iOS:** Se estiver usando `expo-dev-client`, desabilite o Network Inspector em `app.config.ts`:
> ```typescript
> ['expo-build-properties', { ios: { networkInspector: false } }]
> ```

### 3.4 Inicialização — `src/services/api/ssl-pinning.ts`

Inicialize o pinning o mais cedo possível, antes de qualquer chamada de rede.

```typescript
// src/services/api/ssl-pinning.ts
import { initializeSslPinning } from 'react-native-ssl-public-key-pinning';
import { env } from '@config/env';
import { logger } from '@services/monitoring/logger';

/**
 * Hashes das chaves públicas da API (SHA-256 em base64).
 *
 * Como obter o hash:
 *   openssl s_client -connect api.meuapp.com.br:443 -servername api.meuapp.com.br \
 *     | openssl x509 -pubkey -noout \
 *     | openssl pkey -pubin -outform DER \
 *     | openssl dgst -sha256 -binary \
 *     | base64
 *
 * Alternativa online: https://www.ssllabs.com/ssltest/ → Certificate → Pin SHA256
 *
 * ⚠️ SEMPRE inclua pelo menos 2 hashes (primário + backup) para evitar
 *    lockout se o certificado primário precisar ser rotacionado.
 */
const SSL_PINS: Record<string, { includeSubdomains?: boolean; publicKeyHashes: string[] }> = {
  'api.meuapp.com.br': {
    includeSubdomains: false,
    publicKeyHashes: [
      'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=', // Substitua pelo hash real
      'BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=', // Hash de backup
    ],
  },
};

/**
 * Inicializa o SSL Public Key Pinning.
 * Deve ser chamado antes de qualquer chamada de rede (ao inicializar o app).
 *
 * Em development, o pinning é desabilitado para permitir uso de proxies
 * de depuração (Charles, Proxyman). Habilite ONLY se quiser testar pinning local.
 */
export async function initializePinning(): Promise<void> {
  // Desabilitar pinning em development para não bloquear proxies de debug
  if (env.isDev) {
    logger.debug('SSL Pinning desabilitado em development');
    return;
  }

  try {
    await initializeSslPinning(SSL_PINS);
    logger.info('SSL Pinning inicializado com sucesso');
  } catch (error) {
    logger.error('Falha ao inicializar SSL Pinning', {
      error: error instanceof Error ? error.message : 'unknown',
    });
    // Em staging/production, falha no pinning é crítica — re-throw
    throw error;
  }
}
```

### 3.5 Chamada no layout raiz

```tsx
// src/app/_layout.tsx
import { useEffect } from 'react';
import { initializePinning } from '@services/api/ssl-pinning';

// Inicializar antes do render do componente
initializePinning().catch(() => {
  // Erro já logado dentro de initializePinning
});
```

### 3.6 Rotação de certificados

> **CRÍTICO:** Antes de rotacionar o certificado da API, **sempre**:
> 1. Inclua o hash da nova chave pública como segundo pin em um **novo release do app**.
> 2. Aguarde a adoção do novo app pela base de usuários (mínimo 2 semanas ou conforme analytics).
> 3. Somente então rotacione o certificado no servidor.
> 4. Em um release seguinte, remova o hash antigo.

Use EAS Update (OTA) para distribuir rapidamente novos hashes se um certificado for comprometido.

---

## 4. Prevenção de Leaks em Logs

### 4.1 Regra fundamental

> **Nunca logue dados sensíveis.** Logs aparecem em: ferramentas de crash (Sentry), sistemas de APM, `adb logcat`, Xcode Console, e podem ser extraídos por atacantes com acesso físico ao dispositivo.

### 4.2 O que NUNCA deve aparecer em logs

```typescript
// ❌ ERRADO — vaza o token no log
logger.info('Usuário autenticado', { token: accessToken });

// ❌ ERRADO — vaza dados pessoais
logger.info('Login bem-sucedido', { email: user.email, cpf: user.cpf });

// ❌ ERRADO — vaza credenciais da API
logger.debug('Chamando API', { url, headers: { Authorization: `Bearer ${token}` } });

// ❌ ERRADO — vaza stack trace com dados da requisição
logger.error('Erro na API', { error, requestBody: JSON.stringify(payload) });
```

```typescript
// ✅ CORRETO — referência por ID, sem dado sensível
logger.info('Usuário autenticado', { userId: user.id });

// ✅ CORRETO — tipo do erro, sem payload completo
logger.error('Falha na autenticação', {
  errorType: error.type,
  statusCode: error.statusCode,
});

// ✅ CORRETO — metadados úteis sem segredos
logger.debug('Chamando API', { url, method: 'POST', feature: 'auth' });
```

### 4.3 Dados que NUNCA devem aparecer em logs

| Categoria | Exemplos |
|---|---|
| **Tokens** | Access token, refresh token, API keys |
| **Dados pessoais (LGPD/GDPR)** | E-mail, CPF, RG, nome completo, data de nascimento |
| **Credenciais** | Senha, PIN, dados de cartão |
| **Dados financeiros** | Número de conta, agência, dados de cartão |
| **Chaves de criptografia** | Chaves privadas, segredos de criptografia |

### 4.4 Regra ESLint para `no-console`

A skill `setup-react-native` já configura `'no-console': ['warn', { allow: ['warn', 'error'] }]`. Atualize para `error` em projetos novos e adicione uma regra customizada para `console.log`:

```js
// eslint.config.js — regra reforçada
rules: {
  // Proíbe console.* — use logger.* de @services/monitoring/logger
  'no-console': ['error', { allow: [] }], // Não permite NENHUM console.*
}
```

> **Justificativa:** `console.warn` e `console.error` também expõem dados em produção. O logger centralizado controla automaticamente o que vai para o Sentry vs. console usando `env.isDev`.

### 4.5 Redação de Axios Interceptor — headers e body

Configure o interceptor do Axios para remover dados sensíveis antes de qualquer log da requisição:

```typescript
// src/services/api/client.ts
import axios from 'axios';
import { env } from '@config/env';
import { logger } from '@services/monitoring/logger';

export const api = axios.create({
  baseURL: env.API_URL,
  timeout: 15_000,
});

// Interceptor de request — remove Authorization do log
api.interceptors.request.use(
  (config) => {
    // ✅ Log apenas metadados — sem a URL completa com query params sensíveis
    // e sem os headers de autenticação
    if (env.isDev) {
      logger.debug('API Request', {
        method: config.method?.toUpperCase() ?? 'UNKNOWN',
        url: config.url ?? 'unknown',
        // ❌ NÃO logue: config.headers, config.data
      });
    }
    return config;
  },
  (error) => {
    logger.error('Erro ao preparar requisição', { url: error.config?.url });
    return Promise.reject(error);
  },
);

// Interceptor de response — sem dados sensíveis
api.interceptors.response.use(
  (response) => response,
  (error) => {
    const status = error.response?.status;
    const url = error.config?.url;

    // ✅ Loga apenas status e URL — sem body da resposta (pode conter dados do usuário)
    logger.warn('API Error', { status, url });

    return Promise.reject(error);
  },
);
```

### 4.6 Sentry `beforeSend` — filtro de segurança

Já configurado na skill `monitoring-observability`. Reforce removendo campos sensíveis adicionais:

```typescript
// src/services/monitoring/sentry.ts — beforeSend
beforeSend: (event) => {
  // Remover e-mail (LGPD/GDPR)
  if (event.user?.email) {
    delete event.user.email;
  }

  // Remover headers de Authorization de breadcrumbs de rede
  event.breadcrumbs?.values?.forEach((breadcrumb) => {
    if (breadcrumb.data?.['Authorization']) {
      breadcrumb.data['Authorization'] = '[Redacted]';
    }
  });

  // Remover request bodies que podem conter credenciais
  if (event.request?.data) {
    const data = event.request.data as Record<string, unknown>;
    const sensitiveFields = ['password', 'token', 'cpf', 'card_number', 'cvv'];
    sensitiveFields.forEach((field) => {
      if (field in data) data[field] = '[Redacted]';
    });
  }

  return event;
},
```

---

## 5. Outras Práticas de Segurança

### 5.1 Android — Desabilitar Auto Backup

Por padrão, o Android faz backup dos dados do app para o Google Drive, incluindo dados do `AsyncStorage`. Dados do `expo-secure-store` com `_THIS_DEVICE_ONLY` são excluídos automaticamente do backup. Configure o `app.config.ts` para excluir diretórios sensíveis do backup:

```typescript
// app.config.ts
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  android: {
    ...config.android,
    // Configuração de backup exclusivo no arquivo res/xml/backup_rules.xml
    // Gerado pelo Expo Config Plugin abaixo
  },
  plugins: [
    [
      'expo-build-properties',
      {
        android: {
          // Desabilita backup completo se o app lida com dados financeiros/médicos
          // Para a maioria dos apps: apenas configure as excludePatterns abaixo
        },
      },
    ],
  ],
});
```

> **Nota:** O `expo-secure-store` com `WHEN_UNLOCKED_THIS_DEVICE_ONLY` já exclui automaticamente o item do backup do iOS/Android. Para o `AsyncStorage`, configure manualmente as regras de backup no `android/app/src/main/res/xml/backup_rules.xml` (somente em Bare Workflow).

### 5.2 Sanitização de inputs de usuário

Nunca confie em dados vindos de formulários sem validação. Use Zod (skill `forms-pattern`) para validar antes de enviar à API:

```typescript
// ✅ validação com Zod antes de usar os dados
const schema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(128),
});

const result = schema.safeParse(formData);
if (!result.success) {
  // Exibe erros de validação, não processa dados inválidos
  return;
}

// result.data é tipado e sanitizado
await login(result.data);
```

### 5.3 OAuth 2.0 com PKCE

Para fluxos de autenticação com providers externos (Google, Apple, etc.), use sempre o **PKCE (Proof Key for Code Exchange)**:

```bash
npx expo install expo-auth-session expo-crypto
```

```typescript
import * as AuthSession from 'expo-auth-session';
import * as Crypto from 'expo-crypto';

// PKCE é habilitado automaticamente pelo expo-auth-session
const [request, response, promptAsync] = useAuthRequest(
  {
    clientId: 'your-client-id',
    redirectUri: AuthSession.makeRedirectUri(),
    scopes: ['openid', 'profile', 'email'],
    usePKCE: true, // ← sempre habilitar
  },
  discovery,
);
```

### 5.4 Auditoria de dependências no CI

Adicione ao workflow `ci.yml` (skill `ci-cd-pipeline`):

```yaml
# .github/workflows/ci.yml — adicionar step
- name: Auditoria de segurança das dependências
  run: npm audit --audit-level=high
  continue-on-error: false # Falha se houver vulnerabilidades HIGH ou CRITICAL
```

### 5.5 ProGuard / R8 — Minificação no Android

O Expo habilita o Hermes por padrão no SDK 54, que já minifica o JavaScript. Para minificação adicional do bytecode Android nativo:

```typescript
// app.config.ts
android: {
  enableProguardInReleaseBuilds: true,
  enableShrinkResourcesInReleaseBuilds: true,
}
```

---

## 6. Estrutura de Arquivos

```text
src/
├── constants/
│   └── storage-keys.ts         ← Chaves do SecureStore centralizadas
├── services/
│   ├── api/
│   │   ├── client.ts           ← Axios com interceptors sem vazamento de dados
│   │   └── ssl-pinning.ts      ← initializePinning()
│   ├── storage/
│   │   └── secure-storage.ts   ← secureGet / secureSet / secureDelete / clearAuthStorage
│   └── monitoring/
│       ├── sentry.ts           ← initSentry() + beforeSend com filtros
│       └── logger.ts           ← Logger que não expõe dados sensíveis
├── features/
│   └── auth/
│       └── store/
│           └── auth.store.ts   ← Usa secure-storage para persistência de token
└── app/
    └── _layout.tsx             ← initializePinning() + hydrateFromStorage()
```

---

## 7. Fluxo de Setup Completo (Passo a Passo)

**1. Instalar dependências**
```bash
npx expo install expo-secure-store
npx expo install react-native-ssl-public-key-pinning
```

**2. Criar chaves de storage**
Criar `src/constants/storage-keys.ts` com as chaves centralizadas.

**3. Criar o Secure Storage Service**
Criar `src/services/storage/secure-storage.ts` com funções `secureSet`, `secureGet`, `secureDelete` e `clearAuthStorage`.

**4. Integrar com a Store de Auth**
Atualizar `src/features/auth/store/auth.store.ts` para usar `secureSet` no `setAuth` e `clearAuthStorage` no `clearAuth`.

**5. Adicionar hidratação no layout raiz**
Chamar `hydrateFromStorage()` no `useEffect` do `_layout.tsx`.

**6. Configurar SSL Pinning**
Criar `src/services/api/ssl-pinning.ts` com os hashes reais da API.
Adicionar o plugin ao `app.config.ts`.
Chamar `initializePinning()` no layout raiz.

**7. Atualizar o `beforeSend` do Sentry**
Adicionar filtros de campos sensíveis ao `beforeSend` conforme a seção 4.6.

**8. Gerar Custom Dev Build**
O SSL Pinning não funciona no Expo Go — gere um development build:
```bash
eas build --profile development --platform all
```

---

## 8. Checklist de Segurança Antes do Release

### Armazenamento

- [ ] Tokens de acesso e refresh armazenados com `expo-secure-store` (não `AsyncStorage`)?
- [ ] Tokens **não** armazenados em variáveis globais ou estado do React (apenas em `AuthStore` em memória)?
- [ ] `clearAuthStorage()` chamado no logout?
- [ ] Refresh token usa `WHEN_UNLOCKED_THIS_DEVICE_ONLY`?
- [ ] Chaves do SecureStore centralizadas em `src/constants/storage-keys.ts`?

### Comunicação de Rede

- [ ] Todas as chamadas de API usam HTTPS?
- [ ] SSL Pinning configurado para a(s) API(s) crítica(s)?
- [ ] Pelo menos 2 hashes de chave pública configurados por domínio (principal + backup)?
- [ ] Estratégia de rotação de certificados documentada?
- [ ] Proxies de debug (Charles, Proxyman) testaram que o pinning funciona?

### Logs e Observabilidade

- [ ] Nenhum log contém: token, senha, CPF, e-mail, dados de cartão?
- [ ] `no-console: error` habilitado no ESLint?
- [ ] Axios interceptors **não** logam `headers.Authorization` nem `request.data` com credenciais?
- [ ] Sentry `beforeSend` remove e-mail, Authorization e campos sensíveis?
- [ ] `Sentry.mobileReplayIntegration` com `maskAllText: true` e `maskAllImages: true`?

### Variáveis de Ambiente e Secrets

- [ ] Nenhuma API key privada em variável `EXPO_PUBLIC_*` (bundle visível pelo usuário)?
- [ ] Secrets sensíveis gerenciados via **EAS Secrets** (`eas secret:create`)?
- [ ] Chamadas que requerem API keys privadas passam por backend proxy?
- [ ] Hashes de SSL Pinning **não** em variáveis de ambiente (são dados de configuração, podem ficar no código)?

### Autenticação

- [ ] Refresh token implementado com expiração e renovação automática?
- [ ] OAuth 2.0 usa PKCE (`usePKCE: true`)?
- [ ] Token limpo do SecureStore e da memória no logout?
- [ ] Rotas protegidas usam `Stack.Protected` com guard de autenticação (ver skill `navigation-patterns`)?

### Build e Distribuição

- [ ] `enableProguardInReleaseBuilds: true` no Android?
- [ ] Hermes habilitado (padrão no SDK 54)?
- [ ] `npm audit --audit-level=high` passando sem vulnerabilidades HIGH/CRITICAL?
- [ ] Build de produção testado em device real (não simulador) para validar SSL Pinning?
- [ ] App não funciona como esperado com proxy MITM ativo (validado com Charles/Proxyman)?

### LGPD / GDPR

- [ ] E-mail e dados pessoais removidos dos eventos do Sentry (`beforeSend`)?
- [ ] Política de retenção de dados definida (quando limpar dados locais)?
- [ ] Usuário pode solicitar deleção de dados (e o app limpa o SecureStore)?
- [ ] Consentimento de coleta de dados implementado (se aplicável)?

### Dependências

- [ ] `npm audit` rodando automaticamente no CI (skill `ci-cd-pipeline`)?
- [ ] Expo SDK na versão mais recente com patches de segurança?
- [ ] `expo-secure-store`, `@sentry/react-native` e demais libs críticas atualizadas?

---

## 9. Anti-Padrões — O que NUNCA Fazer

```typescript
// ❌ NUNCA armazenar token no AsyncStorage
await AsyncStorage.setItem('token', accessToken);

// ❌ NUNCA logar dados sensíveis
logger.info('Token obtido', { token: accessToken });
logger.info('Usuário logado', { email: user.email });

// ❌ NUNCA colocar API key privada como EXPO_PUBLIC_
// EXPO_PUBLIC_STRIPE_SECRET_KEY=sk_live_... (visível no bundle!)

// ❌ NUNCA hardcodar hashes do pinning como literal mágico sem comentário
publicKeyHashes: ['abc123'] // O que é esse hash? De qual domínio? Expira quando?

// ❌ NUNCA ignorar erro de SSL Pinning silenciosamente
try {
  await initializeSslPinning(pins);
} catch {
  // silêncio — o usuário ficará exposto a ataques MITM
}

// ❌ NUNCA usar console.log em produção
console.log('Resposta da API:', response.data); // Dados do usuário em logcat/Console
```

```typescript
// ✅ CORRETO — SecureStore para token
await secureSet(STORAGE_KEYS.ACCESS_TOKEN, accessToken);

// ✅ CORRETO — log sem dado sensível
logger.info('Login bem-sucedido', { userId: user.id });

// ✅ CORRETO — API key privada via backend proxy ou EAS Secret
// (não aparece no bundle)

// ✅ CORRETO — hashes documentados
publicKeyHashes: [
  'hash_primario_sha256_base64=', // api.meuapp.com.br — Expira: 2026-01
  'hash_backup_sha256_base64=',   // Backup pin — rotacionar ANTES de renovar o certificado
],
```
