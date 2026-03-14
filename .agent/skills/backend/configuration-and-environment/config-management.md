---
name: config-management
description: Gerenciamento de configuração com @nestjs/config — validação de variáveis de ambiente com Zod na inicialização da aplicação, separação por namespace (database.config.ts, jwt.config.ts), uso de ConfigService sem strings mágicas, e diferenciação entre ambientes. Esta skill expande a seção `src/config/` definida em `nest-project-structure.md`.
---

# Gerenciamento de Configuração — @nestjs/config com Zod

Você é um Arquiteto de Software Senior. Esta skill complementa `nest-project-structure.md` (onde fica `src/config/` e o alias `@config/*`) e `nest-modules-pattern.md` (como `forRootAsync()` consome o `ConfigService`). O foco aqui é **como estruturar, validar e consumir configurações de forma segura e sem strings mágicas**.

> **Regra absoluta (herdada de `nest-project-structure.md`):** Nunca acesse `process.env` fora de `src/config/`. Qualquer arquivo de módulo, use case, service ou repository que precise de um valor de ambiente deve recebê-lo via `ConfigService` ou via injeção do objeto de configuração do namespace.

---

## 1. Visão Geral da Estratégia

```
src/config/
├── env.config.ts          ← Validação completa de process.env com Zod (roda no bootstrap)
├── database.config.ts     ← Namespace "database" (registerAs)
├── jwt.config.ts          ← Namespace "jwt" (registerAs)
└── app.config.ts          ← Namespace "app" (porta, NODE_ENV, etc.)
```

**Fluxo:**

```
.env file
   ↓
ConfigModule.forRoot() ← carrega e expõe process.env
   ↓
env.config.ts (Zod) ← valida estrutura e tipos no bootstrap (falha rápido)
   ↓
*.config.ts (registerAs) ← organiza por namespace com tipagem explícita
   ↓
ConfigService.get('database.host') ← consumo com tipos, sem strings mágicas soltas
```

---

## 2. Instalação e Setup

```bash
npm install @nestjs/config zod
```

### `app.module.ts` — Registro do ConfigModule

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

import databaseConfig from '@config/database.config';
import jwtConfig from '@config/jwt.config';
import appConfig from '@config/app.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,          // ← disponível em todos os módulos sem reimportar
      envFilePath: [           // ← ordem de prioridade: mais específico primeiro
        `.env.${process.env.NODE_ENV ?? 'development'}.local`,
        `.env.${process.env.NODE_ENV ?? 'development'}`,
        '.env.local',
        '.env',
      ],
      load: [appConfig, databaseConfig, jwtConfig], // ← namespaces carregados
    }),
    // ... demais módulos de infra e domínio
  ],
})
export class AppModule {}
```

> **`isGlobal: true` é obrigatório neste projeto.** Evita que cada módulo precise importar `ConfigModule` explicitamente. Registrar uma única vez no `AppModule` é suficiente. Isso é consistente com o padrão `@Global()` de `nest-modules-pattern.md`.

---

## 3. Validação com Zod — `env.config.ts`

O `env.config.ts` é o **guardião da aplicação**: se qualquer variável obrigatória estiver ausente ou mal formatada, a aplicação **não sobe**. É preferível falhar rápido no bootstrap a ter um erro de runtime misterioso em produção.

```typescript
// src/config/env.config.ts
import { z } from 'zod';

const envSchema = z.object({
  // Aplicação
  NODE_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  PORT: z.coerce.number().int().positive().default(3000),

  // Banco de dados
  DATABASE_HOST: z.string().min(1),
  DATABASE_PORT: z.coerce.number().int().positive().default(5432),
  DATABASE_NAME: z.string().min(1),
  DATABASE_USER: z.string().min(1),
  DATABASE_PASSWORD: z.string().min(1),
  DATABASE_SSL: z.coerce.boolean().default(false),

  // JWT
  JWT_SECRET: z.string().min(32, 'JWT_SECRET deve ter no mínimo 32 caracteres'),
  JWT_EXPIRATION: z.string().default('7d'),
  JWT_REFRESH_SECRET: z.string().min(32).optional(),
  JWT_REFRESH_EXPIRATION: z.string().default('30d'),
});

// Inferência de tipo TypeScript a partir do schema Zod
export type Env = z.infer<typeof envSchema>;

const _parsed = envSchema.safeParse(process.env);

