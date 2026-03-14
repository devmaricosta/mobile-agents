---
name: dependency-injection
description: Padrões avançados de DI no NestJS — custom providers com useFactory e useClass, injeção de tokens com InjectionToken, como testar módulos com providers mockados, e como evitar god modules. Esta skill complementa nest-modules-pattern.md (estrutura de módulos), config-management.md (tokens de configuração via registerAs) e clean-architecture.md (camadas e responsabilidades).
---

# Dependency Injection Avançado — NestJS

Você é um Arquiteto de Software Senior. Esta skill complementa `nest-modules-pattern.md` (organização de módulos), `config-management.md` (injeção de configuração via `registerAs` e `databaseConfig.KEY`) e `clean-architecture.md` (separação de camadas). O foco aqui é **como usar o sistema de DI do NestJS além do básico**: custom providers, tokens simbólicos, factories assíncronas, testes isolados e como evitar o god module.

> **Regra absoluta (herdada de `nest-modules-pattern.md`):** exports só deve conter o que é contrato público. Use o sistema de DI para inverter dependências, não para expor implementações internas.

---

## 1. Os Quatro Tipos de Provider

O NestJS suporta quatro formas de registrar um provider no array `providers` de um módulo:

| Forma | Quando usar |
|-------|-------------|
| **Classe simples** (`UsersService`) | 99% dos casos — DI clássico |
| **`useValue`** | Constantes, objetos literais, mocks em testes |
| **`useClass`** | Trocar implementação por ambiente ou condição |
| **`useFactory`** | Criação assíncrona, lógica condicional, dependências externas |

```typescript
// Exemplo dos quatro tipos em um módulo de infra
@Module({
  providers: [
    // 1. Classe simples (shorthand)
    UsersService,

    // 2. useValue — objeto literal
    {
      provide: APP_VERSION_TOKEN,
      useValue: '2.0.0',
    },

    // 3. useClass — troca por ambiente
    {
      provide: LoggerService,
      useClass: process.env.NODE_ENV === 'production'
        ? PinoLoggerService
        : ConsoleLoggerService,
    },

    // 4. useFactory — criação com dependências
    {
      provide: DATABASE_CONNECTION,
      useFactory: async (config: DatabaseConfig) => {
        return await createConnection(config);
      },
      inject: [DATABASE_CONFIG_TOKEN],
    },
  ],
})
export class InfraModule {}
```

---

## 2. Tokens de Injeção — Sem Strings Mágicas

### O problema com strings literais

```typescript
// ❌ ERRADO — string literal espalhada pelo código
providers: [{ provide: 'DATABASE_CONFIG', useValue: dbConfig }]
constructor(@Inject('DATABASE_CONFIG') config: any) {}
// Sem autocomplete, sem type-safety, fácil de errar
```

### ✅ Padrão correto: `InjectionToken` tipado

Crie um arquivo de constantes por domínio de infraestrutura:

```typescript
// src/infra/database/database.tokens.ts
import { InjectionToken } from '@nestjs/common';
import type { DatabaseConfig } from '@config/database.config';

export const DATABASE_CONFIG_TOKEN: InjectionToken<DatabaseConfig> =
  'DATABASE_CONFIG';

export const DATABASE_CONNECTION_TOKEN: InjectionToken =
  'DATABASE_CONNECTION';
```

```typescript
// src/infra/mail/mail.tokens.ts
import { InjectionToken } from '@nestjs/common';

export const MAIL_OPTIONS_TOKEN: InjectionToken = 'MAIL_OPTIONS';
export const MAIL_TRANSPORT_TOKEN: InjectionToken = 'MAIL_TRANSPORT';
```

> **Convenção de nomenclatura:** sempre `SCREAMING_SNAKE_CASE` + sufixo `_TOKEN`. Isso é consistente com a convenção de constantes globais definida em `nest-project-structure.md`.

### Uso do token no módulo e na classe

