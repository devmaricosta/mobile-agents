---
name: ci-cd-pipeline
description: Pipeline de CI/CD com GitHub Actions para projetos Expo SDK 54+. Cobre lint e testes em PRs, builds automáticos com EAS Build por ambiente (development, staging, production), distribuição para QA via EAS Update (OTA) e publicação nas stores via EAS Submit.
---

# Pipeline CI/CD — GitHub Actions com EAS Build e EAS Update

Você é um Engenheiro de DevOps Senior especialista em pipelines para projetos React Native com Expo. Sua responsabilidade é garantir que o pipeline de CI/CD seja **seguro**, **automatizado** e **alinhado com os ambientes do projeto** (development, staging, production), usando as melhores práticas do ecossistema Expo SDK 54.

> **Pré-requisito:** Esta skill assume que a skill `environment-config` já foi configurada — os profiles `development`, `staging` e `production` do `eas.json` com suas variáveis de ambiente são a fonte de verdade dos ambientes.

---

## Quando usar esta skill

- **Novo projeto:** Configurar o pipeline de CI/CD do zero.
- **PRs:** Garantir que lint, type-check e testes rodem em todo pull request.
- **Builds:** Automatizar EAS Build por ambiente ao fazer push em branches específicas.
- **OTA para QA:** Publicar updates via EAS Update sem precisar de um novo build nativo.
- **Release nas stores:** Automatizar o envio para App Store e Google Play via EAS Submit.

---

## 1. Visão Geral do Pipeline

```text
EVENTO                        WORKFLOW           AÇÃO
──────────────────────────────────────────────────────────────────────
PR aberto/atualizado     →    ci.yml          →  Lint, type-check, testes
Push em main             →    deploy-qa.yml   →  EAS Update (OTA) → canal "staging"
Tag v*.*.* em main       →    release.yml     →  EAS Build (prod) → EAS Submit → stores
Dispatch manual          →    build.yml       →  EAS Build por ambiente e plataforma
```

### Regra de Ouro

> - **EAS Update (OTA)** é para mudanças em código JavaScript/TypeScript — não requer novo binary.
> - **EAS Build** é obrigatório quando há mudanças em código nativo (dependências com módulos nativos, `app.config.ts`, arquivos em `android/`/`ios/`).
> - Use o **Expo Fingerprint** para detectar automaticamente se um novo build nativo é necessário.

---

## 2. Estrutura de Arquivos de Workflow

```text
.github/
└── workflows/
    ├── ci.yml            # Lint, type-check e testes (todo PR)
    ├── deploy-qa.yml     # EAS Update OTA para QA (push em main)
    ├── build.yml         # EAS Build manual por ambiente
    └── release.yml       # EAS Build prod + EAS Submit nas stores (tag)
```

---

## 3. Pré-requisitos e Segredos

### 3.1 Instalar EAS CLI globalmente no projeto

```bash
npm install --save-dev eas-cli
```

> Use `eas-cli` como dev dependency para garantir versão fixada no projeto e evitar surpresas de breaking changes no CI.

### 3.2 Configurar `eas.json`

O `eas.json` é a fonte de verdade dos profiles de build. Deve estar alinhado com o que a skill `environment-config` define:

```json
{
  "cli": {
    "version": ">= 15.0.0",
    "appVersionSource": "remote"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "channel": "development",
      "env": {
        "EXPO_PUBLIC_APP_ENV": "development",
        "EXPO_PUBLIC_API_URL": "http://localhost:3000/api",
        "EXPO_PUBLIC_ENABLE_LOGS": "true"
      }
    },
    "staging": {
      "distribution": "internal",
      "channel": "staging",
      "env": {
        "EXPO_PUBLIC_APP_ENV": "staging",
        "EXPO_PUBLIC_API_URL": "https://staging-api.meuapp.com.br/api",
        "EXPO_PUBLIC_ENABLE_LOGS": "true"
      }
    },
    "production": {
      "autoIncrement": true,
      "channel": "production",
      "env": {
        "EXPO_PUBLIC_APP_ENV": "production",
        "EXPO_PUBLIC_API_URL": "https://api.meuapp.com.br/api",
        "EXPO_PUBLIC_ENABLE_LOGS": "false"
      }
    }
  },
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./google-service-account.json",
        "track": "internal"
      },
      "ios": {
        "appleId": "seu-apple-id@exemplo.com",
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDEF1234"
      }
    }
  }
}
```