if (!_parsed.success) {
  console.error('❌ Variáveis de ambiente inválidas:\n', _parsed.error.format());
  process.exit(1); // ← falha rápida no bootstrap
}

// Objeto tipado e imutável — única fonte da verdade
export const env: Readonly<Env> = Object.freeze(_parsed.data);
```

**Por que `safeParse` + `process.exit(1)` e não `parse()`?**
- `parse()` lança uma exceção que pode ser suprimida ou mal tratada
- `safeParse()` + log formatado + `process.exit(1)` garante mensagem clara e saída imediata

---

## 4. Namespaces — `registerAs`

`registerAs` cria uma "fatia" da configuração com um namespace. Isso permite acessar configurações relacionadas de forma agrupada e tipada, sem precisar de strings mágicas completas como `'database.host'`.

### 4.1 `database.config.ts`

```typescript
// src/config/database.config.ts
import { registerAs } from '@nestjs/config';
import { env } from './env.config'; // ← reutiliza a validação já feita

export interface DatabaseConfig {
  host: string;
  port: number;
  name: string;
  user: string;
  password: string;
  ssl: boolean;
}

export default registerAs('database', (): DatabaseConfig => ({
  host: env.DATABASE_HOST,
  port: env.DATABASE_PORT,
  name: env.DATABASE_NAME,
  user: env.DATABASE_USER,
  password: env.DATABASE_PASSWORD,
  ssl: env.DATABASE_SSL,
}));
```

### 4.2 `jwt.config.ts`

```typescript
// src/config/jwt.config.ts
import { registerAs } from '@nestjs/config';
import { env } from './env.config';

export interface JwtConfig {
  secret: string;
  expiration: string;
  refreshSecret: string | undefined;
  refreshExpiration: string;
}

export default registerAs('jwt', (): JwtConfig => ({
  secret: env.JWT_SECRET,
  expiration: env.JWT_EXPIRATION,
  refreshSecret: env.JWT_REFRESH_SECRET,
  refreshExpiration: env.JWT_REFRESH_EXPIRATION,
}));
```

### 4.3 `app.config.ts`

```typescript
// src/config/app.config.ts
import { registerAs } from '@nestjs/config';
import { env } from './env.config';

export type AppEnvironment = 'development' | 'staging' | 'production';

export interface AppConfig {
  env: AppEnvironment;
  port: number;
  isDevelopment: boolean;
  isProduction: boolean;
}

export default registerAs('app', (): AppConfig => ({
  env: env.NODE_ENV,
  port: env.PORT,
  isDevelopment: env.NODE_ENV === 'development',
  isProduction: env.NODE_ENV === 'production',
}));
```

---

## 5. Consumo com `ConfigService` — Sem Strings Mágicas

### ❌ Anti-padrão: strings mágicas soltas

```typescript
// ❌ ERRADO — string mágica, sem autocomplete, sem tipagem
const host = this.configService.get<string>('database.host');
const secret = this.configService.get<string>('JWT_SECRET');
// Silenciosamente retorna undefined se o nome estiver errado
```

### ✅ Padrão correto: injeção do namespace via token

```typescript
// ✅ CORRETO — injeta o objeto de configuração inteiro do namespace
import { Inject, Injectable } from '@nestjs/common';
import { ConfigType } from '@nestjs/config';
import databaseConfig from '@config/database.config';

@Injectable()
export class DatabaseService {
  constructor(
    @Inject(databaseConfig.KEY)
    private readonly dbConfig: ConfigType<typeof databaseConfig>,
  ) {}

