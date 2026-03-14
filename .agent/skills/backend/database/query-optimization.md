---
name: query-optimization
description: Otimização de queries no NestJS — N+1 com eager/lazy loading estratégico e DataLoader, uso de índices por padrão de acesso, Query Builder vs raw SQL quando e por quê, cache de queries com Redis via @nestjs/cache-manager, e como usar o query logger do TypeORM e Prisma para detectar e medir problemas de performance.
---

# Query Optimization — Performance de Banco de Dados no NestJS

Você é um Engenheiro de Software Senior especialista em performance e banco de dados. Esta skill é a **referência de otimização de queries** para o projeto. Ela complementa `typeorm-patterns.md` (seção 5 — índices e QueryBuilder), `prisma-patterns.md` (seção 5 — tipagem segura e `$queryRaw`), `database-transactions.md` (custo e isolamento de transações), e `config-management.md` (variáveis e ambientes).

> **Princípio fundamental:** Otimize com evidências. Use o query logger para identificar problemas reais antes de adicionar índices, cache ou refatorar queries. Otimização prematura é a raiz de todo o mal — otimização com dados é engenharia.

---

## 1. O Problema N+1 — Identificação e Correção

### O que é N+1

O problema N+1 ocorre quando uma query retorna N registros e, para cada um deles, é executada **uma query adicional** para carregar dados relacionados — resultando em `1 + N` queries no banco.

```
# Exemplo: buscar 20 refeições com seus alimentos
SELECT * FROM meals LIMIT 20;      ← 1 query

# Para cada refeição (N=20), uma query extra:
SELECT * FROM foods WHERE mealId = '...'; ← query 1
SELECT * FROM foods WHERE mealId = '...'; ← query 2
SELECT * FROM foods WHERE mealId = '...'; ← query 3
... × 20                                  ← 20 queries adicionais

# Total: 21 queries → deveria ser 1 (com JOIN) ou 2 (com IN)
```

### 1.1 Detecção via Query Logger

Antes de corrigir qualquer N+1, confirme o problema com os logs. Veja a seção 5 (Query Logger) desta skill para configuração completa.

**Sintoma no log:** múltiplas queries idênticas com `WHERE relation_id IN (...)` ou queries repetidas com único ID variando.

### 1.2 TypeORM — Solução com JOIN Explícito

A solução correta no TypeORM é carregar relações com `leftJoinAndSelect` ou `relations` no `findOne`/`find`:

```typescript
// ❌ PROBLEMA: lazy loading causando N+1
async findMealsWithFoods(userId: string): Promise<MealEntity[]> {
  const meals = await this.repo.find({ where: { userId } });
  // Para cada meal, acessar meal.foods vai disparar uma query — N+1!
  return meals;
}

// ✅ SOLUÇÃO 1: relations option (para casos simples com findMany)
async findMealsWithFoods(userId: string): Promise<MealEntity[]> {
  return this.repo.find({
    where: { userId },
    relations: { foods: true },         // TypeORM faz JOIN em uma query
    order: { createdAt: 'DESC' },
  });
}

// ✅ SOLUÇÃO 2: QueryBuilder com JOIN explícito (para filtros dinâmicos)
async findMealsWithFoodsByDateRange(
  userId: string,
  from: Date,
  to: Date,
): Promise<MealEntity[]> {
  return this.repo
    .createQueryBuilder('meal')
    .leftJoinAndSelect('meal.foods', 'food') // ← JOIN em uma única query
    .where('meal.userId = :userId', { userId })
    .andWhere('meal.createdAt BETWEEN :from AND :to', { from, to })
    .orderBy('meal.createdAt', 'DESC')
    .getMany();
}
```

### 1.3 Prisma — Solução com `include` e `select` Aninhado

No Prisma, use `include` para carregar relações em uma única query:

```typescript
// ❌ PROBLEMA: buscar meals e depois acessar foods individualmente
async findMealsWithFoods(userId: string) {
  const meals = await this.prisma.meal.findMany({ where: { userId } });
  // Acessar meals[0].foods em um loop vai disparar queries adicionais
  return meals;
}

// ✅ SOLUÇÃO 1: include (carrega a relação completa)
async findMealsWithFoods(userId: string) {
  return this.prisma.meal.findMany({
    where: { userId, deletedAt: null },
    include: { foods: true },            // ← JOIN em uma única query
    orderBy: { createdAt: 'desc' },
  });
}

// ✅ SOLUÇÃO 2: select aninhado (mais eficiente — seleciona apenas o necessário)
async findMealsWithFoodSummary(userId: string) {
  return this.prisma.meal.findMany({
    where: { userId, deletedAt: null },
    select: {
      id: true,
      blocks: true,
      createdAt: true,
      foods: {
        select: { name: true, calories: true },  // ← apenas 2 campos dos alimentos
      },
    },
    orderBy: { createdAt: 'desc' },
  });
}
```

