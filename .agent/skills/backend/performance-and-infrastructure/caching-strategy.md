---
name: caching-strategy
description: Estratégia completa de cache em múltiplas camadas para NestJS — CacheModule global com Redis via @keyv/redis (consistente com AppCacheModule de query-optimization.md), cache em nível de rota com @CacheKey e @CacheTTL, cache manual em Repositories e Services com cache-aside pattern, estratégias de invalidação por chave e por prefixo via SCAN, TTL por tipo de dado, e CacheKeysService para gestão centralizada de chaves. Use esta skill como a referência canônica sobre caching; query-optimization.md referencia esta skill para a configuração do AppCacheModule.
---

# Estratégia de Cache em Múltiplas Camadas — NestJS + Redis

Você é um Engenheiro de Software Senior especialista em performance e caching distribuído. Esta skill define o **padrão completo de caching** do projeto NestJS, cobrindo todas as camadas: módulo global, rota, service/repository, e invalidação.

> **Decisões de arquitetura (não altere sem revisão):**
> - `AppCacheModule` usa `@keyv/redis` — o mesmo pacote já referenciado em `query-optimization.md`. Não use `cache-manager-redis-store` (descontinuado) nem `cache-manager-redis-yet`.
> - TTL é sempre em **milissegundos** no `cache-manager` v5+.
> - Cache manual no Repository é o padrão preferido para dados de domínio — dá controle total sobre TTL e invalidação.
> - `CacheInterceptor` (`@CacheKey` + `@CacheTTL`) é reservado para endpoints **sem contexto de usuário** (dados verdadeiramente estáticos).
> - Invalidação por padrão (prefix scan) usa cliente Redis direto via `ioredis` — nunca use `KEYS *` em produção; sempre use `SCAN`.
> - Cache **nunca** é invalidado antes de uma escrita bem-sucedida — sempre invalidar **após** commit.

---

## 1. Visão Geral das Camadas

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CAMADAS DE CACHE (ordem de consulta)               │
├─────────────────────────────────────────────────────────────────────┤
│  Camada 1 → CacheInterceptor (rota)                                 │
│             ↳ Cacheia response HTTP completa por URL                │
│             ↳ Apenas GET sem contexto de usuário                    │
│             ↳ Decoradores: @CacheKey, @CacheTTL, @UseInterceptors   │
├─────────────────────────────────────────────────────────────────────┤
│  Camada 2 → Cache manual no Repository/Service (cache-aside)        │
│             ↳ cache.get() → DB miss → cache.set()                   │
│             ↳ Controle granular de chave e TTL por tipo de dado     │
│             ↳ Invalidação explícita em operações de escrita         │
├─────────────────────────────────────────────────────────────────────┤
│  Camada 3 → Redis (store)                                           │
│             ↳ Persistência distribuída entre instâncias             │
│             ↳ Fallback: in-memory em desenvolvimento local          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Instalação

```bash
npm install @nestjs/cache-manager cache-manager @keyv/redis
npm install ioredis          # cliente direto para operações de SCAN/pattern delete
npm install -D @types/ioredis
```

> **Por que `@keyv/redis` e não outros stores?** É o pacote rastreado diretamente pela equipe NestJS, compatível com `cache-manager` v5+, e já referenciado em `query-optimization.md`. Mantém consistência no projeto.

---

## 3. Configuração — `AppCacheModule`

### 3.1 Variáveis de Ambiente

Complementa `config-management.md` — adicione ao `env.config.ts`:

```typescript
// src/config/env.config.ts — adicione ao schema existente:
const envSchema = z.object({
  // ... variáveis existentes
  REDIS_URL: z.string().url().optional(),       // ex: redis://localhost:6379
  CACHE_TTL_SECONDS: z.coerce.number().default(300), // 5 min padrão
});
```

### 3.2 `cache.config.ts`

```typescript
// src/config/cache.config.ts
import { registerAs } from '@nestjs/config';
import { env } from './env.config';

export interface CacheConfig {
  ttlSeconds: number;
  redisUrl?: string;
}

export default registerAs('cache', (): CacheConfig => ({
  ttlSeconds: env.CACHE_TTL_SECONDS,
  redisUrl: env.REDIS_URL,
}));
```