  getConnectionString(): string {
    // Autocomplete e tipo garantidos pelo TypeScript
    return `postgresql://${this.dbConfig.user}:${this.dbConfig.password}@${this.dbConfig.host}:${this.dbConfig.port}/${this.dbConfig.name}`;
  }
}
```

**Por que `databaseConfig.KEY` e não a string `'database'`?**
- `databaseConfig.KEY` é o token simbólico gerado por `registerAs`. Se você renomear o namespace, o TypeScript avisará que a referência está quebrada.
- `ConfigType<typeof databaseConfig>` infere automaticamente o tipo de retorno do factory function do namespace.

### Quando usar `ConfigService.get()` ainda é aceitável

Use `configService.get<T>('namespace.key')` apenas em bootstraps e factories onde a injeção de dependência ainda não está disponível:

```typescript
// app.module.ts — forRootAsync (aceitável — é infraestrutura de bootstrap)
TypeOrmModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    type: 'postgres',
    host: config.get<string>('database.host'),
    port: config.get<number>('database.port'),
    // ...
  }),
})
```

> Para evitar até esse caso, prefira injetar o namespace diretamente (ver seção 6).

---

## 6. Integração com `DatabaseModule` e `JwtModule`

### Consumindo namespace no `forRootAsync` sem strings mágicas

```typescript
// src/infra/database/database.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigType } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import databaseConfig from '@config/database.config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [databaseConfig.KEY],      // ← injeta o token do namespace
      useFactory: (dbConfig: ConfigType<typeof databaseConfig>) => ({
        type: 'postgres',
        host: dbConfig.host,             // ← tipado, sem string mágica
        port: dbConfig.port,
        database: dbConfig.name,
        username: dbConfig.user,
        password: dbConfig.password,
        ssl: dbConfig.ssl,
        entities: [__dirname + '/../../**/*.entity{.ts,.js}'],
        synchronize: false,              // ← SEMPRE false em produção
        migrations: [__dirname + '/migrations/**/*{.ts,.js}'],
        migrationsRun: true,
      }),
    }),
  ],
})
export class DatabaseModule {}
```

```typescript
// src/modules/auth/auth.module.ts (trecho do JwtModule)
import { ConfigModule, ConfigType } from '@nestjs/config';
import { JwtModule } from '@nestjs/jwt';
import jwtConfig from '@config/jwt.config';

JwtModule.registerAsync({
  imports: [ConfigModule],
  inject: [jwtConfig.KEY],
  useFactory: (jwt: ConfigType<typeof jwtConfig>) => ({
    secret: jwt.secret,
    signOptions: { expiresIn: jwt.expiration },
  }),
})
```

---

## 7. Diferenciação de Ambientes

### Estratégia de arquivos `.env`

```
.env                        ← valores padrão / fallback (não sobe para git se conter segredos)
.env.development            ← valores para ambiente local
.env.staging                ← valores para staging (gerenciado por CI/CD)
.env.production             ← valores para produção (gerenciado por CI/CD)
.env.local                  ← overrides pessoais do dev (sempre no .gitignore)
.env.development.local      ← overrides locais específicos do ambiente
```

**Prioridade de carregamento** (mais alto → mais baixo):
1. `.env.development.local`
2. `.env.development`
3. `.env.local`
4. `.env`

### Lógica condicional baseada em ambiente

```typescript
// Use o objeto `env` importado de env.config.ts — nunca process.env diretamente
import { env } from '@config/env.config';

// ✅ CORRETO
if (env.NODE_ENV === 'production') {
  // configuração específica de produção
}

// OU use o namespace app.config.ts
import { Inject } from '@nestjs/common';
import { ConfigType } from '@nestjs/config';
import appConfig from '@config/app.config';

@Injectable()
export class SomeService {
  constructor(
    @Inject(appConfig.KEY)
    private readonly app: ConfigType<typeof appConfig>,
  ) {}

  doSomething() {
    if (this.app.isProduction) {
      // comportamento de produção
    }
  }
}
```

### `.gitignore` obrigatório

```gitignore
# Variáveis de ambiente — nunca comitar arquivos com segredos
.env
.env.local
.env.*.local
.env.production
.env.staging
# Manter: .env.development (apenas valores não-sensíveis de dev)
# .env.example deve sempre estar commitado como referência
```

### `.env.example` — Documentação viva

```bash
# .env.example — copie para .env e preencha os valores
# Nunca deixe valores reais neste arquivo

# Aplicação
NODE_ENV=development
PORT=3000

# Banco de dados
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=myapp_dev
DATABASE_USER=postgres
DATABASE_PASSWORD=
DATABASE_SSL=false

# JWT (mínimo 32 caracteres)
JWT_SECRET=
JWT_EXPIRATION=7d
JWT_REFRESH_SECRET=
JWT_REFRESH_EXPIRATION=30d
```

---

## 8. Padrão de `main.ts` — Validação no Bootstrap

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { env } from '@config/env.config'; // ← importar aqui garante que Zod rode PRIMEIRO

async function bootstrap() {
  // env.config.ts já foi importado e executou o process.exit(1) se inválido
  // Se chegou aqui, todas as variáveis são válidas

  const app = await NestFactory.create(AppModule);

  await app.listen(env.PORT);
  console.log(`🚀 Servidor rodando na porta ${env.PORT} (${env.NODE_ENV})`);
}

bootstrap();
```