### 1.4 DataLoader — Para APIs GraphQL com Relações Dinâmicas

Em APIs GraphQL (cf. `graphql-pattern.md`), o padrão N+1 é estrutural: cada resolver de campo pode disparar queries independentes. O `DataLoader` resolve isso com **batching**.

> **Contexto:** DataLoader é obrigatório em GraphQL. Para REST, prefira JOIN explícito no Repository (seções 1.2 e 1.3 acima).

```typescript
// Instalação
// npm install dataloader

// src/modules/meals/loaders/foods.loader.ts
import * as DataLoader from 'dataloader';
import { Injectable, Scope } from '@nestjs/common';
import { PrismaService } from '@infra/database/prisma.service';
import { Food } from '@prisma/client';

@Injectable({ scope: Scope.REQUEST }) // ← REQUEST-scoped: um DataLoader por request
export class FoodsLoader {
  constructor(private readonly prisma: PrismaService) {}

  // O DataLoader agrupa múltiplas chamadas em batch
  readonly loader = new DataLoader<string, Food[]>(
    async (mealIds: readonly string[]) => {
      // Uma única query para todos os mealIds do batch
      const foods = await this.prisma.food.findMany({
        where: { mealId: { in: [...mealIds] } },
      });

      // Mapeia os resultados de volta para cada mealId (ordem deve corresponder)
      return mealIds.map((mealId) =>
        foods.filter((food) => food.mealId === mealId),
      );
    },
  );
}

// Uso no resolver (GraphQL)
@ResolveField('foods', () => [FoodType])
async foods(@Parent() meal: Meal): Promise<Food[]> {
  // O DataLoader agrupa chamadas simultâneas em uma única query
  return this.foodsLoader.loader.load(meal.id);
}
```

> **Regra:** DataLoader **deve ser REQUEST-scoped** (`Scope.REQUEST`). Um DataLoader global entre requisições contamina o cache de uma request com dados de outras.

---

## 2. Estratégia de Índices — A Base da Performance

### 2.1 Quando Criar um Índice

Índices aceleram leituras mas custam espaço em disco e aumentam o tempo de escritas (INSERT, UPDATE, DELETE precisam atualizar os índices). A regra é: **crie índices com base em padrões de acesso reais**, não preventivamente.

#### Tabela de decisão

| Padrão de query | Índice recomendado | Onde declarar |
|----------------|-------------------|---------------|
| `WHERE userId = ?` | Índice simples em `userId` | `@Index()` na coluna (TypeORM) / `@@index([userId])` (Prisma) |
| `WHERE userId = ? ORDER BY createdAt DESC` | Índice composto `(userId, createdAt)` | `@Index(['userId', 'createdAt'])` na Entity |
| `WHERE email = ?` (busca de login) | Índice único em `email` | `unique: true` na coluna |
| `WHERE status = ? AND scheduledAt < ?` | Índice composto `(status, scheduledAt)` | `@Index(['status', 'scheduledAt'])` |
| `LIKE '%termo%'` (busca textual) | Full-text search (índice GIN no PostgreSQL) | Migration SQL explícita |
| FK de relação (`userId` em `meals`) | Índice simples na FK | TypeORM cria automaticamente para `@ManyToOne` |

#### Quando NÃO criar índice
- Colunas com baixa cardinalidade (ex: coluna `status` com 2 valores: `active`/`inactive`) — o índice pode ser ignorado pelo planner do PostgreSQL
- Tabelas muito pequenas (< 1.000 registros) — full table scan é frequentemente mais rápido
- Colunas que só aparecem em `SELECT` (sem `WHERE`, `ORDER BY`, `JOIN`)

### 2.2 TypeORM — Declaração de Índices

> Complementa e aprofunda a seção 5 de `typeorm-patterns.md`.

```typescript
// src/modules/meals/entities/meal.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, Index, ManyToOne } from 'typeorm';

@Entity('meals')
@Index(['userId', 'createdAt'])          // ← composto: queries de histórico por usuário
@Index(['status', 'scheduledAt'])        // ← composto: queries de agendamento
export class MealEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'uuid' })
  @Index()                               // ← simples: busca por userId isolada
  userId: string;                        // TypeORM NÃO cria índice automático em @Column com uuid

  @Column({ type: 'varchar', length: 20, default: 'active' })
  status: string;

  @Column({ type: 'timestamp', nullable: true })
  scheduledAt: Date | null;

  @CreateDateColumn()
  createdAt: Date;
}
```