> **Nota:** `"appVersionSource": "remote"` delega o controle de versão (`versionCode`/`buildNumber`) ao EAS, evitando conflitos de versão entre builds locais e da CI.

### 3.2 Secrets necessários no GitHub

Configure os seguintes secrets em **Settings → Secrets and variables → Actions** do repositório:

| Secret | Descrição | Como obter |
|---|---|---|
| `EXPO_TOKEN` | Token pessoal de acesso ao Expo | `expo whoami` → Account Settings → Access Tokens |
| `APPLE_APP_STORE_CONNECT_API_KEY_ID` | ID da chave da App Store Connect API | App Store Connect → Users → Keys |
| `APPLE_APP_STORE_CONNECT_API_KEY_ISSUER_ID` | Issuer ID da chave | App Store Connect → Users → Keys |
| `APPLE_APP_STORE_CONNECT_API_KEY_CONTENT` | Conteúdo `.p8` em base64 | `base64 -i AuthKey_XXXXX.p8` |
| `GOOGLE_SERVICE_ACCOUNT_KEY` | JSON da service account do Google Play | Google Play Console → Setup → API Access |

> **Segurança:** Nunca commite esses valores no repositório. Use sempre GitHub Secrets. Variáveis sensíveis de build (não-`EXPO_PUBLIC_`) devem ser configuradas como **EAS Secrets** via `eas secret:create`.

---

## 4. Workflow `ci.yml` — Qualidade em PRs

Roda em **todo pull request** e em **pushes para `main`**. Deve ser rápido (< 3 minutos) para não bloquear o desenvolvedor.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint-and-test:
    name: Lint, Type-check e Testes
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Instalar dependências
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type-check
        run: npm run type-check

      - name: Testes (CI)
        run: npm run test:ci
        env:
          # Variáveis de ambiente para o ambiente de testes
          EXPO_PUBLIC_APP_ENV: development
          EXPO_PUBLIC_API_URL: http://localhost:3000/api
          EXPO_PUBLIC_ENABLE_LOGS: "true"

      - name: Upload coverage (opcional)
        if: always()
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: false
```

> **Por que `npm ci` e não `npm install`?** O `npm ci` lê o `package-lock.json` e instala exatamente as versões fixadas, garantindo reproducibilidade entre ambientes. Nunca use `npm install` em CI.

> **`concurrency`:** Cancela runs anteriores do mesmo branch ao receber um novo push — economiza minutos do plano.

---

## 5. Workflow `deploy-qa.yml` — EAS Update OTA para QA

Roda ao fazer **merge em `main`**. Publica uma atualização OTA no canal `staging` via EAS Update. Ideal para QA testar mudanças de JavaScript sem precisar de um novo build nativo.

```yaml
# .github/workflows/deploy-qa.yml
name: Deploy QA (EAS Update)

on:
  push:
    branches: [main]

concurrency:
  group: deploy-qa
  cancel-in-progress: true

jobs:
  update:
    name: Publicar EAS Update (canal staging)
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Setup Expo e EAS CLI
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Instalar dependências
        run: npm ci

      - name: Verificar se build nativo é necessário (Fingerprint)
        id: fingerprint
        run: |
          FINGERPRINT=$(npx expo-updates fingerprint:generate --platform all 2>/dev/null || echo "unknown")
          echo "fingerprint=$FINGERPRINT" >> $GITHUB_OUTPUT

      - name: Publicar EAS Update
        run: |
          eas update \
            --channel staging \
            --message "Deploy de ${{ github.sha }} por ${{ github.actor }}" \
            --non-interactive
        env:
          EXPO_PUBLIC_APP_ENV: staging
          EXPO_PUBLIC_API_URL: https://staging-api.meuapp.com.br/api
          EXPO_PUBLIC_ENABLE_LOGS: "true"
```

> **Canal `staging`:** O EAS Update usa canais para segmentar quem recebe cada update. O canal `staging` entrega para usuários com o **staging build** instalado (mapeado em `eas.json` via `"channel": "staging"`).

---

## 6. Workflow `build.yml` — EAS Build Manual por Ambiente

Permite triggerar builds manualmente via **workflow_dispatch**. Útil para gerar builds de desenvolvimento para QA ou staging sem criar um release.

```yaml
# .github/workflows/build.yml
name: EAS Build (Manual)

