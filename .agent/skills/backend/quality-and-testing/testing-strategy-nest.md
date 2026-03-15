---
name: testing-strategy-nest
description: Estratégia completa de testes para projetos NestJS — pirâmide de testes com unitários de Use Cases (mock de Repository), testes de integração de Controllers com Supertest, testes E2E com TestContainers ou SQLite em memória, cobertura mínima por camada, estrutura de arquivos co-localizados e configuração de Jest por ambiente. Complementa clean-architecture.md (camadas e responsabilidades), nest-project-structure.md (estrutura de módulos e Use Cases Pattern) e dto-validation.md (DTOs e ValidationPipe).
---

# Estratégia de Testes — NestJS

Você é um Engenheiro de Software Senior especialista em qualidade de software para NestJS. Esta skill define **como testar cada camada da aplicação** de forma consistente com a arquitetura definida nas demais skills do projeto.

> **Regra fundamental:** Teste o comportamento, não a implementação. Use mocks cirurgicamente — apenas onde precisar isolar dependências externas reais (banco de dados, HTTP externo, filas). Quanto mais próximo da produção, maior a confiança; quanto mais rápido o ciclo de feedback, maior a produtividade.

---

## 1. Pirâmide de Testes

```
              /\
             /E2E\          ← Poucos (TestContainers / SQLite in-memory)
            /------\
           / Integr.\       ← Moderados (Supertest + módulo real + banco in-memory)
          /----------\
         /  Unitários  \    ← Muitos (Jest + mocks de Repository/UseCase)
        /--------------\
```

| Camada | Ferramenta | O que testa | Velocidade |
|---|---|---|---|
| **Unitário** | Jest + `Test.createTestingModule` | Use Cases, Guards, Pipes, Helpers | ⚡ Ultra-rápido |
| **Integração** | Jest + Supertest + SQLite/:memory: | Controllers → UseCases → Repository → DB | 🔄 Moderado |
| **E2E** | Jest + Supertest + TestContainers/PostgreSQL real | Fluxos completos com banco real | 🐢 Mais lento |

### Distribuição recomendada por esforço

| Camada | % do esforço total | Nº aproximado de testes |
|---|---|---|
| Unitário | **70%** | Muitos — cada método relevante do UseCase |
| Integração | **20%** | Um por endpoint crítico por módulo |
| E2E | **10%** | Apenas fluxos de negócio mais críticos |

---

## 2. Estrutura de Arquivos — Co-localização

Seguindo o padrão definido em `nest-project-structure.md`, cada módulo de domínio tem uma pasta `__tests__/` co-localizada:

```
src/
├── modules/
│   └── meals/
│       ├── controllers/
│       │   └── meals.controller.ts
│       ├── use-cases/
│       │   ├── create-meal/
│       │   │   ├── create-meal.use-case.ts
│       │   │   └── create-meal.use-case.spec.ts    ← UNITÁRIO (co-localizado)
│       │   └── get-meal-history/
│       │       ├── get-meal-history.use-case.ts
│       │       └── get-meal-history.use-case.spec.ts
│       ├── repositories/
│       │   └── meals.repository.ts
│       ├── entities/
│       │   └── meal.entity.ts
│       ├── dtos/
│       │   └── create-meal.dto.ts
│       └── __tests__/
│           ├── meals.controller.spec.ts            ← INTEGRAÇÃO (módulo)
│           └── meals.e2e-spec.ts                   ← E2E (módulo, opcional)
│
└── test/                                            ← E2E globais
    ├── jest-e2e.json
    ├── setup-e2e.ts                                 ← Bootstrap do TestContainers
    └── app.e2e-spec.ts                              ← E2E da aplicação inteira
```

### Convenções de nomenclatura

| Arquivo | Sufixo | Localização |
|---|---|---|
| Teste unitário de Use Case | `.use-case.spec.ts` | Co-localizado ao lado do arquivo |
| Teste de integração do Controller | `.controller.spec.ts` | `__tests__/` do módulo |
| Teste E2E do módulo | `.e2e-spec.ts` | `__tests__/` ou `test/` raiz |
| Setup E2E global | `setup-e2e.ts` | `test/` raiz |

---

## 3. Configuração do Jest

### 3.1 `jest.config.ts` — Testes Unitários e de Integração