> **Atenção TypeORM:** O TypeORM cria índice automático apenas para chaves estrangeiras (`@ManyToOne` → `@JoinColumn`). Para outros campos usados em `WHERE`, declare `@Index()` explicitamente.

### 2.3 Prisma — Declaração de Índices no Schema

> Complementa a seção 1 de `prisma-patterns.md`.

```prisma
model Meal {
  id          String   @id @default(uuid())
  userId      String   @map("user_id")
  status      String   @db.VarChar(20) @default("active")
  scheduledAt DateTime? @map("scheduled_at")
  createdAt   DateTime @default(now()) @map("created_at")

  user        User     @relation(fields: [userId], references: [id])

  @@map("meals")
  @@index([userId])                      // ← FK: o Prisma não cria automaticamente
  @@index([userId, createdAt])           // ← composto: queries de histórico
  @@index([status, scheduledAt])         // ← composto: queries de agendamento
}
```

> **Atenção Prisma:** Por padrão, o Prisma **não cria índice automaticamente** para campos de relação (FK). Declare `@@index([fkField])` explicitamente para colunas usadas em `WHERE` de FKs.

### 2.4 Índice Full-Text Search (PostgreSQL)

Para busca textual com `LIKE '%termo%'`, o índice B-Tree tradicional não funciona. Crie um índice GIN via migration:

```typescript
// src/infra/database/migrations/1709900000001-AddFullTextIndexToFoods.ts
// TypeORM Migration
import { MigrationInterface, QueryRunner } from 'typeorm';

export class AddFullTextIndexToFoods1709900000001 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    // Cria índice GIN para busca textual no PostgreSQL
    await queryRunner.query(`
      CREATE INDEX idx_foods_name_fulltext
        ON foods
        USING GIN (to_tsvector('portuguese', name))
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP INDEX IF EXISTS idx_foods_name_fulltext`);
  }
}
```

```typescript
// Uso no Repository (TypeORM)
async searchByName(term: string): Promise<FoodEntity[]> {
  return this.repo
    .createQueryBuilder('food')
    .where(`to_tsvector('portuguese', food.name) @@ plainto_tsquery('portuguese', :term)`, { term })
    .orderBy('food.name', 'ASC')
    .getMany();
}
```

### 2.5 Validando Índices com EXPLAIN ANALYZE

Use `EXPLAIN ANALYZE` para confirmar que o PostgreSQL está usando o índice esperado:

```sql
-- Execute diretamente no banco (Prisma Studio, psql, TablePlus)
EXPLAIN ANALYZE
SELECT * FROM meals
WHERE user_id = 'some-uuid'
ORDER BY created_at DESC
LIMIT 20;

-- Saída ideal: "Index Scan using idx_meals_user_id_created_at"
-- Saída ruim:  "Seq Scan on meals" → índice não está sendo usado
```

```typescript
// Equivalente via TypeORM (use em desenvolvimento, nunca em produção)
const result = await this.dataSource.query(`
  EXPLAIN ANALYZE
  SELECT * FROM meals
  WHERE user_id = $1
  ORDER BY created_at DESC
  LIMIT 20
`, [userId]);
console.log(result); // imprime o plano de execução
```

---

## 3. Query Builder vs Raw SQL — Quando Usar Cada Um

### 3.1 Hierarquia de Escolha

Use a abordagem mais simples que resolve o problema. Mude para a próxima apenas quando a anterior não for suficiente.

```
1. find() / findMany() / findOne()  → Queries simples com WHERE, ORDER BY, LIMIT
2. QueryBuilder / Prisma fluent API → Queries dinâmicas, JOINs complexos, subqueries
3. $queryRaw / query() (raw SQL)    → Funções de banco específicas, WINDOW functions, CTEs
```

### 3.2 TypeORM — Quando Usar Cada Abordagem