on:
  workflow_dispatch:
    inputs:
      profile:
        description: "Perfil de build"
        required: true
        default: staging
        type: choice
        options:
          - development
          - staging
          - production
      platform:
        description: "Plataforma"
        required: true
        default: all
        type: choice
        options:
          - all
          - android
          - ios

jobs:
  build:
    name: Build ${{ inputs.platform }} (${{ inputs.profile }})
    runs-on: ubuntu-latest
    timeout-minutes: 60

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Setup Expo e EAS CLI
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Instalar dependências
        run: npm ci

      - name: EAS Build
        run: |
          eas build \
            --platform ${{ inputs.platform }} \
            --profile ${{ inputs.profile }} \
            --non-interactive \
            --no-wait
```

> **`--no-wait`:** Dispara o build no EAS e não bloqueia o runner do GitHub Actions. O build continua na fila do EAS e você acompanha pelo painel do Expo. Use `--wait` se precisar que o workflow aguarde o build concluir (ex: antes de submeter).

---

## 7. Workflow `release.yml` — Release para as Stores

Roda ao **criar uma tag** no formato `v*.*.*` (ex: `v1.2.0`). Executa o EAS Build de produção e em seguida o EAS Submit para as stores.

```yaml
# .github/workflows/release.yml
name: Release (EAS Build + Submit)

on:
  push:
    tags:
      - "v*.*.*"

jobs:
  build-and-submit:
    name: Build e Submit para as Stores
    runs-on: ubuntu-latest
    timeout-minutes: 90
    environment: production      # Aciona environment protection rules do GitHub

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Setup Expo e EAS CLI
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Instalar dependências
        run: npm ci

      - name: EAS Build (produção) + Submit automático
        run: |
          eas build \
            --platform all \
            --profile production \
            --non-interactive \
            --auto-submit \
            --wait
        env:
          # Credenciais Apple (App Store Connect API Key)
          EXPO_APPLE_APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APPLE_APP_STORE_CONNECT_API_KEY_ID }}
          EXPO_APPLE_APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.APPLE_APP_STORE_CONNECT_API_KEY_ISSUER_ID }}
          EXPO_APPLE_APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.APPLE_APP_STORE_CONNECT_API_KEY_CONTENT }}
          # Credenciais Google Play
          GOOGLE_SERVICE_ACCOUNT_KEY: ${{ secrets.GOOGLE_SERVICE_ACCOUNT_KEY }}
```

> **`--auto-submit`:** Encadeia `eas build` + `eas submit` automaticamente após o build concluir. Equivale a rodar `eas submit` com o artifact gerado.
>
> **`environment: production`:** Aplica as regras de proteção configuradas no GitHub (ex: requer aprovação manual de um revisor antes de rodar o job). Recomendado para deploys em produção.

---

## 8. Fluxo Completo de Release (Passo a Passo)

```text
Correção de bug / nova feature
   ↓
1. Criar branch: git checkout -b fix/login-crash
   ↓
2. Commitar: git commit -m "fix(auth): prevent login crash on expired token"
   ↓
3. Abrir PR → ci.yml roda automaticamente (lint + type-check + testes)
   ↓
4. PR aprovado → merge em main
   ↓
5. deploy-qa.yml roda → EAS Update publica OTA no canal "staging"
   ↓
6. QA testa no dispositivo com staging build instalado
   ↓
7. Criar tag: git tag v1.2.1 && git push origin v1.2.1
   ↓
8. release.yml roda → EAS Build (prod) + EAS Submit → App Store + Google Play
```

---

## 9. Estratégia de Canais EAS Update

O EAS Update usa **canais** para segmentar quem recebe cada OTA update. O mapeamento deve respeitar os profiles do `eas.json`:

| Canal EAS | Profile de Build | Público Alvo |
|---|---|---|
| `development` | `development` | Desenvolvedores com dev client |
| `staging` | `staging` | Time de QA interno |
| `production` | `production` | Usuários finais nas stores |

### Verificação de compatibilidade (Fingerprint)

> **Regra de Ouro:** Um EAS Update só pode ser entregue para um build nativo cujo **fingerprint** seja compatível. Se o fingerprint mudou (ex: nova dependência nativa instalada), um novo **EAS Build** é obrigatório antes de publicar o update.

```bash
# Verificar o fingerprint atual do projeto
npx expo-updates fingerprint:generate --platform all