### 3.3 `AppCacheModule` — Módulo Global

```typescript
// src/infra/cache/cache.module.ts
import { Global, Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { ConfigModule, ConfigType } from '@nestjs/config';
import { createKeyv } from '@keyv/redis';
import cacheConfig from '@config/cache.config';

@Global()   // ← disponível em todos os módulos sem reimportar
@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      imports: [ConfigModule],
      inject: [cacheConfig.KEY],
      useFactory: (config: ConfigType<typeof cacheConfig>) => ({
        store: config.redisUrl
          ? createKeyv(config.redisUrl)
          : undefined,                    // undefined → in-memory em dev
        ttl: config.ttlSeconds * 1000,    // cache-manager v5+: milissegundos
      }),
    }),
  ],
})
export class AppCacheModule {}
```

### 3.4 Registro no `AppModule`

```typescript
// src/app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [cacheConfig, databaseConfig] }),
    DatabaseModule,
    AppCacheModule,   // ← APÓS DatabaseModule, ANTES dos módulos de domínio
    AuthModule,
    UsersModule,
    // ... demais módulos de domínio
  ],
})
export class AppModule {}
```

---

## 4. Camada 1 — Cache de Rota com `CacheInterceptor`

Use para endpoints **completamente estáticos** — dados que não variam por usuário, papel, tenant ou qualquer parâmetro de contexto.

```typescript
// src/modules/categories/controllers/categories.controller.ts
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { CacheInterceptor, CacheTTL, CacheKey } from '@nestjs/cache-manager';
import { ApiTags, ApiOperation } from '@nestjs/swagger';
import { Public } from '@common/decorators/public.decorator';
import { GetCategoriesUseCase } from '../use-cases/get-categories/get-categories.use-case';
import { CategoryViewModel } from '../view-models/category.view-model';

@ApiTags('categories')
@Controller({ path: 'categories', version: '1' })
@UseInterceptors(CacheInterceptor)   // ← aplica a todos os GETs do controller
export class CategoriesController {
  constructor(private readonly getCategoriesUseCase: GetCategoriesUseCase) {}

  @Public()
  @Get()
  @CacheKey('categories:all')          // ← chave explícita (não depende da URL)
  @CacheTTL(15 * 60 * 1000)           // ← 15 minutos (ms)
  @ApiOperation({ summary: 'Lista todas as categorias (cached 15min)' })
  async findAll(): Promise<CategoryViewModel[]> {
    const categories = await this.getCategoriesUseCase.execute();
    return CategoryViewModel.fromEntities(categories);
  }
}
```

> **Quando NÃO usar `CacheInterceptor`:**
> - Endpoint usa `@CurrentUser()` → cache seria compartilhado entre usuários
> - Endpoint retorna dados paginados com querystring dinâmica
> - Endpoint filtra por `role` ou `tenantId`
> - GraphQL resolvers (não suportado pelo NestJS `CacheInterceptor`)

---

## 5. Camada 2 — Cache Manual no Repository (Cache-Aside Pattern)

Padrão principal para dados de domínio. O Repository é responsável pela lógica de cache.

### 5.1 Estrutura de Chaves — `CacheKeysService`

Centralize a construção de chaves para evitar strings mágicas espalhadas no código:

```typescript
// src/infra/cache/cache-keys.service.ts
import { Injectable } from '@nestjs/common';

/**
 * Fonte de verdade para todas as chaves de cache do projeto.
 * Padrão: <namespace>:<entidade>:<identificador>
 * Exemplo: foods:catalog, foods:item:abc123, users:profile:xyz789
 */
@Injectable()
export class CacheKeysService {
  // ─── Foods ────────────────────────────────────────────────────────────────
  readonly foods = {
    catalog: () => 'foods:catalog',
    item: (id: string) => `foods:item:${id}`,
    prefix: () => 'foods:',
  };

  // ─── Users ────────────────────────────────────────────────────────────────
  readonly users = {
    profile: (id: string) => `users:profile:${id}`,
    prefix: () => 'users:',
  };

  // ─── Categories ───────────────────────────────────────────────────────────
  readonly categories = {
    all: () => 'categories:all',
    prefix: () => 'categories:',
  };

  // ─── Reports (TTL longo — dados caros de computar) ────────────────────────
  readonly reports = {
    nutritionSummary: (userId: string, period: string) =>
      `reports:nutrition:${userId}:${period}`,
    prefix: (userId: string) => `reports:nutrition:${userId}:`,
  };
}
```