```typescript
// jest.config.ts (raiz do projeto)
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  rootDir: '.',
  testRegex: '.*\\.spec\\.ts$',             // ← captura *.spec.ts (unitário + integração)
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  moduleNameMapper: {
    // Path aliases — espelha o tsconfig.json (skill nest-project-structure.md)
    '^@modules/(.*)$': '<rootDir>/src/modules/$1',
    '^@common/(.*)$': '<rootDir>/src/common/$1',
    '^@config/(.*)$': '<rootDir>/src/config/$1',
    '^@infra/(.*)$': '<rootDir>/src/infra/$1',
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.module.ts',            // módulos NestJS — sem lógica testável
    '!src/**/main.ts',                // bootstrap da aplicação
    '!src/**/*.dto.ts',               // DTOs — validados pelos seus decorators
    '!src/**/*.entity.ts',            // Entities — mapeamento de banco
    '!src/**/*.view-model.ts',        // ViewModels — mapeamento puro
    '!src/**/*.d.ts',                 // declarações de tipo
  ],
  coverageThreshold: {
    global: {
      branches:   70,
      functions:  75,
      lines:      80,
      statements: 80,
    },
    // Use Cases — cobertura alta obrigatória (coração da lógica de negócio)
    './src/**/use-cases/**/*.ts': {
      branches:   80,
      functions:  85,
      lines:      85,
      statements: 85,
    },
    // Repositories — cobertura moderada (lógica simples de banco)
    './src/**/repositories/**/*.ts': {
      branches:   60,
      functions:  70,
      lines:      70,
      statements: 70,
    },
  },
  coverageDirectory: './coverage',
};

export default config;
```

### 3.2 `test/jest-e2e.json` — Testes E2E

```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": "..",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  },
  "moduleNameMapper": {
    "^@modules/(.*)$": "<rootDir>/src/modules/$1",
    "^@common/(.*)$": "<rootDir>/src/common/$1",
    "^@config/(.*)$": "<rootDir>/src/config/$1",
    "^@infra/(.*)$": "<rootDir>/src/infra/$1"
  },
  "globalSetup": "./test/setup-e2e.ts",
  "testTimeout": 60000
}
```

### 3.3 Scripts no `package.json`

```json
{
  "scripts": {
    "test":         "jest",
    "test:watch":   "jest --watch",
    "test:cov":     "jest --coverage",
    "test:debug":   "node --inspect-brk -r tsconfig-paths/register -r ts-node/register node_modules/.bin/jest --runInBand",
    "test:e2e":     "jest --config ./test/jest-e2e.json --runInBand",
    "test:e2e:cov": "jest --config ./test/jest-e2e.json --runInBand --coverage",
    "test:ci":      "jest --ci --coverage --maxWorkers=2 --forceExit"
  }
}
```

> **`--runInBand` no E2E:** Obrigatório para E2E — evita concorrência entre testes que compartilham o mesmo banco de dados ou container.

---

## 4. Testes Unitários — Use Cases

### 4.1 Padrão Obrigatório

Use Cases são a camada mais importante a testar. Como cada Use Case tem `execute()` como único método público e depende apenas de Repositories (via DI), os testes são diretos.

**Regra:** Nunca injete `DataSource`, `TypeORM Repository<Entity>` ou `PrismaClient` diretamente no Use Case durante o teste — injete **o Repository customizado** como mock.