```typescript
// src/infra/mail/mail.module.ts
import { Module } from '@nestjs/common';
import { MAIL_OPTIONS_TOKEN, MAIL_TRANSPORT_TOKEN } from './mail.tokens';
import { MailService } from './mail.service';

@Module({
  providers: [
    {
      provide: MAIL_OPTIONS_TOKEN,
      useValue: {
        host: 'smtp.sendgrid.net',
        port: 587,
        from: 'no-reply@app.com',
      },
    },
    {
      provide: MAIL_TRANSPORT_TOKEN,
      useFactory: async (options: MailOptions) => {
        return nodemailer.createTransport(options);
      },
      inject: [MAIL_OPTIONS_TOKEN],
    },
    MailService,
  ],
  exports: [MailService], // ← exporta apenas o serviço, nunca o token interno
})
export class MailModule {}

// src/infra/mail/mail.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { MAIL_TRANSPORT_TOKEN } from './mail.tokens';
import type { Transporter } from 'nodemailer';

@Injectable()
export class MailService {
  constructor(
    @Inject(MAIL_TRANSPORT_TOKEN)
    private readonly transporter: Transporter,
  ) {}

  async send(to: string, subject: string, html: string): Promise<void> {
    await this.transporter.sendMail({ from: 'no-reply@app.com', to, subject, html });
  }
}
```

### Tokens de configuração (`registerAs`) — integração com `config-management.md`

Quando usar `registerAs` (definido em `config-management.md`), o `.KEY` gerado já é o token — não crie um token separado:

```typescript
// ✅ CORRETO — usa o token gerado pelo registerAs
import databaseConfig from '@config/database.config';

providers: [
  {
    provide: DATABASE_CONNECTION,
    useFactory: async (dbConfig: ConfigType<typeof databaseConfig>) => {
      return await createConnection(dbConfig);
    },
    inject: [databaseConfig.KEY], // ← token simbólico do registerAs
  },
]

// ❌ ERRADO — cria token duplicado para config que já tem .KEY
export const DB_CONFIG_TOKEN = 'database'; // ← desnecessário, use databaseConfig.KEY
```

---

## 3. `useFactory` — Factories Avançadas

### Factory síncrona simples

```typescript
{
  provide: HASH_ALGORITHM_TOKEN,
  useFactory: () => bcrypt, // ← retorna diretamente o módulo bcrypt
}
```

### Factory assíncrona com múltiplas dependências

```typescript
// src/infra/cache/cache.module.ts
import { CACHE_OPTIONS_TOKEN, CACHE_CLIENT_TOKEN } from './cache.tokens';
import cacheConfig from '@config/cache.config';

@Module({
  providers: [
    {
      provide: CACHE_CLIENT_TOKEN,
      useFactory: async (
        config: ConfigType<typeof cacheConfig>,
        logger: LoggerService,  // ← múltiplas dependências
      ) => {
        const client = new Redis({
          host: config.host,
          port: config.port,
          password: config.password,
        });

        client.on('connect', () => logger.log('Redis conectado'));
        client.on('error', (err) => logger.error('Redis erro', err));

        await client.ping(); // ← verificação assíncrona
        return client;
      },
      inject: [cacheConfig.KEY, LoggerService],
    },
    CacheService,
  ],
  exports: [CacheService],
})
export class CacheModule {}
```

### Factory com provider opcional

```typescript
// Provider opcional — não falha se não for registrado
{
  provide: METRICS_CLIENT_TOKEN,
  useFactory: (metricsConfig?: MetricsConfig) => {
    if (!metricsConfig) {
      return new NoopMetricsClient(); // ← fallback seguro
    }
    return new PrometheusClient(metricsConfig);
  },
  inject: [{ token: METRICS_CONFIG_TOKEN, optional: true }],
}
```

---

## 4. `useClass` — Inversão de Dependência Real

`useClass` é a ferramenta correta quando você quer **programar para uma interface** (ou classe abstrata) e trocar a implementação por contexto.

### Padrão com classe abstrata (recomendado)

```typescript
// src/modules/notifications/abstractions/notification-sender.abstract.ts
export abstract class NotificationSender {
  abstract send(userId: string, message: string): Promise<void>;
}

// src/infra/push/push-notification.service.ts
@Injectable()
export class PushNotificationService extends NotificationSender {
  async send(userId: string, message: string): Promise<void> {
    // implementação real com Firebase/APNS
  }
}

// src/infra/push/mock-notification.service.ts (apenas para testes)
@Injectable()
export class MockNotificationService extends NotificationSender {
  async send(userId: string, message: string): Promise<void> {
    console.log(`[MOCK] Notificação para ${userId}: ${message}`);
  }
}
```

