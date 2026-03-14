---
name: environment-config
description: Gerenciamento de variáveis de ambiente com Expo SDK 54+, separação entre .env.development, .env.staging e .env.production, expo-constants, e exposição segura via EXPO_PUBLIC_.
---

# Configuração de Variáveis de Ambiente — Expo SDK 54+

Você é um Engenheiro de Software Senior especialista em configuração de projetos React Native com Expo. Sua responsabilidade é garantir que o gerenciamento de variáveis de ambiente seja **seguro**, **tipado** e **separado por ambiente**, seguindo as melhores práticas da documentação oficial do Expo e compatível com o **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

---

## Quando usar esta skill

- **Novo projeto:** Configurar variáveis de ambiente do zero em um projeto Expo.
- **Separação de ambientes:** Criar arquivos `.env` separados para development, staging e production.
- **Segurança:** Garantir que apenas variáveis seguras (com prefixo `EXPO_PUBLIC_`) sejam expostas no bundle.
- **Acesso a config em runtime:** Usar `expo-constants` para acessar valores do `app.config.ts` em tempo de execução.
- **EAS Build/Update:** Configurar variáveis de ambiente para builds de produção via EAS.

---

## 1. Como o Expo Carrega Variáveis de Ambiente

### Mecanismo nativo do Expo CLI

O Expo CLI **automaticamente** carrega variáveis de ambiente de arquivos `.env` no diretório raiz do projeto. Variáveis com o prefixo `EXPO_PUBLIC_` são **inlined** (substituídas diretamente) no bundle JavaScript em tempo de build pelo Metro Bundler.

> **IMPORTANTE:** Variáveis **sem** o prefixo `EXPO_PUBLIC_` são **removidas** do bundle e **não estarão disponíveis** no código JavaScript da aplicação. Elas só ficam acessíveis em arquivos de configuração server-side como `app.config.ts`.

### Ordem de precedência (resolução padrão do dotenv)