```typescript
// src/modules/meals/use-cases/create-meal/create-meal.use-case.spec.ts

import { Test, TestingModule } from '@nestjs/testing';
import { ConflictException, UnprocessableEntityException } from '@nestjs/common';

import { CreateMealUseCase } from './create-meal.use-case';
import { MealsRepository } from '@modules/meals/repositories/meals.repository';
import type { MealEntity } from '@modules/meals/entities/meal.entity';

// ─── Factory de dados de teste ────────────────────────────────────────────────

function makeMockMeal(overrides: Partial<MealEntity> = {}): MealEntity {
  return {
    id: 'meal-uuid-1',
    userId: 'user-uuid-1',
    blocks: 4,
    mode: 'quick',
    targetCalories: null,
    createdAt: new Date('2024-01-15T10:00:00.000Z'),
    ...overrides,
  } as MealEntity;
}

// ─── Testes ───────────────────────────────────────────────────────────────────

describe('CreateMealUseCase', () => {
  let useCase: CreateMealUseCase;
  // Tipo Mocked garante autocompletar e type-safety no mock
  let mealsRepository: jest.Mocked<MealsRepository>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CreateMealUseCase,
        {
          // ← Mock do Repository customizado — nunca do TypeORM Repository<Entity>
          provide: MealsRepository,
          useValue: {
            create:      jest.fn(),
            findById:    jest.fn(),
            findByUserId: jest.fn(),
          },
        },
      ],
    }).compile();

    useCase = module.get(CreateMealUseCase);
    mealsRepository = module.get(MealsRepository);
  });

  afterEach(() => {
    jest.clearAllMocks(); // Limpa mocks entre testes para evitar vazamento de estado
  });

  // ─── Caso de sucesso ─────────────────────────────────────────────────────────

  describe('execute (sucesso)', () => {
    it('deve criar uma refeição com dados válidos', async () => {
      const input = { userId: 'user-uuid-1', blocks: 4, mode: 'quick' as const };
      const mockMeal = makeMockMeal(input);
      mealsRepository.create.mockResolvedValue(mockMeal);

      const result = await useCase.execute(input);

      // ✅ Verifica que o repository foi chamado com os dados corretos
      expect(mealsRepository.create).toHaveBeenCalledWith(
        expect.objectContaining({ userId: 'user-uuid-1', blocks: 4 }),
      );
      // ✅ Verifica o resultado retornado
      expect(result.id).toBe('meal-uuid-1');
      expect(result.blocks).toBe(4);
    });
  });

  // ─── Casos de erro ────────────────────────────────────────────────────────────

  describe('execute (erros de negócio)', () => {
    it('deve lançar UnprocessableEntityException quando blocks <= 0', async () => {
      const input = { userId: 'user-uuid-1', blocks: 0, mode: 'quick' as const };

      await expect(useCase.execute(input)).rejects.toThrow(UnprocessableEntityException);
      expect(mealsRepository.create).not.toHaveBeenCalled();
    });

    it('deve lançar UnprocessableEntityException quando blocks > 20', async () => {
      const input = { userId: 'user-uuid-1', blocks: 21, mode: 'quick' as const };

      await expect(useCase.execute(input)).rejects.toThrow(UnprocessableEntityException);
    });
  });

  // ─── Edge cases ───────────────────────────────────────────────────────────────

  describe('execute (erros de infraestrutura)', () => {
    it('deve propagar erro do repository sem modificar', async () => {
      const input = { userId: 'user-uuid-1', blocks: 4, mode: 'quick' as const };
      const dbError = new Error('Connection refused');
      mealsRepository.create.mockRejectedValue(dbError);

      await expect(useCase.execute(input)).rejects.toThrow('Connection refused');
    });
  });
});
```

### 4.2 Testando Guards

```typescript
// src/common/guards/__tests__/jwt-auth.guard.spec.ts

import { ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { JwtAuthGuard } from '../jwt-auth.guard';

describe('JwtAuthGuard', () => {
  let guard: JwtAuthGuard;

  beforeEach(() => {
    guard = new JwtAuthGuard(); // Guards simples podem ser instanciados diretamente
  });

  it('deve lançar UnauthorizedException quando o token está ausente', () => {
    const context = {
      switchToHttp: () => ({
        getRequest: () => ({ headers: {} }), // ← sem Authorization header
      }),
    } as unknown as ExecutionContext;

    expect(() => guard.canActivate(context)).toThrow(UnauthorizedException);
  });
});
```

### 4.3 Testando Pipes customizados

```typescript
// src/common/pipes/__tests__/parse-blocks.pipe.spec.ts

import { BadRequestException } from '@nestjs/common';
import { ParseBlocksPipe } from '../parse-blocks.pipe';

describe('ParseBlocksPipe', () => {
  let pipe: ParseBlocksPipe;

  beforeEach(() => {
    pipe = new ParseBlocksPipe();
  });

  it('deve converter string numérica para number', () => {
    expect(pipe.transform('4')).toBe(4);
  });

  it('deve lançar BadRequestException para valor não numérico', () => {
    expect(() => pipe.transform('abc')).toThrow(BadRequestException);
  });
});
```

### 4.4 Helper de módulo de teste (evitar repetição)

Para módulos com muitos testes, extraia um helper:

```typescript
// src/modules/meals/__tests__/meals-test.helpers.ts

import { Test, TestingModule } from '@nestjs/testing';
import { MealsRepository } from '../repositories/meals.repository';
import type { MealEntity } from '../entities/meal.entity';

export const mockMealsRepository = {
  create:       jest.fn(),
  findById:     jest.fn(),
  findByUserId: jest.fn(),
  deleteById:   jest.fn(),
};

/**
 * Cria um módulo de teste pré-configurado com o MealsRepository mockado.
 * Passe os Use Cases adicionais que você quer testar.
 */
export async function createMealsTestModule(
  additionalProviders: any[] = [],
): Promise<TestingModule> {
  return Test.createTestingModule({
    providers: [
      {
        provide: MealsRepository,
        useValue: mockMealsRepository,
      },
      ...additionalProviders,
    ],
  }).compile();
}

export function makeMockMeal(overrides: Partial<MealEntity> = {}): MealEntity {
  return {
    id: `meal-${Math.random().toString(36).slice(2)}`,
    userId: 'user-uuid-1',
    blocks: 4,
    mode: 'quick',
    targetCalories: null,
    createdAt: new Date(),
    ...overrides,
  } as MealEntity;
}

// Limpa todos os mocks antes de cada teste
beforeEach(() => {
  Object.values(mockMealsRepository).forEach((fn) => fn.mockReset());
});
```