```typescript
// src/modules/notifications/notifications.module.ts
import { NotificationSender } from './abstractions/notification-sender.abstract';
import { PushNotificationService } from '@infra/push/push-notification.service';
import { MockNotificationService } from '@infra/push/mock-notification.service';
import { env } from '@config/env.config';

@Module({
  providers: [
    {
      provide: NotificationSender,  // ← token é a classe abstrata
      useClass: env.NODE_ENV === 'production'
        ? PushNotificationService
        : MockNotificationService,
    },
  ],
  exports: [NotificationSender],
})
export class NotificationsModule {}
```

```typescript
// src/modules/meals/use-cases/share-meal/share-meal.use-case.ts
@Injectable()
export class ShareMealUseCase {
  constructor(
    private readonly mealsRepository: MealsRepository,
    private readonly notificationSender: NotificationSender, // ← depende da abstração
  ) {}

  async execute(input: ShareMealInput): Promise<void> {
    const meal = await this.mealsRepository.findById(input.mealId);
    await this.notificationSender.send(input.targetUserId, `Refeição compartilhada: ${meal.id}`);
  }
}
```

> **Integração com `clean-architecture.md`:** O `UseCase` nunca sabe qual implementação de `NotificationSender` está sendo usada. Isso respeita a regra de dependência — a camada de Application depende de abstrações, não de infraestrutura concreta.

---

## 5. Como Testar Módulos com Providers Mockados

### Padrão de teste com `Test.createTestingModule()`

Cada Use Case deve ser testado com mocks dos seus providers. Este padrão é consistente com o que está em `nest-project-structure.md` (seção 8 — Teste de Use Cases).

```typescript
// src/modules/meals/__tests__/share-meal.use-case.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { ShareMealUseCase } from '../use-cases/share-meal/share-meal.use-case';
import { MealsRepository } from '../repositories/meals.repository';
import { NotificationSender } from '@modules/notifications/abstractions/notification-sender.abstract';

describe('ShareMealUseCase', () => {
  let useCase: ShareMealUseCase;
  let mealsRepository: jest.Mocked<MealsRepository>;
  let notificationSender: jest.Mocked<NotificationSender>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ShareMealUseCase,
        // Mock do repository — useValue com jest.fn()
        {
          provide: MealsRepository,
          useValue: {
            findById: jest.fn(),
          },
        },
        // Mock da abstração — mesma técnica
        {
          provide: NotificationSender,
          useValue: {
            send: jest.fn(),
          },
        },
      ],
    }).compile();

    useCase = module.get(ShareMealUseCase);
    mealsRepository = module.get(MealsRepository);
    notificationSender = module.get(NotificationSender);
  });

  it('deve compartilhar a refeição e notificar o usuário', async () => {
    const mockMeal = { id: 'meal-1', userId: 'user-1', blocks: 4 };
    mealsRepository.findById.mockResolvedValue(mockMeal as any);
    notificationSender.send.mockResolvedValue();

    await useCase.execute({ mealId: 'meal-1', targetUserId: 'user-2' });

    expect(mealsRepository.findById).toHaveBeenCalledWith('meal-1');
    expect(notificationSender.send).toHaveBeenCalledWith(
      'user-2',
      expect.stringContaining('meal-1'),
    );
  });
});
```

### Mockando tokens de injeção personalizados

```typescript
// Quando o provider usa um InjectionToken, use o mesmo token no mock
import { MAIL_TRANSPORT_TOKEN } from '@infra/mail/mail.tokens';

const module = await Test.createTestingModule({
  providers: [
    MailService,
    {
      provide: MAIL_TRANSPORT_TOKEN, // ← mesmo token do provider real
      useValue: {
        sendMail: jest.fn().mockResolvedValue({ messageId: 'mock-id' }),
      },
    },
  ],
}).compile();
```

### Sobrescrevendo providers com `.overrideProvider()`