```typescript
// src/infra/cache/cache.module.ts — exporte o service junto com o módulo
@Global()
@Module({
  imports: [CacheModule.registerAsync({ ... })],
  providers: [CacheKeysService],
  exports: [CacheKeysService],   // ← disponível em toda aplicação
})
export class AppCacheModule {}
```

### 5.2 Cache Manual no Repository

```typescript
// src/modules/foods/repositories/foods.repository.ts
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import { PrismaService } from '@infra/database/prisma.service';
import { CacheKeysService } from '@infra/cache/cache-keys.service';
import { Food } from '@prisma/client';

// ─── TTL por tipo de dado ──────────────────────────────────────────────────
// Defina aqui perto do uso para fácil auditoria.
// Consulte a seção 7 desta skill para a tabela completa de TTLs.
const TTL = {
  CATALOG: 10 * 60 * 1000,   // 10 min — catálogo muda raramente
  ITEM:     5 * 60 * 1000,   //  5 min — item individual
} as const;

@Injectable()
export class FoodsRepository {
  constructor(
    private readonly prisma: PrismaService,
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
    private readonly keys: CacheKeysService,
  ) {}

  // ─── Leitura com cache-aside ───────────────────────────────────────────────

  async findCatalog(): Promise<Food[]> {
    const key = this.keys.foods.catalog();

    const cached = await this.cache.get<Food[]>(key);
    if (cached) return cached;

    const foods = await this.prisma.food.findMany({
      where: { isActive: true },
      orderBy: { name: 'asc' },
    });

    await this.cache.set(key, foods, TTL.CATALOG);
    return foods;
  }

  async findById(id: string): Promise<Food | null> {
    const key = this.keys.foods.item(id);

    const cached = await this.cache.get<Food>(key);
    if (cached) return cached;

    const food = await this.prisma.food.findUnique({ where: { id } });
    if (food) {
      await this.cache.set(key, food, TTL.ITEM);
    }

    return food;
  }

  // ─── Escrita com invalidação pós-commit ───────────────────────────────────
  // REGRA: invalidar APÓS operação bem-sucedida, NUNCA antes.

  async create(data: Prisma.FoodCreateInput): Promise<Food> {
    const food = await this.prisma.food.create({ data });

    // Catálogo ficou desatualizado — invalidar
    await this.cache.del(this.keys.foods.catalog());

    return food;
  }

  async update(id: string, data: Prisma.FoodUpdateInput): Promise<Food> {
    const food = await this.prisma.food.update({ where: { id }, data });

    // Invalidar catálogo e o item específico
    await Promise.all([
      this.cache.del(this.keys.foods.catalog()),
      this.cache.del(this.keys.foods.item(id)),
    ]);

    return food;
  }

  async delete(id: string): Promise<void> {
    await this.prisma.food.delete({ where: { id } });

    await Promise.all([
      this.cache.del(this.keys.foods.catalog()),
      this.cache.del(this.keys.foods.item(id)),
    ]);
  }
}
```

### 5.3 Cache em Use Cases (quando o dado cruza múltiplos Repositories)

Para dados que exigem múltiplas fontes e são caros de computar (ex: relatórios), o cache pode viver no Use Case em vez do Repository:

```typescript
// src/modules/reports/use-cases/get-nutrition-summary/get-nutrition-summary.use-case.ts
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import { CacheKeysService } from '@infra/cache/cache-keys.service';

const REPORT_TTL = 30 * 60 * 1000; // 30 min — relatórios são caros; usuário tolera staleness

@Injectable()
export class GetNutritionSummaryUseCase {
  constructor(
    private readonly mealsRepository: MealsRepository,
    private readonly foodsRepository: FoodsRepository,
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
    private readonly keys: CacheKeysService,
  ) {}

  async execute(input: { userId: string; period: string }): Promise<NutritionSummary> {
    const key = this.keys.reports.nutritionSummary(input.userId, input.period);

    const cached = await this.cache.get<NutritionSummary>(key);
    if (cached) return cached;

    // Computação cara — múltiplos queries + agregações
    const meals = await this.mealsRepository.findByUserAndPeriod(input);
    const summary = await this.computeSummary(meals);

    await this.cache.set(key, summary, REPORT_TTL);
    return summary;
  }

  // Chamado quando o usuário registra uma refeição — relatório ficou desatualizado
  async invalidateForUser(userId: string): Promise<void> {
    // Usa invalidação por prefixo (ver seção 6.2)
    // Invalida todos os períodos deste usuário de uma vez
    await this.cacheUtils.deleteByPrefix(this.keys.reports.prefix(userId));
  }

  private async computeSummary(meals: Meal[]): Promise<NutritionSummary> {
    // ... lógica de agregação
  }
}
```

---

## 6. Estratégias de Invalidação

### 6.1 Invalidação por Chave Exata (padrão, preferido)

```typescript
// Delete de uma chave específica
await this.cache.del('foods:item:abc123');

// Delete de múltiplas chaves em paralelo
await Promise.all([
  this.cache.del('foods:catalog'),
  this.cache.del('foods:item:abc123'),
  this.cache.del('categories:all'),
]);
```

> Use sempre que souber exatamente quais chaves foram afetadas pela operação. É o método mais eficiente e seguro.

### 6.2 Invalidação por Prefixo com SCAN (para grupos de chaves)

Quando não é possível saber todas as chaves afetadas (ex: todos os relatórios de um usuário), use `SCAN` com padrão. **Nunca use `KEYS *` em produção** — bloqueia o event loop do Redis.

```typescript
// src/infra/cache/cache-utils.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import Redis from 'ioredis';

@Injectable()
export class CacheUtilsService {
  private readonly redis: Redis;

  constructor(private readonly config: ConfigService) {
    const redisUrl = this.config.get<string>('cache.redisUrl');
    // Cria cliente separado — o cache-manager não expõe o cliente diretamente
    this.redis = redisUrl ? new Redis(redisUrl) : new Redis(); // localhost padrão
  }

  /**
   * Deleta todas as chaves que começam com o prefixo fornecido.
   * Usa SCAN para não bloquear o Redis em datasets grandes.
   *
   * ATENÇÃO: Use apenas quando a invalidação por chave exata não for viável.
   * Em Redis com muitas chaves, SCAN ainda pode ser lento — use com critério.
   */
  async deleteByPrefix(prefix: string): Promise<number> {
    let cursor = '0';
    let deletedCount = 0;

    do {
      const [nextCursor, keys] = await this.redis.scan(
        cursor,
        'MATCH', `${prefix}*`,
        'COUNT', 100,   // processa em lotes de 100 — ajuste conforme densidade
      );

      cursor = nextCursor;

      if (keys.length > 0) {
        await this.redis.del(...keys);
        deletedCount += keys.length;
      }
    } while (cursor !== '0');

    return deletedCount;
  }

  async onModuleDestroy(): Promise<void> {
    await this.redis.quit();
  }
}
```

```typescript
// src/infra/cache/cache.module.ts — exporte CacheUtilsService
@Global()
@Module({
  imports: [CacheModule.registerAsync({ ... })],
  providers: [CacheKeysService, CacheUtilsService],
  exports: [CacheKeysService, CacheUtilsService],
})
export class AppCacheModule {}
```

### 6.3 Invalidação por TTL (automática, passiva)

A estratégia mais simples — deixar o dado expirar naturalmente. Use quando:
- Staleness por um período curto é aceitável (ex: estatísticas de dashboard)
- O custo de invalidação explícita não vale a complexidade
- Dados são somente leitura (sem operações de escrita no sistema)