O Expo segue a [resolução padrão do dotenv](https://github.com/bkeepers/dotenv/blob/c6e583a/README.md#what-other-env-files-can-i-use):

| Prioridade | Arquivo | Commitado no Git? | Descrição |
|---|---|---|---|
| 1 (mais alta) | `.env.local` | ❌ Não | Overrides locais da máquina do dev |
| 2 | `.env` | ✅ Sim | Valores padrão compartilhados pelo time |

> **Nota SDK 54:** O Expo **não recomenda** usar `NODE_ENV` para alternar entre arquivos `.env` (ex: `.env.test`, `.env.production`). Comandos como `npx expo export` forçam `NODE_ENV=production`, ignorando qualquer valor customizado. Use a estratégia de **swap via script** ou **EAS environments** descrita nas seções abaixo.

---

## 2. Separação por Ambiente (Development / Staging / Production)

### Estratégia recomendada: arquivos nomeados + script de swap

Como o Expo não suporta nativamente `.env.development`, `.env.staging` e `.env.production` como ambientes automáticos, a abordagem recomendada é:

1. Manter arquivos `.env` nomeados por ambiente.
2. Usar um script que copia o arquivo correto para `.env.local` antes de iniciar o app.
3. O `.env.local` é ignorado no git e tem **prioridade máxima** na resolução.

### Estrutura de arquivos

```text
my-app/
├── .env                    # Valores padrão (commitado) — fallback
├── .env.development        # Variáveis de development (commitado)
├── .env.staging            # Variáveis de staging (commitado)
├── .env.production         # Variáveis de production (commitado)
├── .env.local              # Gerado pelo script — NÃO commitado
├── scripts/
│   └── set-env.js          # Script para alternar ambientes
└── ...
```

### Conteúdo dos arquivos `.env`

#### `.env` (fallback padrão)

```env
# Valores padrão — se nenhum .env.local existir, estes valores serão usados
EXPO_PUBLIC_APP_ENV=development
EXPO_PUBLIC_API_URL=http://localhost:3000/api
EXPO_PUBLIC_ENABLE_LOGS=true
```

#### `.env.development`

```env
EXPO_PUBLIC_APP_ENV=development
EXPO_PUBLIC_API_URL=http://localhost:3000/api
EXPO_PUBLIC_ENABLE_LOGS=true

# Variáveis NÃO-públicas (disponíveis apenas no app.config.ts)
API_SECRET_KEY=dev-secret-key-12345
SENTRY_AUTH_TOKEN=dev-sentry-token
```

#### `.env.staging`

```env
EXPO_PUBLIC_APP_ENV=staging
EXPO_PUBLIC_API_URL=https://staging-api.meuapp.com.br/api
EXPO_PUBLIC_ENABLE_LOGS=true

API_SECRET_KEY=staging-secret-key-67890
SENTRY_AUTH_TOKEN=staging-sentry-token
```

#### `.env.production`

```env
EXPO_PUBLIC_APP_ENV=production
EXPO_PUBLIC_API_URL=https://api.meuapp.com.br/api
EXPO_PUBLIC_ENABLE_LOGS=false

API_SECRET_KEY=prod-secret-key-secure
SENTRY_AUTH_TOKEN=prod-sentry-token
```

### Script de swap — `scripts/set-env.js`

```js
const fs = require('fs');
const path = require('path');

const env = process.argv[2] || 'development';
const validEnvs = ['development', 'staging', 'production'];

if (!validEnvs.includes(env)) {
  console.error(`❌ Ambiente inválido: "${env}". Use: ${validEnvs.join(', ')}`);
  process.exit(1);
}

const source = path.resolve(__dirname, '..', `.env.${env}`);
const target = path.resolve(__dirname, '..', '.env.local');

if (!fs.existsSync(source)) {
  console.error(`❌ Arquivo não encontrado: ${source}`);
  process.exit(1);
}

fs.copyFileSync(source, target);
console.log(`✅ Ambiente configurado: ${env}`);
console.log(`   .env.${env} → .env.local`);
```

### Scripts no `package.json`

```json
{
  "scripts": {
    "env:dev": "node scripts/set-env.js development",
    "env:staging": "node scripts/set-env.js staging",
    "env:prod": "node scripts/set-env.js production",
    "start:dev": "npm run env:dev && expo start",
    "start:staging": "npm run env:staging && expo start",
    "start:prod": "npm run env:prod && expo start"
  }
}
```

### `.gitignore`

```gitignore
# Variáveis de ambiente locais (gerado pelo script set-env.js)
.env.local
.env*.local
```

> **Nota:** Os arquivos `.env.development`, `.env.staging` e `.env.production` **podem ser commitados** se contêm apenas variáveis `EXPO_PUBLIC_*`. Variáveis sensíveis (sem o prefixo) devem ser gerenciadas via **EAS Secrets** em produção.

---

## 3. Lendo Variáveis de Ambiente no Código

### Regras obrigatórias de leitura

```typescript
// ✅ CORRETO — referência estática com dot notation
const apiUrl = process.env.EXPO_PUBLIC_API_URL;

// ❌ INCORRETO — bracket notation NÃO funciona
const apiUrl = process.env['EXPO_PUBLIC_API_URL'];

// ❌ INCORRETO — destructuring NÃO funciona
const { EXPO_PUBLIC_API_URL } = process.env;

// ❌ INCORRETO — acesso dinâmico NÃO funciona
const key = 'EXPO_PUBLIC_API_URL';
const value = process.env[key];
```

> **Por quê?** O Metro Bundler faz substituição **em tempo de build** via AST (Abstract Syntax Tree). Apenas expressões na forma `process.env.EXPO_PUBLIC_*` com dot notation são reconhecidas e inlined.

### Recarregamento

- Variáveis são atualizadas **sem precisar reiniciar** o Expo CLI ou limpar cache do Metro.
- É necessário fazer um **full reload** (shake gesture → Reload, ou Ctrl+M no Android) para ver valores atualizados.

---

## 4. Módulo Centralizado de Configuração — `src/config/env.ts`

Crie um módulo centralizado que **valida** e **exporta** todas as variáveis de ambiente de forma tipada.

### `src/config/env.ts`

```typescript
/**
 * Módulo centralizado de variáveis de ambiente.
 *
 * Todas as variáveis EXPO_PUBLIC_* são lidas aqui e exportadas
 * com tipos, valores padrão e validação.
 *
 * REGRA: Nunca use `process.env.EXPO_PUBLIC_*` diretamente nos
 * componentes/hooks/services. Sempre importe deste módulo.
 */

// ─── Ambiente ────────────────────────────────────────────────────────────────
export const APP_ENV = process.env.EXPO_PUBLIC_APP_ENV ?? 'development';

export type AppEnvironment = 'development' | 'staging' | 'production';

export const isDev = APP_ENV === 'development';
export const isStaging = APP_ENV === 'staging';
export const isProd = APP_ENV === 'production';

// ─── API ─────────────────────────────────────────────────────────────────────
export const API_URL = process.env.EXPO_PUBLIC_API_URL ?? 'http://localhost:3000/api';

// ─── Feature Flags ───────────────────────────────────────────────────────────
export const ENABLE_LOGS = process.env.EXPO_PUBLIC_ENABLE_LOGS === 'true';

// ─── Validação em runtime ────────────────────────────────────────────────────
function validateEnv(): void {
  const required: Array<{ name: string; value: string | undefined }> = [
    { name: 'EXPO_PUBLIC_APP_ENV', value: process.env.EXPO_PUBLIC_APP_ENV },
    { name: 'EXPO_PUBLIC_API_URL', value: process.env.EXPO_PUBLIC_API_URL },
  ];

  const missing = required.filter((v) => !v.value);

  if (missing.length > 0) {
    const names = missing.map((v) => v.name).join(', ');
    console.warn(`⚠️ Variáveis de ambiente ausentes: ${names}`);

    if (isProd) {
      throw new Error(`❌ Variáveis de ambiente obrigatórias ausentes em production: ${names}`);
    }
  }
}

// Executa validação ao importar o módulo
validateEnv();

// ─── Export consolidado ──────────────────────────────────────────────────────
export const env = {
  APP_ENV: APP_ENV as AppEnvironment,
  API_URL,
  ENABLE_LOGS,
  isDev,
  isStaging,
  isProd,
} as const;
```

### Uso nos componentes

```typescript
import { env } from '@config/env';

// Acessar valores
console.log(env.API_URL);       // 'https://api.meuapp.com.br/api'
console.log(env.isDev);         // false
console.log(env.APP_ENV);       // 'production'

// Condicional por ambiente
if (env.ENABLE_LOGS) {
  console.log('Debug info...');
}
```

---

## 5. Expo Constants — Acesso à Configuração em Runtime

### Quando usar `expo-constants`

O `expo-constants` fornece acesso ao **manifesto da aplicação** em runtime — ou seja, os valores definidos no `app.config.ts`. Use-o para:

- Acessar metadados do app (versão, nome, `slug`).
- Ler valores do campo `extra` definidos no `app.config.ts`.
- Identificar o ambiente de execução (Expo Go, Development Build, Standalone).

> **Diferença chave:**
> - `process.env.EXPO_PUBLIC_*` → Variáveis inlined no bundle JavaScript (substituídas em build time).
> - `Constants.expoConfig?.extra` → Valores do `app.config.ts` acessíveis via manifesto em runtime.

### Instalação

```bash
npx expo install expo-constants
```

### `app.config.ts` com variáveis de ambiente

```typescript
import type { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: 'Meu App',
  slug: 'meu-app',
  version: '1.0.0',
  extra: {
    // Valores disponíveis via Constants.expoConfig?.extra
    appEnv: process.env.EXPO_PUBLIC_APP_ENV ?? 'development',
    apiUrl: process.env.EXPO_PUBLIC_API_URL ?? 'http://localhost:3000/api',

    // Variáveis sensíveis (SEM prefixo EXPO_PUBLIC_)
    // Acessíveis apenas no app.config.ts e em scripts de build — NÃO no bundle JS
    sentryAuthToken: process.env.SENTRY_AUTH_TOKEN,

    // Metadados do EAS
    eas: {
      projectId: 'your-eas-project-id',
    },
  },
});
```

### Leitura via `expo-constants`

```typescript
import Constants from 'expo-constants';

// Acesso ao manifesto
const appVersion = Constants.expoConfig?.version;        // '1.0.0'
const appName = Constants.expoConfig?.name;              // 'Meu App'

// Acesso ao campo extra
const apiUrl = Constants.expoConfig?.extra?.apiUrl;      // 'https://api.meuapp.com.br/api'
const appEnv = Constants.expoConfig?.extra?.appEnv;      // 'production'

// Ambiente de execução
const executionEnv = Constants.executionEnvironment;
// 'storeClient' (Expo Go) | 'standalone' (build de produção) | 'bare'
```

### Tipagem do `extra` — `src/types/app-config.d.ts`

```typescript
declare module 'expo-constants' {
  interface AppConfig {
    extra?: {
      appEnv: 'development' | 'staging' | 'production';
      apiUrl: string;
      sentryAuthToken?: string;
      eas?: {
        projectId: string;
      };
    };
  }
}
```

---

## 6. Segurança — EXPO_PUBLIC_ e Variáveis Sensíveis

### Regra de ouro

> **NUNCA armazene secrets (API keys privadas, tokens de autenticação, chaves de criptografia) em variáveis com prefixo `EXPO_PUBLIC_`.** O bundle JavaScript é acessível ao usuário final — qualquer variável `EXPO_PUBLIC_*` pode ser extraída por engenharia reversa.

### O que acontece com cada tipo de variável

```text
Variável com EXPO_PUBLIC_ prefix:
├── ✅ Disponível em process.env no código JS
├── ✅ Inline no bundle (substituída em build time)
├── ✅ Acessível no app.config.ts
└── ⚠️  VISÍVEL para o usuário final (no bundle)

Variável SEM EXPO_PUBLIC_ prefix:
├── ❌ NÃO disponível em process.env no código JS
├── ❌ NÃO inline no bundle (removida pelo Metro)
├── ✅ Acessível no app.config.ts (via process.env normal)
└── ✅ Segura (não vai para o bundle)
```

### Tabela de classificação

| Variável | Prefixo `EXPO_PUBLIC_`? | Segura no bundle? | Exemplo |
|---|---|---|---|
| URL da API (pública) | ✅ Sim | N/A (dado público) | `EXPO_PUBLIC_API_URL` |
| Feature flags | ✅ Sim | N/A (dado público) | `EXPO_PUBLIC_ENABLE_LOGS` |
| Identificador do app | ✅ Sim | N/A (dado público) | `EXPO_PUBLIC_APP_ENV` |
| API Key pública (ex: Google Maps) | ✅ Sim | ⚠️ Aceitável se restrita por domínio/app | `EXPO_PUBLIC_MAPS_KEY` |
| API Secret / Token privado | ❌ Não | ✅ Segura | `API_SECRET_KEY` |
| Token do Sentry | ❌ Não | ✅ Segura | `SENTRY_AUTH_TOKEN` |
| Chaves de criptografia | ❌ Não | ✅ Segura | `ENCRYPTION_KEY` |
| Credenciais de banco | ❌ Não | ✅ Segura | `DATABASE_URL` |

### Onde colocar secrets em produção

1. **EAS Secrets** — Para builds na nuvem via EAS Build. Configurados via `eas secret:create` ou no painel do Expo.
2. **`app.config.ts` (extra)** — Variáveis sem `EXPO_PUBLIC_` podem ser lidas no `app.config.ts` para configurar SDKs nativos (ex: Sentry DSN), mas **não estarão no bundle JS**.
3. **Backend proxy** — Para API keys verdadeiramente sensíveis, a chamada deve ser feita pelo backend, nunca diretamente pelo app mobile.

---

## 7. Integração com EAS Build / Update

### Variáveis no `eas.json`

Para builds na nuvem, variáveis podem ser definidas por **profile** no `eas.json`:

```json
{
  "cli": {
    "version": ">= 15.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "EXPO_PUBLIC_APP_ENV": "development",
        "EXPO_PUBLIC_API_URL": "http://localhost:3000/api"
      }
    },
    "staging": {
      "distribution": "internal",
      "env": {
        "EXPO_PUBLIC_APP_ENV": "staging",
        "EXPO_PUBLIC_API_URL": "https://staging-api.meuapp.com.br/api"
      }
    },
    "production": {
      "env": {
        "EXPO_PUBLIC_APP_ENV": "production",
        "EXPO_PUBLIC_API_URL": "https://api.meuapp.com.br/api"
      }
    }
  }
}
```

### EAS Secrets (para variáveis sensíveis)

```bash
# Criar um secret (disponível em todas as builds)
eas secret:create --name API_SECRET_KEY --value "super-secret-value" --scope project

# Listar secrets existentes
eas secret:list

# Deletar um secret
eas secret:delete --name API_SECRET_KEY
```

> Secrets do EAS ficam disponíveis como `process.env.API_SECRET_KEY` durante o **build** (no `app.config.ts`), mas **nunca são inlined no bundle JS**.

### `eas env:pull` — Sincronizar variáveis do EAS localmente

```bash
# Puxa as variáveis do ambiente "development" do EAS para .env.local
eas env:pull --environment development

# Puxa as variáveis do ambiente "production"
eas env:pull --environment production
```

> Este comando substitui o conteúdo de `.env.local` com as variáveis configuradas no painel EAS para o ambiente escolhido.

---

## 8. Fluxo de Setup Completo (Passo a Passo)

Ao receber a solicitação de configurar variáveis de ambiente:

**1. Instalar `expo-constants`**
```bash
npx expo install expo-constants
```

**2. Criar arquivos `.env`**
Criar `.env`, `.env.development`, `.env.staging` e `.env.production` na raiz do projeto conforme a seção 2.

**3. Configurar `.gitignore`**
Adicionar `.env.local` e `.env*.local` ao `.gitignore`.

**4. Criar script de swap**
Criar `scripts/set-env.js` conforme a seção 2.

**5. Converter `app.json` para `app.config.ts`**
Se o projeto usa `app.json`, converter para `app.config.ts` conforme a seção 5 para suportar variáveis dinâmicas.

**6. Criar módulo de configuração**
Criar `src/config/env.ts` conforme a seção 4 com validação e tipagem.

**7. Criar declaração de tipos**
Criar `src/types/app-config.d.ts` conforme a seção 5 para tipar `Constants.expoConfig?.extra`.

**8. Adicionar scripts ao `package.json`**
Adicionar scripts `env:dev`, `env:staging`, `env:prod`, `start:dev`, `start:staging`, `start:prod` conforme a seção 2.

**9. Configurar EAS (se aplicável)**
Configurar profiles e variáveis no `eas.json` conforme a seção 7. Criar secrets via `eas secret:create` para variáveis sensíveis.

---

## Checklist de Verificação

Antes de considerar a configuração concluída, verifique cada item:

### Arquivos de Ambiente
- [ ] `.env` criado com valores padrão (commitado)?
- [ ] `.env.development`, `.env.staging` e `.env.production` criados?
- [ ] `.env.local` adicionado ao `.gitignore`?
- [ ] Todas as variáveis públicas têm prefixo `EXPO_PUBLIC_`?
- [ ] Nenhum secret/token está em variável `EXPO_PUBLIC_*`?

### Módulo de Configuração
- [ ] `src/config/env.ts` criado com validação e tipagem?
- [ ] Nenhum componente/hook usa `process.env.EXPO_PUBLIC_*` diretamente (todos importam de `@config/env`)?
- [ ] Leitura usa dot notation (`process.env.EXPO_PUBLIC_*`) — sem bracket ou destructuring?
- [ ] Validação roda ao importar o módulo? Erro em production, warn em dev?

### Expo Constants
- [ ] `expo-constants` instalado?
- [ ] `app.config.ts` (não `app.json`) configurado com campo `extra`?
- [ ] `src/types/app-config.d.ts` criado com tipagem do `extra`?
- [ ] Valores sensíveis (sem `EXPO_PUBLIC_`) acessados **apenas** via `app.config.ts`, nunca em componentes?

### Scripts e Workflow
- [ ] Script `scripts/set-env.js` criado e funcional?
- [ ] Scripts `env:dev`, `env:staging`, `env:prod` no `package.json`?
- [ ] `npm run env:dev && expo start` funciona corretamente?
- [ ] Variáveis mudam ao trocar de ambiente (full reload necessário)?

### EAS (se aplicável)
- [ ] `eas.json` tem profiles `development`, `staging` e `production` com `env`?
- [ ] Secrets sensíveis criados via `eas secret:create` (não em `.env` files de produção)?
- [ ] `eas env:pull` funciona para sincronizar variáveis?

### Segurança
- [ ] API keys privadas **NÃO** têm prefixo `EXPO_PUBLIC_`?
- [ ] O bundle de produção **NÃO** contém tokens ou secrets?
- [ ] Chamadas que requerem API keys privadas passam por um backend proxy?