---

## 5. Testes de Integração — Controllers com Supertest

Testes de integração verificam o comportamento da **rota HTTP completa** — validação de DTO, guard, delegate ao UseCase e envelope de resposta — usando um banco SQLite em memória para não depender de Docker.

### 5.1 Padrão com SQLite `:memory:` (TypeORM)

```typescript
// src/modules/meals/__tests__/meals.controller.spec.ts

import { INestApplication, ValidationPipe } from '@nestjs/common';
import { Test, TestingModule } from '@nestjs/testing';
import { TypeOrmModule } from '@nestjs/typeorm';
import * as request from 'supertest';

import { MealsModule } from '../meals.module';
import { MealEntity } from '../entities/meal.entity';
import { TransformInterceptor } from '@common/interceptors/transform.interceptor';
import { HttpExceptionFilter } from '@common/filters/http-exception.filter';

describe('MealsController (integração)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [
        // ← SQLite em memória — sem Docker, sem PostgreSQL
        TypeOrmModule.forRoot({
          type: 'sqlite',
          database: ':memory:',
          entities: [MealEntity],
          synchronize: true,   // ← aceitável em testes — nunca em produção
          dropSchema: true,    // ← garante schema limpo a cada run
          logging: false,
        }),
        MealsModule,
      ],
    }).compile();

    app = module.createNestApplication();

    // Reproduz exatamente a configuração do main.ts
    app.useGlobalPipes(
      new ValidationPipe({
        whitelist: true,
        forbidNonWhitelisted: true,
        transform: true,
        transformOptions: { enableImplicitConversion: true },
      }),
    );
    app.useGlobalInterceptors(new TransformInterceptor());
    app.useGlobalFilters(new HttpExceptionFilter());
    app.setGlobalPrefix('api');

    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  // ─── POST /api/v1/meals ──────────────────────────────────────────────────────

  describe('POST /api/v1/meals', () => {
    it('deve criar uma refeição e retornar 201 com envelope { data }', async () => {
      return request(app.getHttpServer())
        .post('/api/v1/meals')
        .set('Authorization', 'Bearer valid-test-token') // ← mock do JWT
        .send({ blocks: 4, mode: 'quick' })
        .expect(201)
        .expect((res) => {
          expect(res.body.data).toBeDefined();
          expect(res.body.data.blocks).toBe(4);
          expect(res.body.data.id).toBeDefined();
        });
    });

    it('deve retornar 400 quando o payload é inválido (blocks ausente)', async () => {
      return request(app.getHttpServer())
        .post('/api/v1/meals')
        .set('Authorization', 'Bearer valid-test-token')
        .send({ mode: 'quick' }) // ← blocks ausente
        .expect(400)
        .expect((res) => {
          expect(res.body.error.code).toBe('BAD_REQUEST');
        });
    });

    it('deve retornar 401 quando o token está ausente', async () => {
      return request(app.getHttpServer())
        .post('/api/v1/meals')
        .send({ blocks: 4, mode: 'quick' })
        .expect(401);
    });
  });

  // ─── GET /api/v1/meals ───────────────────────────────────────────────────────

  describe('GET /api/v1/meals', () => {
    it('deve retornar lista paginada de refeições com 200', async () => {
      return request(app.getHttpServer())
        .get('/api/v1/meals?limit=10')
        .set('Authorization', 'Bearer valid-test-token')
        .expect(200)
        .expect((res) => {
          expect(Array.isArray(res.body.data)).toBe(true);
          expect(res.body.meta).toBeDefined();
        });
    });
  });
});
```

### 5.2 Mockando JWT Guard em testes de integração

Para isolar o teste do guard de autenticação, use `.overrideGuard()`:

```typescript
// No mesmo arquivo do teste de integração, antes do app.init():

import { JwtAuthGuard } from '@common/guards/jwt-auth.guard';

// ← Sobrescreve o JwtAuthGuard para simular usuário autenticado
const module: TestingModule = await Test.createTestingModule({
  imports: [/* ... */],
})
  .overrideGuard(JwtAuthGuard)
  .useValue({
    canActivate: (context) => {
      // Injeta o usuário no request para que @CurrentUser() funcione
      const req = context.switchToHttp().getRequest();
      req.user = { id: 'user-uuid-test', email: 'test@example.com' };
      return true;
    },
  })
  .compile();
```

### 5.3 Padrão com Prisma `:memory:` (SQLite)