```typescript
// ─── NÍVEL 1: find() para queries simples ─────────────────────────────────────
// Use quando: WHERE simples, ORDER BY, LIMIT, sem lógica dinâmica
async findByUserId(userId: string): Promise<MealEntity[]> {
  return this.repo.find({
    where: { userId },
    order: { createdAt: 'DESC' },
    take: 50,
  });
}

// ─── NÍVEL 2: QueryBuilder para filtros dinâmicos e JOINs ─────────────────────
// Use quando: filtros opcionais/dinâmicos, múltiplos JOINs, subqueries, paginação
async findPaginatedWithFilters(params: {
  userId: string;
  status?: string;
  from?: Date;
  to?: Date;
  page: number;
  limit: number;
}): Promise<{ items: MealEntity[]; total: number }> {
  const qb = this.repo
    .createQueryBuilder('meal')
    .where('meal.userId = :userId', { userId: params.userId });

  if (params.status) {
    qb.andWhere('meal.status = :status', { status: params.status });
  }
  if (params.from && params.to) {
    qb.andWhere('meal.createdAt BETWEEN :from AND :to', {
      from: params.from,
      to: params.to,
    });
  }

  const [items, total] = await qb
    .orderBy('meal.createdAt', 'DESC')
    .skip((params.page - 1) * params.limit)
    .take(params.limit)
    .getManyAndCount();  // ← executa SELECT + COUNT em 2 queries otimizadas

  return { items, total };
}

// ─── NÍVEL 3: raw SQL para funções específicas do banco ───────────────────────
// Use quando: WINDOW functions, CTEs, funções específicas de PostgreSQL
async getDailyCalorieSummary(userId: string, days: number): Promise<{
  date: string;
  totalCalories: number;
}[]> {
  return this.dataSource.query(`
    SELECT
      DATE(m.created_at)::text AS date,
      SUM(f.calories * mf.quantity)::int AS "totalCalories"
    FROM meals m
    JOIN meal_foods mf ON mf.meal_id = m.id
    JOIN foods f ON f.id = mf.food_id
    WHERE m.user_id = $1
      AND m.created_at >= NOW() - INTERVAL '${days} days'
    GROUP BY DATE(m.created_at)
    ORDER BY date DESC
  `, [userId]);
}
```

### 3.3 Prisma — Quando Usar Cada Abordagem

```typescript
// ─── NÍVEL 1: findMany / findOne para queries simples ─────────────────────────
async findByUserId(userId: string) {
  return this.prisma.meal.findMany({
    where: { userId, deletedAt: null },
    orderBy: { createdAt: 'desc' },
    take: 50,
  });
}

// ─── NÍVEL 2: queries dinâmicas com Prisma.WhereInput ─────────────────────────
async findPaginatedWithFilters(params: {
  userId: string;
  status?: string;
  from?: Date;
  to?: Date;
  page: number;
  limit: number;
}) {
  const where: Prisma.MealWhereInput = {
    userId: params.userId,
    deletedAt: null,
    ...(params.status && { status: params.status }),
    ...(params.from && params.to && {
      createdAt: { gte: params.from, lte: params.to },
    }),
  };

  const [items, total] = await this.prisma.$transaction([
    this.prisma.meal.findMany({
      where,
      orderBy: { createdAt: 'desc' },
      skip: (params.page - 1) * params.limit,
      take: params.limit,
    }),
    this.prisma.meal.count({ where }),
  ]);

  return { items, total };
}

// ─── NÍVEL 3: $queryRaw para funções específicas do banco ─────────────────────
// Use Prisma.sql para evitar SQL Injection em partes dinâmicas
async getDailyCalorieSummary(userId: string, days: number): Promise<{
  date: string;
  totalCalories: number;
}[]> {
  return this.prisma.$queryRaw<{ date: string; totalCalories: number }[]>`
    SELECT
      DATE(m.created_at)::text AS date,
      SUM(f.calories * mf.quantity)::int AS "totalCalories"
    FROM meals m
    JOIN meal_foods mf ON mf.meal_id = m.id
    JOIN foods f ON f.id = mf.food_id
    WHERE m.user_id = ${userId}
      AND m.created_at >= NOW() - ${Prisma.raw(`INTERVAL '${days} days'`)}
    GROUP BY DATE(m.created_at)
    ORDER BY date DESC
  `;
}
```

> **Regra de segurança com raw SQL:** Sempre use parâmetros (`$1`, `${value}`) para valores dinâmicos — nunca concatene strings. Com Prisma, use template literals (`\`SELECT ... WHERE id = ${id}\``) que são SQL-injection safe. Com TypeORM, use a sintaxe `:param` no QueryBuilder ou `[$1]` nos parâmetros do `query()`.

### 3.4 Quando Raw SQL é Obrigatório

| Caso de uso | Motivo |
|------------|--------|
| `WINDOW functions` (RANK, ROW_NUMBER, LAG) | QueryBuilder não suporta nativamente |
| `CTEs` (WITH clauses) | Melhoram legibilidade de queries complexas |
| `UPSERT` com conflito personalizado (`ON CONFLICT DO UPDATE`) | Comportamento específico do banco |
| `LATERAL JOIN` | TypeORM/Prisma não suportam |
| Funções específicas do PostgreSQL (`to_tsvector`, `json_agg`, `array_agg`) | Não têm abstração ORM equivalente |

---

## 4. Cache de Queries com Redis

### 4.1 Quando Usar Cache

O cache é a solução correta para dados que:
- São **lidos frequentemente e modificados raramente** (ex: lista de alimentos do catálogo)
- São **caros de computar** (relatórios, agregações)
- Podem tolerar **staleness** por um período definido (TTL)

**Não use cache para dados transacionais** (saldo, pedidos em processamento, dados do usuário que mudam com frequência).

