---
name: ci-cd-nest
description: Pipeline de deploy para projetos NestJS — Docker multi-stage build para imagem mínima, GitHub Actions com lint + testes + build + push de imagem, estratégia de migrations em deploy (Prisma e TypeORM), e rollback seguro. Use ao configurar CI/CD do zero, estruturar o Dockerfile de produção, decidir quando rodar migrations, ou definir estratégia de rollback em caso de falha.
---

# CI/CD — Pipeline de Deploy NestJS

Você é um Engenheiro de DevOps Senior especialista em pipelines para projetos NestJS com TypeScript. Sua responsabilidade é garantir que o pipeline de CI/CD seja **seguro**, **automatizado** e **alinhado com os ambientes do projeto** (`development`, `staging`, `production`), usando as melhores práticas do ecossistema Node.js/NestJS.

> **Pré-requisitos:** Esta skill assume que as skills `config-management`, `nest-project-structure`, `testing-strategy-nest` e `prisma-patterns` (ou `typeorm-patterns`) já foram aplicadas. O objeto `env` de `@config/env.config` é a fonte de verdade para variáveis de ambiente. A estratégia de migrations segue o contrato definido em `prisma-patterns.md` (`prisma migrate deploy`) ou `typeorm-patterns.md` (`migration:run`).

---

## Quando usar esta skill

- **Novo projeto:** Configurar o pipeline de CI/CD e o Dockerfile de produção do zero.
- **Dockerfile:** Criar ou otimizar imagem Docker com multi-stage build.
- **GitHub Actions:** Estruturar workflows de CI (PRs) e CD (deploy por branch/tag).
- **Migrations em deploy:** Decidir como e quando rodar `migrate deploy` em produção.
- **Rollback:** Definir estratégia de rollback seguro de imagem e banco de dados.

---

## 1. Visão Geral do Pipeline

```text
EVENTO                          WORKFLOW           AÇÃO
─────────────────────────────────────────────────────────────────────────
PR aberto/atualizado       →    ci.yml          →  Lint + type-check + testes
Push em main               →    deploy-staging  →  Build imagem + push + deploy staging
Tag v*.*.* em main         →    deploy-prod     →  Build imagem + push + deploy produção
Dispatch manual            →    rollback.yml    →  Re-deploy de tag anterior
```

### Regra de Ouro

> - **`ci.yml`** nunca faz deploy — apenas valida qualidade do código.
> - **Imagem Docker** é construída uma única vez por commit SHA e reusada em staging e produção.
> - **Migrations** nunca rodam dentro da imagem Docker (etapa de `RUN`) — sempre no `CMD` ou via job separado no CI.
> - **`synchronize: false`** e `prisma db push` são proibidos em todos os ambientes não-locais.

---

## 2. Docker Multi-Stage Build

O objetivo é uma imagem de produção **mínima** (~150–200 MB) sem código-fonte TypeScript, sem `devDependencies`, sem Nest CLI e sem arquivos de teste.

### Estrutura de 3 estágios

```
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1: deps                                                  │
│  node:20-alpine — instala todas as dependências                 │
├─────────────────────────────────────────────────────────────────┤
│  STAGE 2: builder                                               │
│  node:20-alpine — compila TypeScript, poda devDependencies      │
├─────────────────────────────────────────────────────────────────┤
│  STAGE 3: production                                            │
│  node:20-alpine — copia apenas dist/ + node_modules de prod     │
└─────────────────────────────────────────────────────────────────┘
```

### `Dockerfile` — Padrão do projeto