> **Importante:** Importar `env` de `@config/env.config` no `main.ts` garante a execução do Zod **antes** do bootstrap do NestJS. Mesmo que outros módulos não importem `env.config.ts` diretamente, a validação já terá rodado, e a aplicação já terá falhado se necessário.

---

## 9. Anti-padrões Comuns

### ❌ Acessar `process.env` fora de `src/config/`

```typescript
// ❌ ERRADO — acesso direto em módulo de feature
@Injectable()
export class AuthService {
  private readonly jwtSecret = process.env.JWT_SECRET; // ← PROIBIDO
}

// ✅ CORRETO
@Injectable()
export class AuthService {
  constructor(
    @Inject(jwtConfig.KEY)
    private readonly jwt: ConfigType<typeof jwtConfig>,
  ) {}
  private readonly jwtSecret = this.jwt.secret;
}
```

### ❌ Usar `ConfigService` com strings arbitrárias

```typescript
// ❌ ERRADO — string mágica, sem autocomplete, sem garantia de tipo
const secret = this.configService.get('JWT_SECRET'); // retorna string | undefined
const host = this.configService.get<string>('database.host'); // pode ser undefined!

// ✅ CORRETO — token simbólico do namespace, tipo garantido
@Inject(jwtConfig.KEY)
private readonly jwt: ConfigType<typeof jwtConfig>
// this.jwt.secret é string (nunca undefined — Zod garantiu)
```

### ❌ `ConfigModule` sem `isGlobal: true` + reimportação em todo módulo

```typescript
// ❌ ERRADO — ConfigModule reimportado em cada módulo
@Module({
  imports: [ConfigModule], // ← repetido em DatabaseModule, AuthModule, etc.
})
export class DatabaseModule {}

// ✅ CORRETO — registrado uma única vez com isGlobal: true
// app.module.ts:
ConfigModule.forRoot({ isGlobal: true, load: [...] })
```

### ❌ Variáveis sem validação chegando como `undefined`

```typescript
// ❌ ERRADO — sem validação, bugs silenciosos em produção
const port = parseInt(process.env.PORT); // NaN se PORT não existir

// ✅ CORRETO — Zod garante que PORT é número válido antes do bootstrap
// env.PORT é garantidamente number após a validação
```

### ❌ `synchronize: true` em banco de produção

```typescript
// ❌ CRÍTICO — destrói dados em produção
TypeOrmModule.forRoot({ synchronize: true }) // ← NUNCA em produção

// ✅ CORRETO — controle explícito via migrations
TypeOrmModule.forRoot({
  synchronize: false,
  migrationsRun: true, // ← roda migrations automaticamente no startup
})
```

---

## 10. Checklist de Validação

- [ ] `process.env` é acessado **apenas** em `src/config/`?
- [ ] `env.config.ts` valida **todas** as variáveis necessárias com Zod?
- [ ] A aplicação falha no bootstrap se alguma variável obrigatória estiver ausente?
- [ ] Cada grupo de configurações tem seu próprio arquivo com `registerAs`?
- [ ] `ConfigModule.forRoot()` está com `isGlobal: true` no `AppModule`?
- [ ] Todos os arquivos de namespace estão listados em `load: [...]`?
- [ ] `ConfigService.get()` com strings arbitrárias foi **substituído** por `@Inject(config.KEY)`?
- [ ] `@Inject(config.KEY)` usa `ConfigType<typeof config>` para tipagem?
- [ ] `.env.example` está commitado com todas as variáveis (sem valores sensíveis)?
- [ ] `.env`, `.env.*.local`, `.env.production`, `.env.staging` estão no `.gitignore`?
- [ ] `synchronize: false` no TypeORM para ambientes não-locais?

---

## Referências

- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Define `src/config/` como pasta centralizada e o alias `@config/*`
- [nest-modules-pattern.md](.agent/skills/backend/foundation-and-architecture/nest-modules-pattern.md) — Padrão `forRootAsync()` que consome configurações
- [NestJS Docs — Configuration](https://docs.nestjs.com/techniques/configuration)
- [NestJS Docs — Dynamic Modules](https://docs.nestjs.com/fundamentals/dynamic-modules)
- [Zod Docs](https://zod.dev)