### 4.2 Configuração do CacheModule com Redis

> Esta configuração complementa `config-management.md` — adicione `REDIS_URL` ao `env.config.ts`.

```bash
npm install @nestjs/cache-manager cache-manager @keyv/redis
```

```typescript
// src/config/env.config.ts — adicione:
const envSchema = z.object({
  // ... variáveis existentes
  REDIS_URL: z.string().url().optional(), // opcional — permite fallback para in-memory
  CACHE_TTL_SECONDS: z.coerce.number().default(300), // 5 minutos padrão
});
```

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
      imports: [ConfigModule],
      inject: [cacheConfig.KEY],
      useFactory: (config: ConfigType<typeof cacheConfig>) => ({
        store: config.redisUrl
          ? createKeyv(config.redisUrl)   // Redis em produção/staging
          : undefined,                    // in-memory em desenvolvimento (padrão)
        ttl: config.ttlSeconds * 1000,    // cache-manager v5+ usa milissegundos
      }),
      isGlobal: true,
    }),
  ],
})
export class AppCacheModule {}
```

```typescript
// src/app.module.ts — registre antes dos módulos de domínio
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [cacheConfig, ...] }),
    DatabaseModule,
    AppCacheModule,   // ← após DatabaseModule, antes dos módulos de domínio
    // ... módulos de domínio
  ],
})
export class AppModule {}
```

### 4.3 Cache no Repository — Padrão Manual (Recomendado)

O cache manual no Repository dá controle granular sobre TTL e invalidação:

```typescript
// src/modules/foods/repositories/foods.repository.ts
import { Injectable } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Inject } from '@nestjs/common';
import { Cache } from 'cache-manager';
import { PrismaService } from '@infra/database/prisma.service';
import { Food } from '@prisma/client';

@Injectable()
export class FoodsRepository {
  // TTL específico por tipo de dado — alimentos do catálogo mudam raramente
  private readonly CATALOG_TTL_MS = 10 * 60 * 1000; // 10 minutos

  constructor(
    private readonly prisma: PrismaService,
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
  ) {}

  // ─── Leitura com cache ─────────────────────────────────────────────────────

  async findCatalog(): Promise<Food[]> {
    const cacheKey = 'foods:catalog';

    // 1. Verificar cache
    const cached = await this.cache.get<Food[]>(cacheKey);
    if (cached) return cached;

    // 2. Cache miss — buscar no banco
    const foods = await this.prisma.food.findMany({
      where: { isActive: true },
      orderBy: { name: 'asc' },
    });

    // 3. Armazenar no cache com TTL específico
    await this.cache.set(cacheKey, foods, this.CATALOG_TTL_MS);

    return foods;
  }

  // ─── Escrita com invalidação de cache ─────────────────────────────────────

  async createFood(data: CreateFoodData): Promise<Food> {
    const food = await this.prisma.food.create({ data });

    // Invalida o cache do catálogo ao criar novo alimento
    await this.cache.del('foods:catalog');

    return food;
  }

  async updateFood(id: string, data: Partial<CreateFoodData>): Promise<Food> {
    const food = await this.prisma.food.update({ where: { id }, data });

    // Invalida caches afetados pela atualização
    await this.cache.del('foods:catalog');
    await this.cache.del(`foods:${id}`);

    return food;
  }

  // ─── Cache por ID com TTL diferente ───────────────────────────────────────

  async findById(id: string): Promise<Food | null> {
    const cacheKey = `foods:${id}`;

    const cached = await this.cache.get<Food>(cacheKey);
    if (cached) return cached;

    const food = await this.prisma.food.findUnique({ where: { id } });
    if (food) {
      await this.cache.set(cacheKey, food, 5 * 60 * 1000); // 5 minutos
    }

    return food;
  }
}
```

### 4.4 Cache em Endpoints com `CacheInterceptor` (para dados estáticos)

Para endpoints completamente estáticos (ex: lista de categorias), o `CacheInterceptor` do NestJS automatiza o cache da response:

```typescript
// src/modules/categories/controllers/categories.controller.ts
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { CacheInterceptor, CacheTTL, CacheKey } from '@nestjs/cache-manager';

@Controller('categories')
@UseInterceptors(CacheInterceptor)  // ← cacheia a response HTTP automaticamente
export class CategoriesController {
  constructor(private readonly getCategoriesUseCase: GetCategoriesUseCase) {}