```dockerfile
# ─────────────────────────────────────────────────────────────────
# STAGE 1: deps — instala todas as dependências (dev + prod)
# ─────────────────────────────────────────────────────────────────
FROM node:20-alpine AS deps

# Instalar libs nativas necessárias no Alpine
RUN apk add --no-cache libc6-compat

WORKDIR /app

# Copiar arquivos de dependência ANTES do código-fonte
# Permite que o Docker reaproveite cache se package.json não mudar
COPY package.json package-lock.json ./

RUN npm ci

# ─────────────────────────────────────────────────────────────────
# STAGE 2: builder — compila TypeScript e poda devDependencies
# ─────────────────────────────────────────────────────────────────
FROM node:20-alpine AS builder

RUN apk add --no-cache libc6-compat

WORKDIR /app

# Reutilizar node_modules do estágio anterior (inclui Nest CLI)
COPY --chown=node:node --from=deps /app/node_modules ./node_modules

# Copiar arquivos de configuração necessários para o build
COPY --chown=node:node package.json package-lock.json ./
COPY --chown=node:node tsconfig.json tsconfig.build.json nest-cli.json ./
COPY --chown=node:node src ./src

# Copiar arquivos do ORM (necessários na imagem para migrations)
# Se usar Prisma:
COPY --chown=node:node prisma ./prisma

# Se usar TypeORM, a pasta de migrations fica em src/ (já copiada acima)

# Compilar TypeScript → dist/
RUN npm run build

# Remover devDependencies do node_modules — apenas prod deps na imagem final
RUN npm prune --production

# ─────────────────────────────────────────────────────────────────
# STAGE 3: production — imagem mínima de runtime
# ─────────────────────────────────────────────────────────────────
FROM node:20-alpine AS production

# Instalar atualizações de segurança
RUN apk update && apk upgrade --no-cache && apk add --no-cache libc6-compat

# Usuário não-root (segurança)
RUN addgroup --system --gid 1001 nodejs \
 && adduser  --system --uid 1001 nestjs

WORKDIR /app

# Copiar apenas o necessário para rodar em produção
COPY --chown=nestjs:nodejs --from=builder /app/dist          ./dist
COPY --chown=nestjs:nodejs --from=builder /app/node_modules  ./node_modules
COPY --chown=nestjs:nodejs --from=builder /app/package.json  ./package.json

# Se usar Prisma: copiar schema e migrations para o container
COPY --chown=nestjs:nodejs --from=builder /app/prisma        ./prisma

USER nestjs

EXPOSE 3000

# CMD executa NO CONTAINER, não no build — garante que migrations rodam
# com acesso ao banco real no momento do deploy
# Veja seção 4 para a estratégia de migrations
CMD ["node", "dist/main.js"]
```

### `.dockerignore` — Obrigatório

```dockerignore
# Dependências locais
node_modules
npm-debug.log*

# Build artifacts
dist
build

# Testes
coverage
**/*.spec.ts
**/*.e2e-spec.ts
test/

# Variáveis de ambiente (NUNCA na imagem)
.env
.env.*
!.env.example

# Git
.git
.gitignore
.github

# Ferramentas de desenvolvimento
.eslintrc.js
.prettierrc
jest.config.js
jest-e2e.json

# Documentação
README.md
docs/

# Docker (evita recursão)
Dockerfile
docker-compose*.yml
```

### Por que `npm prune --production` no stage 2 e não no stage 3?

O Nest CLI (`@nestjs/cli`) é uma `devDependency` necessária apenas para rodar `npm run build`. Ao fazer `prune --production` no stage `builder` após o build, eliminamos as devDependencies de dentro daquele estágio. O stage `production` então copia o `node_modules` já podado — nunca teve acesso ao Nest CLI.

---

## 3. Estrutura de Arquivos de Workflow

```text
.github/
└── workflows/
    ├── ci.yml               ← Qualidade: lint + type-check + testes (todo PR)
    ├── deploy-staging.yml   ← Build + push + deploy em staging (push em main)
    ├── deploy-prod.yml      ← Build + push + deploy em produção (tag v*.*.*)
    └── rollback.yml         ← Re-deploy de tag anterior (manual)
```

---

## 4. Estratégia de Migrations em Deploy

Esta é a decisão mais crítica do pipeline. Existem duas abordagens — o projeto usa a **abordagem 2 (job separado no CI)** como padrão.

### Comparação das abordagens

| Critério | Abordagem 1: run-on-startup (CMD) | Abordagem 2: job separado no CI ✅ |
|---|---|---|
| **Simplicidade** | Alta — um único `CMD` | Média — requer job adicional no workflow |
| **Visibilidade** | Baixa — falhas escondidas nos logs do container | Alta — falha visível no pipeline antes do deploy |
| **Rollback de migration** | Difícil — container pode subir em loop | Seguro — deploy é bloqueado se migration falhar |
| **Múltiplas réplicas** | Race condition — N containers tentam migrar | Sem race condition — roda uma única vez |
| **Recomendado por Prisma** | ❌ Não recomendado para múltiplas réplicas | ✅ Recomendado para CI/CD automatizado |
| **Adequado para** | Projetos simples com 1 réplica | Projetos com N réplicas, staging+prod, blue-green |

