---
name: NestJS Modules Pattern
description: Como estruturar módulos NestJS corretamente — quando criar um módulo, como evitar dependências circulares, uso de @Global() e DynamicModule para configurações, e o padrão de exportar apenas o contrato público do módulo.
---

# NestJS Modules Pattern

## Visão Geral

Módulos são a **unidade fundamental de organização** em NestJS. Cada módulo encapsula um domínio funcional completo e é responsável por declarar, fornecer e exportar apenas o que é contrato público para os demais módulos.

---

## 1. Quando Criar um Módulo

Crie um módulo quando o conjunto de funcionalidades:

| Critério | Exemplo |
|----------|---------|
| Representa um domínio de negócio coeso | `UsersModule`, `AuthModule`, `NotificationsModule` |
| Possui seus próprios providers/controllers | `ProductsModule` com `ProductsService` + `ProductsController` |
| Precisa ser reutilizado por outros módulos | `SharedModule`, `DatabaseModule` |
| Possui configuração própria (conexão, chave de API) | `MailModule`, `StorageModule` |

> **Regra de ouro**: Se você precisa importar um conjunto de providers em mais de um lugar, isso é um sinal de que precisa de um módulo.

### ❌ Anti-padrões — quando NÃO criar um módulo separado

```
# Ruim: criar módulo para cada arquivo/classe
UserProfileModule  ← desnecessário se já existe UsersModule
UserAddressModule  ← desnecessário se é simples sub-recurso
```

---

## 2. Anatomia de um Módulo Correto

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserRepository } from './users.repository';
import { User } from './entities/user.entity';

// providers internos que NÃO devem ser expostos
import { UserPasswordHasher } from './internals/user-password-hasher';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [
    UsersService,
    UserRepository,
    UserPasswordHasher, // ← interno, não exportado
  ],
  exports: [
    UsersService, // ← apenas o contrato público
  ],
})
export class UsersModule {}
```

### Regras para `exports`

- Exporte **apenas o que outros módulos precisam consumir**
- Providers não exportados são privados do módulo (encapsulamento)
- Nunca exporte implementações internas (hashers, mappers internos, helpers)

```typescript
// ✅ Correto: exporta apenas a interface pública
exports: [UsersService]

// ❌ Errado: expõe detalhes de implementação
exports: [UsersService, UserPasswordHasher, UserRepository]
```

---

## 3. SharedModule — Utilitários Comuns

Use `SharedModule` para providers **genuinamente compartilhados** sem domínio próprio:

```typescript
// shared/shared.module.ts
import { Module } from '@nestjs/common';

import { HashService } from './services/hash.service';
import { LoggerService } from './services/logger.service';
import { PaginationHelper } from './helpers/pagination.helper';

@Module({
  providers: [HashService, LoggerService, PaginationHelper],
  exports: [HashService, LoggerService, PaginationHelper],
})
export class SharedModule {}
```

Candidatos para `SharedModule`:
- Serviços de hash / criptografia
- Logger customizado
- Helpers de paginação / serialização
- Pipes, Guards, Interceptors reutilizáveis

---

## 4. @Global() — Módulos Globais

O decorator `@Global()` torna os providers exportados **disponíveis em toda a aplicação** sem precisar importar o módulo explicitamente.

```typescript
// config/config.module.ts
import { Global, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Global() // ← todos os módulos terão acesso ao ConfigService
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}
```

### Quando usar `@Global()`

| Use `@Global()` | NÃO use `@Global()` |
|-----------------|---------------------|
| `ConfigModule` (configurações da app) | Módulos de feature negócio |
| `LoggerModule` (logger global) | `UsersModule`, `AuthModule` |
| `DatabaseModule` (conexão base) | Qualquer módulo com domínio específico |
| Providers usados em **todos** os módulos | Providers usados em apenas alguns módulos |

> ⚠️ **Cuidado**: o abuso de `@Global()` torna as dependências invisíveis e dificulta testes e manutenção. Prefira importar explicitamente quando possível.

### Registrar no AppModule apenas uma vez

```typescript
// app.module.ts
@Module({
  imports: [
    ConfigModule, // ← registrado uma vez, disponível em toda a app
    UsersModule,
    AuthModule,
  ],
})
export class AppModule {}
```

---

## 5. DynamicModule — Configuração em Tempo de Execução

Use `DynamicModule` para módulos que precisam receber **configuração externa** (conexão de banco, chave de API, URL de serviço).

### Padrão `forRoot()` / `forRootAsync()`

```typescript
// mail/mail.module.ts
import { DynamicModule, Module } from '@nestjs/common';
import { MailService } from './mail.service';
import { MAIL_OPTIONS } from './mail.constants';

export interface MailModuleOptions {
  host: string;
  port: number;
  from: string;
}

@Module({})
export class MailModule {
  // Configuração síncrona
  static forRoot(options: MailModuleOptions): DynamicModule {
    return {
      module: MailModule,
      providers: [
        {
          provide: MAIL_OPTIONS,
          useValue: options,
        },
        MailService,
      ],
      exports: [MailService],
      global: false, // deixar false por padrão, ser explícito
    };
  }

  // Configuração assíncrona (lê de ConfigService, por exemplo)
  static forRootAsync(options: {
    useFactory: (...args: any[]) => MailModuleOptions | Promise<MailModuleOptions>;
    inject?: any[];
    imports?: any[];
  }): DynamicModule {
    return {
      module: MailModule,
      imports: options.imports ?? [],
      providers: [
        {
          provide: MAIL_OPTIONS,
          useFactory: options.useFactory,
          inject: options.inject ?? [],
        },
        MailService,
      ],
      exports: [MailService],
    };
  }
}
```

### Uso no AppModule

```typescript
// app.module.ts — Síncrono
MailModule.forRoot({
  host: 'smtp.example.com',
  port: 587,
  from: 'no-reply@example.com',
})