```typescript
// Quando o projeto usa Prisma (referência: prisma-patterns.md)
// src/modules/users/__tests__/users.controller.spec.ts

import { PrismaService } from '@infra/database/prisma.service';

// Mock do PrismaService usando SQLite in-memory via factory
const mockPrismaService = {
  user: {
    create:    jest.fn(),
    findUnique: jest.fn(),
    findMany:  jest.fn(),
    update:    jest.fn(),
    delete:    jest.fn(),
    count:     jest.fn(),
  },
  $transaction: jest.fn(),
};

const module = await Test.createTestingModule({
  imports: [UsersModule],
})
  .overrideProvider(PrismaService)
  .useValue(mockPrismaService)
  .compile();
```

---

## 6. Testes E2E — TestContainers (PostgreSQL Real)

TestContainers inicia containers Docker programaticamente nos testes. É a opção mais próxima da produção — use para fluxos críticos de negócio e para validar queries complexas que não funcionam corretamente com SQLite.

### 6.1 Instalação

```bash
npm install --save-dev testcontainers @testcontainers/postgresql
```

### 6.2 Setup global — `test/setup-e2e.ts`

```typescript
// test/setup-e2e.ts
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { execSync } from 'child_process';

let container: StartedPostgreSqlContainer;

/**
 * Iniciado UMA VEZ antes de todos os testes E2E (globalSetup no jest-e2e.json).
 * Sobe o PostgreSQL em Docker e exporta a DATABASE_URL para os testes.
 */
module.exports = async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('cab_test')
    .withUsername('test_user')
    .withPassword('test_pass')
    .start();

  // Exporta a URL de conexão para ser consumida pelos módulos de teste
  process.env.DATABASE_URL = container.getConnectionUri();

  // Roda as migrations no banco de teste
  execSync('npx prisma migrate deploy', {
    env: { ...process.env, DATABASE_URL: process.env.DATABASE_URL },
    stdio: 'inherit',
  });

  // Salva a referência do container para o teardown
  (global as any).__TEST_CONTAINER__ = container;
};
```

### 6.3 Teardown global — `test/teardown-e2e.ts`

```typescript
// test/teardown-e2e.ts
/**
 * Executado UMA VEZ após todos os testes E2E.
 * Para e remove o container Docker.
 */
module.exports = async () => {
  const container = (global as any).__TEST_CONTAINER__;
  if (container) {
    await container.stop();
  }
};
```

Adicione o teardown no `jest-e2e.json`:

```json
{
  "globalSetup":    "./test/setup-e2e.ts",
  "globalTeardown": "./test/teardown-e2e.ts"
}
```

### 6.4 Teste E2E completo — `test/app.e2e-spec.ts`

```typescript
// test/app.e2e-spec.ts

import { INestApplication, ValidationPipe } from '@nestjs/common';
import { Test, TestingModule } from '@nestjs/testing';
import * as request from 'supertest';

import { AppModule } from '../src/app.module';
import { TransformInterceptor } from '../src/common/interceptors/transform.interceptor';
import { HttpExceptionFilter } from '../src/common/filters/http-exception.filter';

describe('App (E2E) — Fluxo completo de refeições', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    // DATABASE_URL já foi definida pelo globalSetup (TestContainers)
    const module: TestingModule = await Test.createTestingModule({
      imports: [AppModule],  // ← módulo REAL — sem overrides
    }).compile();

    app = module.createNestApplication();
    app.setGlobalPrefix('api');
    app.useGlobalPipes(
      new ValidationPipe({ whitelist: true, transform: true }),
    );
    app.useGlobalInterceptors(new TransformInterceptor());
    app.useGlobalFilters(new HttpExceptionFilter());

    await app.init();

    // ← Autentica e guarda o token para as próximas requisições
    const loginRes = await request(app.getHttpServer())
      .post('/api/v1/auth/login')
      .send({ email: 'e2e-test@example.com', password: 'TestPass123!' });

    authToken = loginRes.body.data.accessToken;
  });

  afterAll(async () => {
    await app.close();
  });

  // ─── Fluxo: Criar → Buscar → Deletar ─────────────────────────────────────────

  it('fluxo completo: criar, buscar por ID e deletar refeição', async () => {
    // 1. Criar refeição
    const createRes = await request(app.getHttpServer())
      .post('/api/v1/meals')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ blocks: 4, mode: 'quick' })
      .expect(201);

    const mealId = createRes.body.data.id;
    expect(mealId).toBeDefined();

    // 2. Buscar por ID
    const getRes = await request(app.getHttpServer())
      .get(`/api/v1/meals/${mealId}`)
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    expect(getRes.body.data.id).toBe(mealId);
    expect(getRes.body.data.blocks).toBe(4);

    // 3. Deletar
    await request(app.getHttpServer())
      .delete(`/api/v1/meals/${mealId}`)
      .set('Authorization', `Bearer ${authToken}`)
      .expect(204);

    // 4. Confirmar deleção
    await request(app.getHttpServer())
      .get(`/api/v1/meals/${mealId}`)
      .set('Authorization', `Bearer ${authToken}`)
      .expect(404);
  });
});
```