### Regra do projeto

> **Use sempre a Abordagem 2 (job no CI) para staging e produção.**
> A Abordagem 1 (run-on-startup) é aceitável apenas em `development` local via `docker-compose`.

### Implementação — Prisma

```yaml
# No workflow de deploy (staging ou produção):
- name: Rodar migrations (Prisma)
  run: |
    npx prisma migrate deploy
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

```bash
# package.json scripts (usados localmente e no CI)
"db:migrate:deploy":  "prisma migrate deploy",
"db:migrate:dev":     "prisma migrate dev",
"db:migrate:status":  "prisma migrate status",
"db:generate":        "prisma generate",
```

> **`prisma migrate deploy`** aplica apenas migrations pendentes — nunca gera novas, nunca reseta o banco. É o único comando seguro para CI/CD.

### Implementação — TypeORM

```yaml
# No workflow de deploy (staging ou produção):
- name: Rodar migrations (TypeORM)
  run: |
    npm run migration:run
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

```bash
# package.json scripts
"migration:run":      "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:run",
"migration:revert":   "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:revert",
"migration:generate": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:generate src/infra/database/migrations/$npm_config_name",
"migration:show":     "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:show",
```

> Ver `typeorm-patterns.md` para configuração do `data-source.ts` e `migrationsRun: false` no `DatabaseModule`.

### Regras absolutas de migrations

| Regra | Detalhe |
|---|---|
| **`synchronize: false` sempre** | Em todos os ambientes, sem exceção — ver `config-management.md` |
| **`prisma db push` proibido em produção** | Apenas para prototipagem local |
| **Migrations no git** | A pasta `prisma/migrations/` (ou `src/infra/database/migrations/`) deve estar versionada |
| **Nunca editar migration aplicada** | Crie uma nova migration para corrigir |
| **`down()` implementado** | Toda migration TypeORM deve ter `down()` testado localmente |
| **Uma migration por PR** | Facilita revisão e rollback granular |

---

## 5. Workflow `ci.yml` — Qualidade em PRs