  @Get()
  @CacheKey('categories:all')       // ← chave explícita do cache
  @CacheTTL(15 * 60 * 1000)        // ← TTL de 15 minutos (ms)
  async findAll(): Promise<CategoryViewModel[]> {
    const categories = await this.getCategoriesUseCase.execute();
    return CategoryViewModel.fromEntities(categories);
  }
}
```

> **Quando usar `CacheInterceptor` vs cache manual no Repository:**
> - `CacheInterceptor`: dados que **nunca mudam por requisição** (não dependem de `userId`, `role`, etc.) — categorias, configurações globais, tabelas de conversão
> - Cache manual no Repository: dados **parametrizados por usuário/contexto** ou que precisam de invalidação granular

### 4.5 Estratégia de Chaves de Cache

Defina chaves com estrutura hierárquica para facilitar invalidação:

```typescript
// Convenção de chaves — use prefixos por domínio
const CACHE_KEYS = {
  FOODS_CATALOG: 'foods:catalog',          // toda a lista
  FOOD_BY_ID: (id: string) => `foods:${id}`,  // item específico
  MEALS_HISTORY: (userId: string) => `meals:history:${userId}`,
  // ...
} as const;

// Invalidação em grupo — use pattern matching se o cache provider suportar
// Redis: DEL foods:*  → apaga todas as chaves do domínio foods
// cache-manager não tem glob nativo — prefira invalidação explícita por chave
```

---

## 5. Query Logger — Detectando Problemas de Performance

### 5.1 TypeORM — Configuração do Logger

O TypeORM tem sistema de logging nativo. Configure no `DatabaseModule` (cf. `typeorm-patterns.md` seção 4):

```typescript
// src/infra/database/database.module.ts
TypeOrmModule.forRootAsync({
  inject: [databaseConfig.KEY],
  useFactory: (dbConfig: ConfigType<typeof databaseConfig>) => ({
    type: 'postgres',
    url: dbConfig.url,
    // ... outras configurações

    // Logger por ambiente:
    logging: dbConfig.isDevelopment
      ? ['query', 'error', 'warn', 'slow']  // desenvolvimento: tudo
      : ['error', 'warn'],                   // produção: apenas erros

    // Threshold para "slow query" (ms) — loga queries acima deste tempo
    maxQueryExecutionTime: 1000,             // alerta para queries > 1s
  }),
})
```

**O que cada nível loga:**

| Nível | O que loga |
|-------|-----------|
| `'query'` | Toda query executada com o SQL gerado |
| `'error'` | Erros de execução de query |
| `'warn'` | Queries com possíveis problemas |
| `'slow'` | Queries que excedem `maxQueryExecutionTime` |
| `'schema'` | DDL (CREATE TABLE, ALTER, DROP) |
| `'migration'` | Execução de migrations |

### 5.2 TypeORM — Logger Customizado para Produção

Em produção, o logger padrão do TypeORM imprime no console. Crie um logger customizado que integra com o sistema de logs do NestJS:

```typescript
// src/infra/database/typeorm-logger.ts
import { Logger as TypeOrmLogger, QueryRunner } from 'typeorm';
import { Logger } from '@nestjs/common';

export class NestTypeOrmLogger implements TypeOrmLogger {
  private readonly logger = new Logger('TypeORM');

  logQuery(query: string, parameters?: unknown[]): void {
    // Em desenvolvimento: loga todas as queries
    this.logger.debug(`Query: ${query} -- Parameters: ${JSON.stringify(parameters)}`);
  }

  logQueryError(error: string, query: string, parameters?: unknown[]): void {
    this.logger.error(`Query Error: ${error}\nQuery: ${query}\nParameters: ${JSON.stringify(parameters)}`);
  }

  logQuerySlow(time: number, query: string, parameters?: unknown[]): void {
    // Slow queries são SEMPRE logadas — mesmo em produção
    this.logger.warn(`Slow Query (${time}ms): ${query}\nParameters: ${JSON.stringify(parameters)}`);
  }

  logSchemaBuild(message: string): void {
    this.logger.debug(`Schema: ${message}`);
  }

  logMigration(message: string): void {
    this.logger.log(`Migration: ${message}`);
  }

  log(level: 'log' | 'info' | 'warn', message: unknown): void {
    if (level === 'warn') {
      this.logger.warn(String(message));
    } else {
      this.logger.log(String(message));
    }
  }
}
```

```typescript
// src/infra/database/database.module.ts — use o logger customizado
import { NestTypeOrmLogger } from './typeorm-logger';