```typescript
// Exemplo: dados de saúde/status do sistema — expiram em 1 min automaticamente
await this.cache.set('system:health:summary', healthData, 60 * 1000);
// Não precisa de invalidação explícita — expira sozinho
```

---

## 7. TTL por Tipo de Dado

Defina TTLs próximos do uso (na constante `TTL` do arquivo do Repository/UseCase). A tabela abaixo é o **guia de referência** — ajuste conforme a frequência real de atualização do dado no seu domínio.

| Tipo de Dado | TTL Sugerido | Justificativa |
|---|---|---|
| Catálogo de itens (lista pública) | 10–15 min | Muda raramente; admins atualizam esporadicamente |
| Item individual (por ID) | 5 min | Pode ser editado pelo usuário |
| Perfil de usuário | 5 min | Editável; tolerância baixa a staleness |
| Relatórios / agregações | 30–60 min | Caro de computar; usuário tolera dados com atraso |
| Dados de referência (enums, config) | 1–6 h | Quase imutáveis; mudança requer deploy ou admin |
| Dados estáticos totais (país, moeda) | 24 h | Praticamente imutáveis |
| Blacklist de tokens JWT | TTL residual do token | Segurança — deve expirar exatamente com o token |
| Session / estado temporário | 15 min | Deve expirar com inatividade |

> **Regra de ouro:** se o dado tem uma operação de escrita conhecida no sistema, prefira **invalidação explícita + TTL longo** (garante consistência). Se não tem escrita conhecida, use **TTL curto** (garante frescor sem complexidade).

---

## 8. O que Não Cachear

```typescript
// ❌ NUNCA cache para dados transacionais ou com consistência forte requerida
// Exemplos:
- Saldo de conta / pontuação em tempo real
- Pedidos em processamento
- Status de pagamento
- Dados de autenticação (senha, tokens ativos — exceto blacklist)
- Dados de auditoria / logs

// ❌ NUNCA cache para dados específicos por usuário no CacheInterceptor
@Get('profile')
@UseInterceptors(CacheInterceptor)  // ERRADO — todos os usuários receberiam o mesmo perfil
async getProfile(@CurrentUser() user: UserPayload) { ... }

// ✅ Para dados de usuário, use cache manual com userId na chave
async getProfile(userId: string): Promise<UserProfile> {
  const key = this.keys.users.profile(userId);   // 'users:profile:abc123'
  // ...
}
```

---

## 9. Testes

### 9.1 Mock do `CACHE_MANAGER` em Testes Unitários

```typescript
// src/modules/foods/__tests__/foods.repository.spec.ts
import { Test } from '@nestjs/testing';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { FoodsRepository } from '../repositories/foods.repository';
import { CacheKeysService } from '@infra/cache/cache-keys.service';

const mockCache = {
  get: jest.fn(),
  set: jest.fn(),
  del: jest.fn(),
};

const mockPrisma = {
  food: {
    findMany: jest.fn(),
    findUnique: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
  },
};

describe('FoodsRepository', () => {
  let repository: FoodsRepository;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        FoodsRepository,
        CacheKeysService,
        { provide: CACHE_MANAGER, useValue: mockCache },
        { provide: PrismaService, useValue: mockPrisma },
      ],
    }).compile();

    repository = module.get(FoodsRepository);
    jest.clearAllMocks();
  });

  describe('findCatalog', () => {
    it('deve retornar dados do cache quando disponíveis (cache hit)', async () => {
      const cachedFoods = [{ id: '1', name: 'Arroz' }];
      mockCache.get.mockResolvedValue(cachedFoods);

      const result = await repository.findCatalog();

      expect(result).toEqual(cachedFoods);
      expect(mockPrisma.food.findMany).not.toHaveBeenCalled(); // banco não foi consultado
    });

    it('deve consultar o banco e salvar no cache em caso de cache miss', async () => {
      const dbFoods = [{ id: '1', name: 'Arroz' }];
      mockCache.get.mockResolvedValue(null);     // cache miss
      mockPrisma.food.findMany.mockResolvedValue(dbFoods);

      const result = await repository.findCatalog();

      expect(result).toEqual(dbFoods);
      expect(mockCache.set).toHaveBeenCalledWith(
        'foods:catalog',
        dbFoods,
        expect.any(Number), // TTL
      );
    });
  });

  describe('create', () => {
    it('deve invalidar o cache do catálogo após criar um food', async () => {
      const newFood = { id: '2', name: 'Feijão' };
      mockPrisma.food.create.mockResolvedValue(newFood);

      await repository.create({ name: 'Feijão', isActive: true });

      expect(mockCache.del).toHaveBeenCalledWith('foods:catalog');
    });
  });
});
```