// app.module.ts — Assíncrono com ConfigService
MailModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService): MailModuleOptions => ({
    host: config.get('MAIL_HOST'),
    port: config.get<number>('MAIL_PORT'),
    from: config.get('MAIL_FROM'),
  }),
})
```

### `forFeature()` — configuração por módulo filho

Use `forFeature()` quando o módulo precisa de registro específico por feature (padrão TypeORM, Mongoose):

```typescript
// users/users.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([User, UserProfile]), // ← apenas entidades deste módulo
  ],
})
export class UsersModule {}
```

### ConfigurableModuleBuilder (NestJS 9+)

Para módulos dinâmicos mais complexos, use o builder oficial:

```typescript
// cache/cache.module-definition.ts
import { ConfigurableModuleBuilder } from '@nestjs/common';

export interface CacheModuleOptions {
  ttl: number;
  maxItems: number;
}

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<CacheModuleOptions>()
    .setClassMethodName('forRoot') // gera forRoot() e forRootAsync()
    .build();
```

```typescript
// cache/cache.module.ts
@Module({
  providers: [CacheService],
  exports: [CacheService],
})
export class CacheModule extends ConfigurableModuleClass {}
```

---

## 6. Dependências Circulares — Como Evitar e Resolver

### Por que são um problema

Dependências circulares ocorrem quando `ModuleA` importa `ModuleB` e `ModuleB` importa `ModuleA`. Isso cria um loop no grafo de inicialização do NestJS:

```
Nest cannot create the <module> instance.
The module at index [1] of the UsersModule "imports" array is undefined.
```

### Estratégia 1 (preferida): Extrair para módulo intermediário

```typescript
// ❌ Problema: UsersModule ↔ AuthModule
@Module({ imports: [AuthModule] })  // UsersModule precisa verificar roles
export class UsersModule {}

@Module({ imports: [UsersModule] }) // AuthModule precisa buscar usuários
export class AuthModule {}

// ✅ Solução: extrair para módulo compartilhado
@Module({
  providers: [UserFinderService], // apenas o que AuthModule precisa
  exports: [UserFinderService],
})
export class UserCoreModule {} // sem dependência de AuthModule

@Module({ imports: [UserCoreModule] })
export class AuthModule {}

@Module({ imports: [AuthModule, UserCoreModule] })
export class UsersModule {}
```

### Estratégia 2 (último recurso): `forwardRef()`

Use apenas quando **não for possível** refatorar a estrutura:

```typescript
// users/users.module.ts
import { forwardRef } from '@nestjs/common';

@Module({
  imports: [forwardRef(() => AuthModule)], // ← referência postergada
})
export class UsersModule {}

// auth/auth.module.ts
@Module({
  imports: [forwardRef(() => UsersModule)],
})
export class AuthModule {}
```

> ⚠️ `forwardRef()` é um **paliativo**, não uma solução arquitetural. A presença de `forwardRef()` é um indicador de que o design de módulos precisa ser revisado.

### Detectar dependências circulares

```bash
# Ferramenta para visualizar o grafo de dependências
npx madge --circular --extensions ts ./src
```

### Hierarquia saudável de módulos

```
AppModule
├── CoreModule (infraestrutura: DB, Config, Logger)
├── SharedModule (utilitários compartilhados)
├── AuthModule (autenticação)
│   └── imports: [SharedModule, UserCoreModule]
└── UsersModule (domínio de usuários)
    └── imports: [SharedModule, UserCoreModule]
```

**Fluxo de dependência sempre deve ser unidirecional (de cima para baixo).**

---

## 7. Checklist de Revisão de Módulo

Antes de criar ou modificar um módulo, valide:

- [ ] O módulo representa um domínio de negócio coeso?
- [ ] `providers` internos estão fora do `exports`?
- [ ] `exports` contém apenas o contrato público do módulo?
- [ ] Não há dependência circular entre módulos?
- [ ] `@Global()` está sendo usado apenas para infraestrutura transversal?
- [ ] Módulos dinâmicos implementam `forRoot()` / `forRootAsync()` quando necessário?
- [ ] A configuração assíncrona usa `forRootAsync()` com `inject` correto?

---

## 8. Estrutura de Pastas Alinhada ao Módulo

```
src/
└── modules/
    └── users/
        ├── users.module.ts          ← declaração do módulo
        ├── users.controller.ts      ← rotas HTTP
        ├── users.service.ts         ← lógica de negócio (exportado)
        ├── users.repository.ts      ← acesso a dados (interno)
        ├── dto/
        │   ├── create-user.dto.ts
        │   └── update-user.dto.ts
        ├── entities/
        │   └── user.entity.ts
        └── internals/               ← detalhes de implementação (não exportamos)
            └── user-password-hasher.ts
```

> Esta estrutura deve ser consistente com `nest-project-structure.md`. Para detalhes sobre estrutura de pastas e camadas, consulte aquela skill.

---

## Referências

- [NestJS Docs — Modules](https://docs.nestjs.com/modules)
- [NestJS Docs — Dynamic Modules](https://docs.nestjs.com/fundamentals/dynamic-modules)
- [NestJS Docs — Circular Dependency](https://docs.nestjs.com/fundamentals/circular-dependency)
- [NestJS Docs — ConfigurableModuleBuilder](https://docs.nestjs.com/fundamentals/dynamic-modules#configurable-module-builder)