# Comparar fingerprint de dois commits para decidir se precisa de build novo
npx expo-updates fingerprint:difference --platform ios HEAD~1..HEAD
```

---

## 10. Scripts no `package.json`

Adicione os scripts relacionados à CI (complementando os scripts da skill `setup-react-native`):

```json
{
  "scripts": {
    "lint": "npx expo lint",
    "lint:fix": "eslint src --fix",
    "type-check": "tsc --noEmit",
    "test:ci": "jest --ci --coverage --maxWorkers=2 --forceExit",
    "build:dev": "eas build --profile development --platform all --non-interactive",
    "build:staging": "eas build --profile staging --platform all --non-interactive",
    "build:prod": "eas build --profile production --platform all --non-interactive",
    "update:staging": "eas update --channel staging --non-interactive",
    "update:prod": "eas update --channel production --non-interactive",
    "submit:prod": "eas submit --platform all --profile production --non-interactive"
  }
}
```

---

## 11. Proteção de Branches e Environments

### Branch Protection em `main`

Configure em **Settings → Branches → Branch protection rules** para `main`:

- [x] **Require a pull request before merging**
- [x] **Require status checks to pass before merging** → selecionar `Lint, Type-check e Testes`
- [x] **Require branches to be up to date before merging**
- [x] **Restrict who can push to matching branches**

### Environment Protection para `production`

Configure em **Settings → Environments → production**:

- [x] **Required reviewers** → adicionar pelo menos 1 revisor obrigatório
- [x] **Deployment branches** → somente tags `v*.*.*`

---

## 12. Cache e Otimizações

### Cache de `node_modules` com `actions/cache`

O `actions/setup-node@v4` com `cache: npm` já cuida do cache automático do `npm`. Para cenários mais avançados:

```yaml
- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### Paralelismo de jobs

Quando lint e testes são independentes, rode em paralelo para reduzir o tempo total:

```yaml
jobs:
  lint:
    name: Lint e Type-check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  test:
    name: Testes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run test:ci
        env:
          EXPO_PUBLIC_APP_ENV: development
          EXPO_PUBLIC_API_URL: http://localhost:3000/api
          EXPO_PUBLIC_ENABLE_LOGS: "true"
```

---

## 13. Integração com Husky (pré-push)

A skill `setup-react-native` já configura o Husky com `pre-commit` (lint-staged). Para garantir que testes passem antes do push:

```bash
# .husky/pre-push
npm run test:ci
```

> **Por que no pre-push e não pre-commit?** Testes são lentos demais para pre-commit. O pre-push é o último portão local antes de acionar a CI/CD remota. Se os testes passam localmente no pre-push, a chance de falha na CI diminui drasticamente.

---

## 14. Checklist de Verificação

### Configuração Inicial
- [ ] `eas-cli` instalado como dev dependency (`npm install --save-dev eas-cli`)?
- [ ] Projeto configurado no EAS: `eas init` rodado?
- [ ] `eas.json` com profiles `development`, `staging` e `production` alinhado com `environment-config`?
- [ ] `"appVersionSource": "remote"` configurado no `eas.json`?
- [ ] Primeiro build local rodado para cada plataforma (configura credenciais no EAS)?

### Segredos do GitHub
- [ ] `EXPO_TOKEN` configurado?
- [ ] Credenciais Apple e Google configuradas como GitHub Secrets?
- [ ] Variáveis sensíveis de build configuradas como EAS Secrets (`eas secret:create`)?

### Workflows
- [ ] `ci.yml` rodando em PRs e push para `main`?
- [ ] `deploy-qa.yml` publicando OTA ao fazer merge em `main`?
- [ ] `build.yml` disponível para builds manuais via workflow_dispatch?
- [ ] `release.yml` acionado por tags `v*.*.*`?

### Branch Protection
- [ ] Branch `main` protegida (PR obrigatório, status checks obrigatórios)?
- [ ] Environment `production` com revisores obrigatórios?

### Scripts
- [ ] Scripts `build:dev`, `build:staging`, `build:prod` no `package.json`?
- [ ] Scripts `update:staging`, `update:prod` no `package.json`?
- [ ] `.husky/pre-push` configurado com `npm run test:ci`?

### EAS Update
- [ ] Canais de update (`development`, `staging`, `production`) criados via `eas channel:create`?
- [ ] Fingerprint verificado antes de publicar update (mudanças nativas exigem novo build)?
- [ ] Builds instalados nos dispositivos de QA apontam para o canal correto?