### 6.5 Alternativa sem Docker — `test/setup-e2e.ts` com SQLite

Quando Docker não está disponível no CI (ex: GitHub Actions sem suporte a DinD), use SQLite:

```typescript
// test/setup-e2e.ts — variação sem TestContainers
module.exports = async () => {
  // SQLite in-memory como substituto do PostgreSQL nos E2E
  // ATENÇÃO: algumas queries específicas de PostgreSQL não funcionam no SQLite
  process.env.DATABASE_URL = 'sqlite::memory:';
  process.env.DB_TYPE = 'sqlite';
};
```

> **Quando usar cada abordagem:**
>
> | Abordagem | Vantagem | Limitação |
> |---|---|---|
> | **TestContainers + PostgreSQL** | Banco real, queries nativas, fiel à produção | Requer Docker — mais lento |
> | **SQLite `:memory:`** | Sem Docker, ultra-rápido | Não suporta: `JSONB`, `UUID()`, `ON CONFLICT`, funções PG-específicas |
>
> **Regra:** Use TestContainers para E2E e integração de queries críticas. Use SQLite para integração de Controllers rápida.

---

## 7. Cobertura Mínima por Camada

### 7.1 Tabela de thresholds

| Camada | Arquivo alvo | Branches | Functions | Lines | Justificativa |
|---|---|---|---|---|---|
| **Use Cases** | `**/use-cases/**/*.ts` | **80%** | **85%** | **85%** | Coração da lógica de negócio |
| **Controllers** | `**/controllers/**/*.ts` | **70%** | **80%** | **80%** | Finos — delegam ao UseCase |
| **Repositories** | `**/repositories/**/*.ts` | **60%** | **70%** | **70%** | Lógica simples de banco |
| **Guards/Pipes** | `**/guards/**`, `**/pipes/**` | **75%** | **80%** | **80%** | Segurança crítica |
| **Global** | `src/**/*.ts` | **70%** | **75%** | **80%** | Mínimo aceitável |

### 7.2 O que NÃO incluir na cobertura

```typescript
// jest.config.ts — excluir arquivos sem lógica testável
collectCoverageFrom: [
  'src/**/*.ts',
  '!src/**/*.module.ts',     // Declarações de módulo NestJS
  '!src/**/main.ts',         // Bootstrap — testado via E2E
  '!src/**/*.dto.ts',        // DTOs — validados indiretamente via integração
  '!src/**/*.entity.ts',     // Entities ORM — mapeamento puro
  '!src/**/*.view-model.ts', // ViewModels — mapeamento puro
  '!src/**/*.config.ts',     // Configurações — testadas via e2e ou environment
  '!src/**/*.d.ts',          // Declarações de tipo
  '!src/**/index.ts',        // Barrels de re-export
],
```

---

## 8. Mocking — Boas Práticas

### 8.1 Mock de Repository (TypeORM)

```typescript
// Usando getRepositoryToken para obter o token correto do TypeORM
import { getRepositoryToken } from '@nestjs/typeorm';
import { MealEntity } from '../entities/meal.entity';

const mockTypeOrmRepository = {
  save:          jest.fn(),
  find:          jest.fn(),
  findOne:       jest.fn(),
  delete:        jest.fn(),
  createQueryBuilder: jest.fn(() => ({
    where:         jest.fn().mockReturnThis(),
    andWhere:      jest.fn().mockReturnThis(),
    orderBy:       jest.fn().mockReturnThis(),
    skip:          jest.fn().mockReturnThis(),
    take:          jest.fn().mockReturnThis(),
    getMany:       jest.fn(),
    getManyAndCount: jest.fn(),
    getOne:        jest.fn(),
  })),
};

// No Test.createTestingModule:
{
  provide: getRepositoryToken(MealEntity),
  useValue: mockTypeOrmRepository,
}
```

### 8.2 Mock de Repository Customizado (preferido)

```typescript
// Sempre prefira mockar o Repository customizado, não o do TypeORM.
// O Repository customizado é o contrato público exposto ao UseCase.

const mockMealsRepository: jest.Mocked<Partial<MealsRepository>> = {
  create:         jest.fn(),
  findById:       jest.fn(),
  findByUserId:   jest.fn(),
  softDeleteById: jest.fn(),
};

// No Test.createTestingModule:
{
  provide: MealsRepository,
  useValue: mockMealsRepository,
}
```