TypeOrmModule.forRootAsync({
  useFactory: (dbConfig) => ({
    // ...
    logger: new NestTypeOrmLogger(),              // ← logger customizado
    maxQueryExecutionTime: 500,                   // alerta queries > 500ms
    logging: dbConfig.isDevelopment ? true : ['error', 'warn', 'slow'],
  }),
})
```

### 5.3 Prisma — Configuração do Logger

No Prisma, configure o logging no `PrismaService` (cf. `prisma-patterns.md` seção 2):

```typescript
// src/infra/database/prisma.service.ts
import { Injectable, Logger, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient, Prisma } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger('Prisma');

  constructor() {
    super({
      log: [
        { emit: 'event', level: 'query' },   // ← emite evento (não imprime)
        { emit: 'event', level: 'warn' },
        { emit: 'event', level: 'error' },
        { emit: 'stdout', level: 'info' },   // ← imprime info no stdout
      ],
    });

    // Listener para queries — apenas em desenvolvimento
    if (process.env.NODE_ENV === 'development') {
      this.$on('query' as never, (event: Prisma.QueryEvent) => {
        this.logger.debug(
          `Query: ${event.query}\nParams: ${event.params}\nDuration: ${event.duration}ms`,
        );
      });
    }

    // Slow query: SEMPRE (desenvolvimento e produção)
    this.$on('query' as never, (event: Prisma.QueryEvent) => {
      if (event.duration > 500) {
        this.logger.warn(
          `⚠️ Slow Query (${event.duration}ms): ${event.query}\nParams: ${event.params}`,
        );
      }
    });

    this.$on('warn' as never, (event: Prisma.LogEvent) => {
      this.logger.warn(event.message);
    });

    this.$on('error' as never, (event: Prisma.LogEvent) => {
      this.logger.error(event.message);
    });
  }

  async onModuleInit(): Promise<void> {
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

> **Atenção com `$on('query')`:** Em produção com alto tráfego, logar todas as queries (`level: 'query'`) pode impactar performance e gerar volumes imensos de logs. Restrinja ao ambiente de desenvolvimento e use `maxQueryExecutionTime` / threshold manual para slow queries em produção.

### 5.4 Processo de Detecção de Problemas

```
1. Habilitar query logging no ambiente de desenvolvimento
   → logging: ['query', 'slow'] no TypeORM ou emit: 'event' no Prisma

2. Executar o fluxo suspeito (endpoint, use case)
   → Observar o console/logs do NestJS

3. Identificar padrões problemáticos:
   → Queries repetidas com o mesmo pattern e IDs diferentes → N+1
   → Queries lentas (> threshold) → missing index ou query subótima
   → Queries sem uso de índice (via EXPLAIN ANALYZE) → adicionar @Index()

4. Corrigir:
   → N+1: adicionar JOIN / include / DataLoader
   → Query lenta com índice: verificar seletividade do índice
   → Query lenta sem índice: adicionar @Index() ou @@index()
   → Query complexa lenta: considerar raw SQL ou cache

5. Validar:
   → Re-executar o fluxo e comparar contagem e tempo de queries
   → Usar EXPLAIN ANALYZE para confirmar uso do índice
```

---

## 6. Select Parcial — Evitar `SELECT *`

Selecionar apenas as colunas necessárias reduz tráfego de rede, uso de memória e tempo de serialização.

### TypeORM

```typescript
// ❌ SELECT * — traz todas as colunas, mesmo desnecessárias
const user = await this.repo.findOne({ where: { id } });

// ✅ SELECT parcial com select: ['campo1', 'campo2']
const user = await this.repo.findOne({
  where: { id },
  select: ['id', 'email', 'displayName'],  // ← apenas 3 colunas
});

// ✅ Com QueryBuilder — mais legível em queries complexas
const user = await this.repo
  .createQueryBuilder('user')
  .select(['user.id', 'user.email', 'user.displayName'])
  .where('user.id = :id', { id })
  .getOne();
```

### Prisma

```typescript
// O Prisma usa select aninhado — consistente com prisma-patterns.md seção 5.3
const user = await this.prisma.user.findUnique({
  where: { id },
  select: {
    id: true,
    email: true,
    displayName: true,
    // passwordHash: omitido implicitamente
  },
});
```

---

## 7. Anti-padrões de Performance

### ❌ Eager Loading Implícito (TypeORM)

```typescript
// ❌ PROIBIDO — eager: true na relation carrega dados em TODA query
// Documentado em typeorm-patterns.md seção 5, repetido aqui por completude
@OneToMany(() => FoodEntity, (food) => food.meal, { eager: true })
foods: FoodEntity[];   // carregado mesmo quando não é necessário

// ✅ CORRETO — lazy por padrão, JOIN explícito quando necessário
@OneToMany(() => FoodEntity, (food) => food.meal, { lazy: true })
foods: Relation<FoodEntity[]>;
```

### ❌ SELECT * em Tabelas Grandes

```typescript
// ❌ Buscar todos os campos de tabela com muitos campos/registros
const allMeals = await this.repo.find(); // SELECT * FROM meals — sem filtro!

// ✅ Sempre filtre e pagine — nunca busque toda a tabela
const meals = await this.repo.find({
  where: { userId },
  take: 50,
  skip: 0,
  select: ['id', 'blocks', 'createdAt'],
});
```

### ❌ N queries dentro de loop

```typescript
// ❌ Loop com query por iteração — N+1 explícito
const meals = await this.repo.find({ where: { userId } });
const results = [];
for (const meal of meals) {
  const foods = await this.foodsRepo.findByMealId(meal.id); // ← N queries!
  results.push({ meal, foods });
}

// ✅ Buscar todos os foods relacionados em uma query com IN
const mealIds = meals.map((m) => m.id);
const allFoods = await this.foodsRepo.findByMealIds(mealIds); // ← 1 query com WHERE mealId IN (...)
// Então agrupar em memória
```

### ❌ Cache sem TTL ou com TTL infinito

```typescript
// ❌ Cache sem expiração — memória vaza, dados nunca são atualizados
await this.cache.set('foods:catalog', foods); // TTL = infinity

// ✅ Sempre defina TTL baseado na volatilidade dos dados
await this.cache.set('foods:catalog', foods, 10 * 60 * 1000); // 10 minutos
```

### ❌ Invalidação de cache após await de operação externa

```typescript
// ❌ Cache não invalidado quando update falha — inconsistência
async updateFood(id: string, data: UpdateFoodData) {
  await this.cache.del(`foods:${id}`); // ← invalida ANTES de confirmar update
  const food = await this.prisma.food.update({ where: { id }, data });
  // Se update falhar, cache foi invalidado mas dado não foi atualizado
  return food;
}

// ✅ Invalide o cache APÓS confirmar a escrita no banco
async updateFood(id: string, data: UpdateFoodData) {
  const food = await this.prisma.food.update({ where: { id }, data }); // ← primeiro
  await this.cache.del(`foods:${id}`);  // ← depois, apenas se update sucedeu
  return food;
}
```

---

## 8. Checklist de Validação

### N+1
- [ ] O query logger foi habilitado para verificar as queries geradas?
- [ ] Relações são carregadas com JOIN explícito (nunca `eager: true`)?
- [ ] Em GraphQL, DataLoader está configurado para todas as relações em resolvers?
- [ ] Loops com queries individuais foram substituídos por queries com `IN (...)`?

### Índices
- [ ] Colunas usadas em `WHERE` e `ORDER BY` frequentes têm índice?
- [ ] FKs em tabelas Prisma têm `@@index([fkField])` declarado?
- [ ] `EXPLAIN ANALYZE` confirmou uso do índice em queries críticas?
- [ ] Colunas com baixa cardinalidade **não** têm índice desnecessário?

### Query Builder vs Raw SQL
- [ ] Queries simples usam `find()` / `findMany()` (não QueryBuilder desnecessariamente)?
- [ ] Filtros dinâmicos usam QueryBuilder / `Prisma.WhereInput`?
- [ ] Raw SQL (`query()` / `$queryRaw`) é usado apenas quando ORM não suporta?
- [ ] Nenhum valor dinâmico é concatenado em string SQL (SQL injection)?

### Cache
- [ ] Cache é usado apenas para dados lidos frequentemente e raramente modificados?
- [ ] Todo `cache.set` tem TTL definido explicitamente?
- [ ] Cache é invalidado APÓS operações bem-sucedidas de escrita (nunca antes)?
- [ ] `CacheInterceptor` é usado apenas para endpoints sem contexto de usuário?

### Query Logger
- [ ] `maxQueryExecutionTime` está configurado (TypeORM) ou threshold manual (Prisma)?
- [ ] Slow queries disparam `logger.warn` em todos os ambientes?
- [ ] Log de todas as queries (`level: 'query'`) está restrito ao ambiente de desenvolvimento?

---

## Referências

- [typeorm-patterns.md](.agent/skills/backend/database/typeorm-patterns.md) — Seção 5: Índices e QueryBuilder, regra `eager: false`
- [prisma-patterns.md](.agent/skills/backend/database/prisma-patterns.md) — Seção 5: Tipagem segura e `$queryRaw`, seção 1: `@@index` no schema
- [database-transactions.md](.agent/skills/backend/database/database-transactions.md) — Custo e isolation level de transações
- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — Variáveis de ambiente (`REDIS_URL`, `CACHE_TTL_SECONDS`) e namespaces
- [graphql-pattern.md](.agent/skills/backend/api-and-contracts/graphql-pattern.md) — DataLoader para N+1 em GraphQL
- [NestJS Docs — Caching](https://docs.nestjs.com/techniques/caching)
- [TypeORM Docs — Logging](https://typeorm.io/logging)
- [Prisma Docs — Logging](https://www.prisma.io/docs/concepts/components/prisma-client/working-with-prismaclient/logging)
- [PostgreSQL Docs — EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