Para testes de integração onde você quer usar o módulo real mas substituir apenas um provider:

```typescript
// src/modules/auth/__tests__/auth.controller.spec.ts
import { Test } from '@nestjs/testing';
import { AuthModule } from '../auth.module';
import { MailService } from '@infra/mail/mail.service';

describe('AuthController (integration)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AuthModule], // ← importa o módulo REAL
    })
      .overrideProvider(MailService)   // ← substitui apenas o MailService
      .useValue({ send: jest.fn() })
      .compile();

    app = module.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });
});
```

### Padrão de factory de módulo de teste (evitar repetição)

Para módulos testados em múltiplos specs, crie um helper:

```typescript
// src/modules/meals/__tests__/meals-test.module.ts
import { Test, TestingModule } from '@nestjs/testing';
import { MealsRepository } from '../repositories/meals.repository';

export const mockMealsRepository = {
  create: jest.fn(),
  findByUserId: jest.fn(),
  findById: jest.fn(),
};

export async function createMealsTestModule(
  providers: any[] = [],
): Promise<TestingModule> {
  return Test.createTestingModule({
    providers: [
      {
        provide: MealsRepository,
        useValue: mockMealsRepository,
      },
      ...providers,
    ],
  }).compile();
}
```

```typescript
// uso em specs diferentes
describe('CreateMealUseCase', () => {
  beforeEach(async () => {
    const module = await createMealsTestModule([CreateMealUseCase]);
    // ...
  });
});
```

---

## 6. Como Evitar God Modules

Um **god module** é um módulo que registra providers demais, mistura responsabilidades e vira um ponto central de acoplamento. É o anti-padrão mais comum em projetos NestJS que crescem sem disciplina.

### Sintomas de um god module

```typescript
// ❌ God Module — tudo em um lugar
@Module({
  providers: [
    UsersService,
    AuthService,
    MealsService,
    NotificationsService,
    MailService,
    HashService,
    JwtService,
    CacheService,
    // ... 20 outros providers ...
  ],
  exports: [
    UsersService,
    AuthService,
    MealsService,
    // ... tudo exportado ...
  ],
})
export class AppModule {}     // ← NUNCA faça isso
```

### ✅ Solução: decomposição por responsabilidade

| Tipo de módulo | Responsabilidade | Exemplos |
|----------------|-----------------|---------|
| **Módulo de domínio** | Um único bounded context | `UsersModule`, `MealsModule`, `AuthModule` |
| **Módulo de infraestrutura** | Uma conexão/serviço externo | `DatabaseModule`, `CacheModule`, `MailModule` |
| **SharedModule** | Utilitários genuinamente transversais | `HashService`, `LoggerService`, paginação |
| **CoreModule** | Bootstrap de infraestrutura | importa `DatabaseModule`, `CacheModule`, `MailModule` |

```typescript
// src/infra/core/core.module.ts — agrupa infra sem ser um god module
@Module({
  imports: [
    DatabaseModule,   // ← cada um tem sua própria responsabilidade
    CacheModule,
    MailModule,
  ],
  // exports vazios — módulos de domínio importam diretamente o que precisam
})
export class CoreModule {}

// src/app.module.ts — limpo e declarativo
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [appConfig, databaseConfig, jwtConfig] }),
    CoreModule,       // ← toda a infra
    SharedModule,     // ← utilitários compartilhados
    AuthModule,       // ← domínio de auth
    UsersModule,      // ← domínio de usuários
    MealsModule,      // ← domínio de refeições
  ],
})
export class AppModule {}
```

### Regras para prevenir god modules

**1. Regra do tamanho:** Se um módulo tem mais de 8-10 providers, questione se ele está focado.

**2. Regra de coesão:** Pergunte: "Todos os providers neste módulo trabalham juntos para um propósito único?" Se não, separe.

**3. Regra de exports:** Um módulo que exporta mais de 3-4 itens provavelmente está fazendo coisas demais.

**4. Regra do `@Global()`:** Conforme definido em `nest-modules-pattern.md`, use `@Global()` **apenas** para infraestrutura transversal (`ConfigModule`, `LoggerModule`). Nunca para módulos de domínio.