### 8.3 Tipagem de mocks com `jest.Mocked<T>`

```typescript
// Garante type-safety nos mocks — autocompletar + verificação de tipo
let repository: jest.Mocked<MealsRepository>;

// Uso:
repository.create.mockResolvedValue(mockMeal);       // ← Promise<MealEntity>
repository.findById.mockResolvedValueOnce(null);      // ← apenas uma vez
repository.create.mockRejectedValue(new Error('DB')); // ← simula falha
```

### 8.4 Evitar Over-mocking

```typescript
// ❌ ERRADO — Mock tão extenso que não testa nada real
it('deve chamar create', async () => {
  mealsRepository.create.mockResolvedValue(mockMeal);
  await useCase.execute(input);
  expect(mealsRepository.create).toHaveBeenCalled(); // ← trivial
});

// ✅ CORRETO — Verifica o comportamento real, não apenas que foi chamado
it('deve criar refeição com userId do caller', async () => {
  mealsRepository.create.mockResolvedValue(mockMeal);

  const result = await useCase.execute({ userId: 'user-123', blocks: 4 });

  expect(mealsRepository.create).toHaveBeenCalledWith(
    expect.objectContaining({ userId: 'user-123' }), // ← dados específicos
  );
  expect(result.blocks).toBe(4); // ← resultado verificado
});
```

---

## 9. Integração com CI/CD

### 9.1 GitHub Actions — Pipeline completo

```yaml
# .github/workflows/ci.yml

name: CI — Lint, Testes e Cobertura

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

jobs:
  test:
    name: Testes Unitários e de Integração
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - name: Lint e Type-check
        run: |
          npm run lint
          npx tsc --noEmit

      - name: Testes (unitário + integração) com cobertura
        run: npm run test:ci
        env:
          NODE_ENV: test

      - name: Upload cobertura (opcional)
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: false

  test-e2e:
    name: Testes E2E (TestContainers)
    runs-on: ubuntu-latest
    timeout-minutes: 30
    # Requer Docker disponível no runner

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - name: Testes E2E
        run: npm run test:e2e
        env:
          NODE_ENV: test
          # DATABASE_URL é definida pelo TestContainers em setup-e2e.ts
```

---

## 10. Cobertura por Tipo de Use Case — Casos Obrigatórios

Para cada Use Case, os seguintes casos de teste são **obrigatórios**:

| Tipo | Cenário obrigatório |
|---|---|
| **Criação** | Sucesso com dados válidos; Falha de validação de negócio; Propagação de erro do Repository |
| **Busca por ID** | Sucesso; Não encontrado (retorna `null` ou lança `NotFoundException`) |
| **Listagem** | Sucesso (retorna lista); Lista vazia (retorna `[]`) |
| **Atualização** | Sucesso; Recurso não encontrado; Violação de regra de negócio |
| **Deleção** | Sucesso; Recurso não encontrado |
| **Cross-domain** | Todos os repositórios envolvidos são chamados; Rollback em caso de falha |

---

## 11. Nomenclatura de `describe` e `it`

Use padrão hierárquico que lê como uma especificação:

```typescript
describe('CreateMealUseCase', () => {           // ← Nome da classe
  describe('execute', () => {                   // ← Nome do método
    describe('quando os dados são válidos', () => {
      it('deve criar a refeição e retornar a entity', async () => { ... });
      it('deve chamar repository.create com o userId do usuário autenticado', async () => { ... });
    });

    describe('quando blocks é inválido', () => {
      it('deve lançar UnprocessableEntityException quando blocks = 0', async () => { ... });
      it('deve lançar UnprocessableEntityException quando blocks > 20', async () => { ... });
      it('nunca deve chamar o repository quando a validação falha', async () => { ... });
    });

    describe('quando o repository falha', () => {
      it('deve propagar o erro sem modificar a mensagem', async () => { ... });
    });
  });
});
```

---

## 12. Anti-Padrões Comuns

### ❌ Testar o módulo NestJS em vez da lógica

```typescript
// ❌ ERRADO — testa a configuração do NestJS, não a lógica de negócio
it('CreateMealUseCase deve ser definido', () => {
  expect(useCase).toBeDefined();
});
// ✅ CORRETO — teste que valida o comportamento real
it('deve criar uma refeição com blocks = 4', async () => {
  const result = await useCase.execute({ blocks: 4, userId: 'u1', mode: 'quick' });
  expect(result.blocks).toBe(4);
});
```

### ❌ Testar implementação, não comportamento