### 9.2 Checklist de Testes de Cache

- [ ] Cache hit: banco **não** é chamado quando cache retorna dados
- [ ] Cache miss: banco **é** chamado e resultado **é** salvo no cache
- [ ] Operação de escrita: `cache.del` é chamado com a(s) chave(s) corretas
- [ ] `cache.del` é chamado **após** operação bem-sucedida (não antes)
- [ ] TTL passado ao `cache.set` é o correto para o tipo de dado

---

## 10. Checklist de Implementação

### Configuração
- [ ] `REDIS_URL` e `CACHE_TTL_SECONDS` adicionados ao `env.config.ts` com Zod?
- [ ] `cache.config.ts` criado com `registerAs('cache', ...)`?
- [ ] `AppCacheModule` usa `@keyv/redis` (não `cache-manager-redis-store`)?
- [ ] `AppCacheModule` registrado no `AppModule` após `DatabaseModule`?
- [ ] `CacheKeysService` e `CacheUtilsService` exportados do `AppCacheModule`?

### Cache de Rota
- [ ] `CacheInterceptor` é usado **somente** em endpoints sem `@CurrentUser()`?
- [ ] `@CacheKey` explícita definida (não depende da URL como chave padrão)?
- [ ] `@CacheTTL` em ms (não segundos — `cache-manager` v5)?

### Cache Manual
- [ ] Cache-aside pattern: `get` → miss → DB → `set`?
- [ ] Chaves construídas via `CacheKeysService` (sem strings mágicas inline)?
- [ ] TTL declarado como constante próxima ao uso?
- [ ] Todo `cache.set` tem TTL definido explicitamente?

### Invalidação
- [ ] Invalidação ocorre **após** operação bem-sucedida de escrita?
- [ ] Invalidações múltiplas usam `Promise.all` para paralelismo?
- [ ] `SCAN` (via `CacheUtilsService`) usado no lugar de `KEYS *` para padrões?
- [ ] Operações de delete, update e create invalidam **todas** as chaves afetadas?

### O que Não Cachear
- [ ] Dados transacionais (saldo, pedidos) **não** estão em cache?
- [ ] `CacheInterceptor` **não** é usado em endpoints que dependem de contexto de usuário?
- [ ] Senhas e tokens brutos **jamais** passam pelo cache?

---

## 11. Referências

- [query-optimization.md](.agent/skills/backend/database/query-optimization.md) — Seção 4: Contexto de cache de queries (referencia esta skill para detalhes)
- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — `registerAs`, acesso tipado a config namespaces
- [auth-jwt-pattern.md](.agent/skills/backend/authentication-and-security/auth-jwt-pattern.md) — Blacklist Redis de access tokens (usa `AppCacheModule`)
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — `src/infra/cache/` como local canônico do `AppCacheModule`
- [dependency-injection.md](.agent/skills/backend/configuration-and-environment/dependency-injection.md) — `@Inject(CACHE_MANAGER)` como token de injeção
- [testing-strategy-nest.md](.agent/skills/backend/quality-and-testing/testing-strategy-nest.md) — Mock de `CACHE_MANAGER` em testes unitários
- [NestJS Docs — Caching](https://docs.nestjs.com/techniques/caching)
- [cache-manager v5 — milissegundos](https://github.com/node-cache-manager/node-cache-manager#readme)
- [Redis Docs — SCAN vs KEYS](https://redis.io/commands/scan/)