```typescript
// ❌ ERRADO — exportar tudo "por precaução"
@Module({
  providers: [UserService, AuthHelper, TokenService, SessionService],
  exports: [UserService, AuthHelper, TokenService, SessionService], // ← tudo exposto
})
export class UsersModule {}

// ✅ CORRETO — exportar apenas o contrato público
@Module({
  providers: [
    LoginUseCase,         // providers registrados
    RegisterUseCase,
    AuthRepository,
    TokenService,         // ← interno ao módulo
  ],
  exports: [
    LoginUseCase,         // ← apenas o que outros módulos precisam
    RegisterUseCase,
  ],
})
export class AuthModule {}
```

---

## 7. Providers Assíncronos e Ciclo de Vida

### `onModuleInit` e `onApplicationBootstrap`

Para providers que precisam de inicialização assíncrona após a injeção estar completa:

```typescript
import { Injectable, OnModuleInit, OnApplicationBootstrap } from '@nestjs/common';

@Injectable()
export class DatabaseService implements OnModuleInit {
  private connection: Connection;

  constructor(
    @Inject(DATABASE_CLIENT_TOKEN)
    private readonly client: DatabaseClient,
  ) {}

  // Executado após o módulo ser inicializado e DI estar completo
  async onModuleInit(): Promise<void> {
    this.connection = await this.client.connect();
  }
}
```

> **Quando usar `useFactory` vs `onModuleInit`:** Use `useFactory` quando a inicialização é necessária para criar o próprio objeto do provider. Use `onModuleInit` quando o objeto já existe mas precisa de setup após ser criado.

### Cleanup com `onModuleDestroy`

```typescript
@Injectable()
export class RedisService implements OnModuleDestroy {
  constructor(
    @Inject(CACHE_CLIENT_TOKEN)
    private readonly redis: Redis,
  ) {}

  async onModuleDestroy(): Promise<void> {
    await this.redis.quit(); // ← libera conexão ao fechar a app
  }
}
```

---

## 8. Checklist de DI Avançado

### Tokens e Providers
- [ ] Strings literais em `provide:` foram substituídas por constantes `InjectionToken` em arquivos `*.tokens.ts`?
- [ ] Tokens de `registerAs` usam `.KEY` em vez de criar constante duplicada?
- [ ] Tokens nomeados com `SCREAMING_SNAKE_CASE` + sufixo `_TOKEN`?
- [ ] `@Inject(TOKEN)` tem o tipo TypeScript correto no parâmetro?

### `useFactory`
- [ ] Factories assíncronas retornam `Promise<T>`?
- [ ] Dependências opcionais usam `{ token: TOKEN, optional: true }` no `inject`?
- [ ] Factory tem fallback para quando provider opcional não estiver disponível?

### Testes
- [ ] Use cases testados com `Test.createTestingModule()` + `useValue` para dependências?
- [ ] Tokens de injeção mockados usando o mesmo token do provider real?
- [ ] `.overrideProvider()` usado em testes de integração (não substituir em unit tests)?
- [ ] Helper de módulo de teste criado para evitar repetição de mocks entre specs?

### Evitar God Modules
- [ ] Módulo tem menos de 10 providers?
- [ ] Todos os providers do módulo pertencem ao mesmo bounded context?
- [ ] `exports` contém apenas o contrato público (3-4 itens máximo)?
- [ ] `@Global()` está sendo usado apenas para infraestrutura transversal?
- [ ] `AppModule` é declarativo (lista de módulos) e não contém lógica?

---

## Referências

- [nest-modules-pattern.md](.agent/skills/backend/foundation-and-architecture/nest-modules-pattern.md) — `@Global()`, `DynamicModule`, `forRootAsync()`, dependências circulares
- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — `registerAs`, `.KEY` como token simbólico, `ConfigType<typeof config>`
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Dependências fluem para dentro, UseCase depende de abstrações
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — `src/infra/`, path aliases, convenção de nomenclatura, testes de use cases
- [NestJS Docs — Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
- [NestJS Docs — Injection Scopes](https://docs.nestjs.com/fundamentals/injection-scopes)
- [NestJS Docs — Testing](https://docs.nestjs.com/fundamentals/testing)