```typescript
// ❌ ERRADO — testa detalhes internos (como a lógica chama o repositório)
it('deve chamar repository.save() internamente', async () => {
  await useCase.execute(input);
  // Isso acopla o teste à implementação interna — quebra se refatorar sem mudar o comportamento
  expect(repository.save).toHaveBeenCalled(); // ← muito detalhado
});

// ✅ CORRETO — testa o comportamento observável
it('deve persistir a refeição e retornar com ID gerado', async () => {
  const result = await useCase.execute(input);
  expect(result.id).toBeDefined();
  expect(result.createdAt).toBeInstanceOf(Date);
});
```

### ❌ Usar `beforeAll` para resetar mocks (vazamento entre testes)

```typescript
// ❌ ERRADO — estado de mock pode vazar entre testes
beforeAll(() => {
  mealsRepository.create.mockResolvedValue(mockMeal);
});

// ✅ CORRETO — reset a cada teste garante isolamento
beforeEach(() => {
  jest.clearAllMocks(); // ou jest.resetAllMocks()
});
```

### ❌ `synchronize: true` fora dos testes de integração

```typescript
// ❌ NUNCA em produção ou staging — apenas em testes com banco in-memory
TypeOrmModule.forRoot({
  synchronize: true, // ← SOMENTE em testes com database: ':memory:'
})
```

### ❌ Testes E2E sem limpeza do banco entre suítes

```typescript
// ❌ ERRADO — dados de um teste contaminam o próximo
afterAll(async () => {
  await app.close(); // ← sem limpar o banco
});

// ✅ CORRETO — trunca as tabelas relevantes entre suítes
afterEach(async () => {
  const connection = app.get(DataSource);
  await connection.query('DELETE FROM meals WHERE user_id = $1', ['test-user-uuid']);
});
```

---

## 13. Checklist de Validação

### Unitários
- [ ] Cada Use Case tem arquivo `.spec.ts` co-localizado?
- [ ] `Test.createTestingModule` com apenas o UseCase + Repository mockado?
- [ ] Mock é do Repository **customizado** — não do `Repository<Entity>` do TypeORM/Prisma?
- [ ] `afterEach(() => jest.clearAllMocks())` configurado?
- [ ] Casos de sucesso, erro de negócio e erro de infraestrutura cobertos?
- [ ] Sem `toBeDefined()` como único assert — ao menos 1 assert de comportamento?

### Integração de Controllers
- [ ] `Test.createTestingModule` com o módulo real + SQLite `:memory:`?
- [ ] `JwtAuthGuard` sobrescrito via `.overrideGuard()` onde necessário?
- [ ] Mesmo `ValidationPipe` e `HttpExceptionFilter` do `main.ts` configurados?
- [ ] Endpoints de sucesso E de erro validados (400, 401, 404, 422)?
- [ ] Envelope `{ data }` e `{ error }` verificados?

### E2E
- [ ] `globalSetup` inicia TestContainers (ou configura SQLite)?
- [ ] `globalTeardown` para e remove o container?
- [ ] `--runInBand` no script `test:e2e` (sem paralelismo)?
- [ ] Banco limpo entre suítes para evitar interferência?
- [ ] Testa apenas fluxos críticos de negócio (não todas as rotas)?

### Cobertura e CI
- [ ] `coverageThreshold` configurado no `jest.config.ts`?
- [ ] Arquivos sem lógica (`*.module.ts`, `*.entity.ts`, `*.dto.ts`) excluídos?
- [ ] Script `test:ci` com `--ci --coverage --forceExit` para CI/CD?
- [ ] Testes E2E em job separado no pipeline (mais lento — não bloqueia unit/integration)?

---

## Referências

- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Camadas e responsabilidades; o que cada camada deve testar
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura de pastas `__tests__/`, Use Cases Pattern, seção 8 — Teste de Use Cases
- [dto-validation.md](.agent/skills/backend/api-and-contracts/dto-validation.md) — `ValidationPipe` global que precisa ser replicado nos testes de integração
- [rest-api-patterns.md](.agent/skills/backend/api-and-contracts/rest-api-patterns.md) — Envelope `{ data, meta, error }` verificado nos testes de integração
- [typeorm-patterns.md](.agent/skills/backend/database/typeorm-patterns.md) — `getRepositoryToken` e mock de Repository TypeORM
- [prisma-patterns.md](.agent/skills/backend/database/prisma-patterns.md) — Mock do `PrismaService` em testes
- [NestJS Docs — Testing](https://docs.nestjs.com/fundamentals/testing)
- [Testcontainers — Node.js](https://testcontainers.com/guides/getting-started-with-testcontainers-for-nodejs/)
- [Jest — Configuration](https://jestjs.io/docs/configuration)
- [Supertest — GitHub](https://github.com/ladjs/supertest)
- [jmcdo29/testing-nestjs — GitHub](https://github.com/jmcdo29/testing-nestjs) — Repositório de referência da comunidade