Roda em **todo pull request**. Deve ser rápido (< 5 minutos).

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
  quality:
    name: Lint, Type-check e Testes
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Instalar dependências
        run: npm ci  # ← sempre npm ci em CI, nunca npm install

      - name: Lint
        run: npm run lint

      - name: Type-check
        run: npm run type-check  # tsc --noEmit

      - name: Testes unitários e integração
        run: npm run test:ci
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          JWT_SECRET: test-secret-must-be-at-least-32-characters-long
          JWT_EXPIRATION: 7d

      - name: Upload coverage
        if: always()
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: false

    # Banco PostgreSQL para testes de integração (Supertest)
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

  build-check:
    name: Verificar build Docker
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: quality

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build Docker (sem push)
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: app:pr-${{ github.event.pull_request.number }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

> **Por que verificar o build Docker no CI?** Garante que o `Dockerfile` continua funcional antes de chegar em `main`. Falhas de build em produção são caras.

> **`concurrency`:** Cancela runs anteriores do mesmo branch — economiza minutos do plano gratuito.

---

## 6. Workflow `deploy-staging.yml` — Deploy Automático em Staging

Roda ao fazer **merge em `main`**. Constrói a imagem, roda migrations e sobe o container.

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [main]

concurrency:
  group: deploy-staging
  cancel-in-progress: true  # cancela deploy anterior se houver novo push

jobs:
  build-and-push:
    name: Build e Push da Imagem
    runs-on: ubuntu-latest
    timeout-minutes: 20
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      sha-tag: ${{ github.sha }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login no Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io  # GitHub Container Registry
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extrair metadados da imagem
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=sha-,format=short
            type=raw,value=staging
            type=raw,value=latest

      - name: Build e Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  migrate:
    name: Rodar Migrations (Staging)
    runs-on: ubuntu-latest
    needs: build-and-push
    timeout-minutes: 10
    environment: staging  # environment com secrets do staging

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Instalar dependências
        run: npm ci

      # Prisma: gerar cliente + rodar migrations
      - name: Gerar Prisma Client
        run: npx prisma generate
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: Rodar migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      # TypeORM: alternativa ao bloco acima
      # - name: Rodar migrations (TypeORM)
      #   run: npm run migration:run
      #   env:
      #     DATABASE_URL: ${{ secrets.DATABASE_URL }}

  deploy:
    name: Deploy em Staging
    runs-on: ubuntu-latest
    needs: migrate
    timeout-minutes: 15
    environment:
      name: staging
      url: https://staging-api.meuapp.com.br

    steps:
      - name: Deploy no servidor
        # Adapte este step para sua infraestrutura:
        # - SSH + docker pull + docker compose up
        # - AWS ECS update-service
        # - Railway / Render / Fly.io deploy
        run: |
          echo "Deploy da imagem ghcr.io/${{ github.repository }}:sha-${{ github.sha }}"
          # Exemplo SSH:
          # ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} \
          #   "docker pull ghcr.io/${{ github.repository }}:sha-${{ github.sha }} && \
          #    docker compose -f /app/docker-compose.yml up -d --no-build"
```

---

## 7. Workflow `deploy-prod.yml` — Deploy em Produção via Tag

Roda ao criar uma **tag semântica** (ex: `v1.2.3`). Reusa a imagem já construída no staging.

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy Produção

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'  # ex: v1.2.3

jobs:
  migrate:
    name: Rodar Migrations (Produção)
    runs-on: ubuntu-latest
    timeout-minutes: 10
    environment: production  # requer aprovação manual se configurado

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Instalar dependências
        run: npm ci

      - name: Gerar Prisma Client
        run: npx prisma generate
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL_PROD }}

      - name: Rodar migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL_PROD }}

  deploy:
    name: Deploy em Produção
    runs-on: ubuntu-latest
    needs: migrate
    timeout-minutes: 15
    environment:
      name: production
      url: https://api.meuapp.com.br

    steps:
      - name: Deploy da tag ${{ github.ref_name }}
        run: |
          echo "Deploy da tag ${{ github.ref_name }}"
          # Ex: re-tag a imagem de staging para a versão e faz deploy
```

---

## 8. Workflow `rollback.yml` — Rollback Manual

Permite redeployar uma tag de imagem anterior em caso de incidente.

```yaml
# .github/workflows/rollback.yml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Ambiente para rollback'
        required: true
        type: choice
        options: [staging, production]
      image_tag:
        description: 'Tag da imagem para reverter (ex: sha-abc1234 ou v1.2.2)'
        required: true
        type: string

jobs:
  rollback:
    name: Rollback ${{ inputs.environment }} → ${{ inputs.image_tag }}
    runs-on: ubuntu-latest
    timeout-minutes: 15
    environment: ${{ inputs.environment }}

    steps:
      - name: Login no Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Re-deploy da imagem ${{ inputs.image_tag }}
        run: |
          echo "Revertendo para ghcr.io/${{ github.repository }}:${{ inputs.image_tag }}"
          # Adapte para sua infraestrutura:
          # ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} \
          #   "docker pull ghcr.io/${{ github.repository }}:${{ inputs.image_tag }} && \
          #    IMAGE_TAG=${{ inputs.image_tag }} docker compose up -d --no-build"

      - name: Notificar rollback
        run: |
          echo "⚠️ Rollback executado: ${{ inputs.environment }} → ${{ inputs.image_tag }}"
          echo "Executor: ${{ github.actor }}"
          echo "Timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

> **Rollback de banco de dados:** O rollback de imagem NÃO reverte automaticamente as migrations. Se a migration foi aplicada, use:
> - Prisma: `prisma migrate deploy` com uma migration de rollback manual criada previamente
> - TypeORM: `npm run migration:revert` (reverte a última migration via `down()`)

---

## 9. Rollback Seguro — Estratégia Completa

### Regras de rollback

| Situação | Ação |
|---|---|
| **Bug no código, sem migration** | Acionar `rollback.yml` com tag anterior — seguro |
| **Bug no código, com migration backward-compatible** | Acionar `rollback.yml` — seguro (ex: coluna adicionada) |
| **Bug no código, com migration breaking (DROP COLUMN)** | ⚠️ Não é possível rollback automático — requer hotfix |
| **Migration falhou no CI** | Deploy bloqueado automaticamente — zero impacto |
| **Migration falhou em produção** | Banco em estado parcial — acionar DBA + migration de hotfix |

### Migrations backward-compatible (Blue-Green safe)

Para permitir rollback seguro em **qualquer cenário**, siga o padrão expand-contract:

```
PR 1: EXPAND   — adiciona nova coluna como nullable (sem breaking change)
Deploy 1:      — migration roda, app antiga continua funcionando
PR 2: MIGRATE  — popula dados na nova coluna, app passa a usar nova coluna
PR 3: CONTRACT — remove coluna antiga (apenas depois de confirmar que nada usa)
```

```sql
-- ✅ EXPAND: seguro para rollback
ALTER TABLE users ADD COLUMN display_name VARCHAR(100);  -- nullable, sem DEFAULT obrigatório

-- ❌ BREAKING: impede rollback
ALTER TABLE users DROP COLUMN name;  -- faça isso apenas no PR de CONTRACT
ALTER TABLE users ALTER COLUMN email SET NOT NULL;  -- pode quebrar app antiga
```

### `docker-compose.yml` — Ambiente local com migrations

```yaml
# docker-compose.yml — desenvolvimento local
version: '3.9'

services:
  api:
    build:
      context: .
      target: production       # ← usa o stage production do Dockerfile
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://postgres:postgres@db:5432/app_dev
      JWT_SECRET: local-dev-secret-must-be-at-least-32-chars
      JWT_EXPIRATION: 7d
    depends_on:
      db:
        condition: service_healthy
    # Para desenvolvimento local: CMD com migration automática
    command: >
      sh -c "npx prisma migrate deploy && node dist/main.js"

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

> **`command: sh -c "npx prisma migrate deploy && node dist/main.js"`** é aceitável **apenas em `docker-compose` local** onde há uma única réplica e conveniência é prioridade. Em staging/produção, o job `migrate` do CI é obrigatório.

---

## 10. Secrets e Variáveis de Ambiente no CI

### Nomenclatura de secrets no GitHub

```
# Secrets de banco (por environment)
DATABASE_URL            ← staging environment
DATABASE_URL_PROD       ← production environment (ou use environments separados)

# Container Registry
GITHUB_TOKEN            ← automático (GitHub Container Registry)

# Deploy (se necessário)
SERVER_HOST             ← IP/hostname do servidor
SERVER_USER             ← usuário SSH
SSH_PRIVATE_KEY         ← chave SSH para deploy

# Observabilidade (ver monitoring-apm.md)
SENTRY_DSN
SENTRY_AUTH_TOKEN
```

### Configurar GitHub Environments

Crie environments no GitHub (`Settings → Environments`):

| Environment | Branches | Proteção recomendada |
|---|---|---|
| `staging` | `main` | Sem aprovação obrigatória |
| `production` | Tags `v*.*.*` | Aprovação obrigatória de 1 reviewer |

> Environments no GitHub permitem secrets **por ambiente** — evita usar o mesmo `DATABASE_URL` para staging e produção.

---

## 11. Scripts do `package.json` — Referência Completa

```json
{
  "scripts": {
    // Build
    "build": "nest build",
    "start:prod": "node dist/main.js",

    // Qualidade (usados no ci.yml)
    "lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:ci": "jest --ci --coverage --forceExit --runInBand",
    "test:e2e": "jest --config ./test/jest-e2e.json",

    // Prisma (ver prisma-patterns.md)
    "db:generate": "prisma generate",
    "db:migrate:dev": "prisma migrate dev",
    "db:migrate:deploy": "prisma migrate deploy",
    "db:migrate:status": "prisma migrate status",
    "db:migrate:reset": "prisma migrate reset",
    "db:studio": "prisma studio",

    // TypeORM (ver typeorm-patterns.md) — use UM dos dois ORMs, não ambos
    "migration:generate": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:generate src/infra/database/migrations/$npm_config_name",
    "migration:run": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:run",
    "migration:revert": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:revert",
    "migration:show": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:show"
  }
}
```

---

## 12. Checklist de CI/CD

### Dockerfile
- [ ] Multi-stage com 3 estágios: `deps` → `builder` → `production`?
- [ ] Imagem base `node:20-alpine`?
- [ ] `npm ci` no lugar de `npm install`?
- [ ] `npm prune --production` roda no stage `builder` após o build?
- [ ] `COPY package*.json` vem antes de `COPY src` (cache de camadas)?
- [ ] Usuário não-root configurado (`addgroup` + `adduser`)?
- [ ] `EXPOSE 3000` declarado?
- [ ] `.dockerignore` criado com `node_modules`, `.env`, `coverage`, `**/*.spec.ts`?
- [ ] Pasta `prisma/` (ou `src/infra/database/migrations/`) copiada para a imagem?
- [ ] `CMD` usa `node dist/main.js` (não `nest start`)?

### GitHub Actions — ci.yml
- [ ] Dispara em `pull_request` e `push` para `main`?
- [ ] `concurrency` configurado para cancelar runs anteriores?
- [ ] `npm ci` no lugar de `npm install`?
- [ ] Lint (`npm run lint`) roda antes dos testes?
- [ ] Type-check (`npm run type-check`) roda antes dos testes?
- [ ] Testes usam `npm run test:ci` (com `--ci --coverage --forceExit`)?
- [ ] Service `postgres` configurado para testes de integração?
- [ ] Build Docker verificado sem push no CI?

### GitHub Actions — deploy
- [ ] Job `migrate` roda **antes** do job `deploy`?
- [ ] `migrate` e `deploy` usam o `environment` correto (com secrets separados)?
- [ ] `production` environment tem aprovação obrigatória configurada?
- [ ] Imagem tagueada com SHA do commit (rastreabilidade)?
- [ ] `rollback.yml` com `workflow_dispatch` configurado?
- [ ] Secrets `DATABASE_URL` separados por environment (staging vs production)?

### Migrations
- [ ] `synchronize: false` em todos os ambientes (ver `config-management.md`)?
- [ ] `prisma migrate deploy` (ou `migration:run`) roda no job CI, não no `CMD`?
- [ ] Migrations estão commitadas no git?
- [ ] Migrations são backward-compatible (padrão expand-contract)?
- [ ] `down()` implementado em todas as migrations TypeORM?
- [ ] `prisma db push` e `migrate dev` nunca rodam em staging/produção?

### Rollback
- [ ] Tags de imagem com SHA do commit preservadas no registry?
- [ ] `rollback.yml` testado em staging antes de usar em produção?
- [ ] Equipe sabe o procedimento para rollback de migration (TypeORM: `migration:revert` / Prisma: migration manual de down)?

---

## Referências

- [config-management.md](../configuration-and-environment/config-management.md) — `env.config.ts`, `synchronize: false`, variáveis por ambiente
- [prisma-patterns.md](../database/prisma-patterns.md) — `prisma migrate deploy`, scripts, `prisma generate`
- [typeorm-patterns.md](../database/typeorm-patterns.md) — `data-source.ts`, `migration:run`, `migrationsRun: false`
- [testing-strategy-nest.md](../quality-and-testing/testing-strategy-nest.md) — `test:ci`, TestContainers, configuração de Jest
- [monitoring-apm.md](./monitoring-apm.md) — upload de source maps para Sentry no pipeline de release
- [logging-pattern.md](./logging-pattern.md) — logs estruturados em produção
- [ci-cd-pipeline.md](../../frontend/infrastructure-and-deployment/ci-cd-pipeline.md) — pipeline do frontend (Expo/React Native) — usa mesma estrutura de environments e branches
- [Docker Docs — Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [GitHub Actions — Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [Prisma Docs — migrate deploy](https://www.prisma.io/docs/orm/prisma-migrate/workflows/development-and-production)
- [notiz.dev — Prisma Migrate Deploy with Docker](https://notiz.dev/blog/prisma-migrate-deploy-with-docker/)