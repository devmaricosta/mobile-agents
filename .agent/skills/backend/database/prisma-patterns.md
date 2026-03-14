---
name: prisma-patterns
description: Alternativa ao TypeORM com Prisma no NestJS — estrutura do schema.prisma, PrismaService como singleton global, migrations com prisma migrate, padrão de repositório sobre o Prisma Client, e como tipar queries complexas com segurança usando os tipos gerados automaticamente (Prisma.UserWhereInput, Prisma.validator, $queryRaw tipado).
---

# Prisma Patterns — Banco de Dados no NestJS

Você é um Engenheiro de Software Senior especialista em Prisma e NestJS. Esta skill é a **alternativa à `typeorm-patterns.md`** para projetos que optam pelo Prisma ORM. Ela complementa `clean-architecture.md` (regras de camada) e `nest-project-structure.md` (estrutura de pastas). O foco é **como usar o Prisma corretamente** dentro da arquitetura já estabelecida.

> **Regra absoluta:** Models do Prisma são detalhes de infraestrutura. Eles **nunca** saem da camada de repositório em direção às camadas superiores (Use Cases, Controllers, API responses).

> **Escolha exclusiva:** Um projeto usa **TypeORM OU Prisma**, nunca os dois. Se o projeto já tem `typeorm-patterns.md` aplicado, não misture Prisma. Decida no início do projeto.

---

## 1. Schema Prisma — Anatomia do `schema.prisma`

O `schema.prisma` é a **única fonte de verdade** do schema do banco no Prisma. Tudo — models, tipos, relações, índices, enums — é declarado aqui. O Prisma Client é gerado a partir dele.

### Localização

```
prisma/
├── schema.prisma       ← schema único do banco
└── migrations/         ← pasta gerenciada pelo prisma migrate
    ├── 20240101000000_init/
    │   └── migration.sql
    └── 20240215000000_add_display_name_to_users/
        └── migration.sql
```

> **Nota:** A pasta `prisma/` fica na raiz do projeto, **fora** de `src/`. As migrations são geradas automaticamente pelo CLI — nunca edite os arquivos `.sql` manualmente.

### Anatomia de um Model

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── Enums ───────────────────────────────────────────────────────────────────

enum UserStatus {
  active
  inactive
  suspended
}

// ─── Models ──────────────────────────────────────────────────────────────────

model User {
  id          String     @id @default(uuid())
  email       String     @unique
  passwordHash String    @map("password_hash")   // camelCase no código, snake_case no banco
  displayName String?    @map("display_name")    // ? = opcional (nullable)
  status      UserStatus @default(active)
  createdAt   DateTime   @default(now()) @map("created_at")
  updatedAt   DateTime   @updatedAt @map("updated_at")
  deletedAt   DateTime?  @map("deleted_at")      // soft delete

  // Relações
  meals       Meal[]

  // Índices e mapeamento de tabela
  @@map("users")                                 // nome da tabela em snake_case plural
  @@index([status, createdAt])                   // índice composto
}

model Meal {
  id             String   @id @default(uuid())
  userId         String   @map("user_id")
  blocks         Int
  mode           String   @db.VarChar(20)
  targetCalories Int?     @map("target_calories")
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")

  // Relação — sempre declare os dois lados
  user           User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("meals")
  @@index([userId, createdAt])                   // índice composto — queries de histórico
}
```

### Convenções do Schema

| Regra | Detalhe |
|-------|---------|
| **Campos em `camelCase`** | Código TypeScript usa `camelCase`; `@map("snake_case")` para o banco |
| **Tabelas em `snake_case` plural** | `@@map("users")`, `@@map("meals")` |
| **`@id @default(uuid())`** | UUID como chave primária — consistente com `typeorm-patterns.md` |
| **`@updatedAt`** | Prisma atualiza automaticamente ao fazer `update` |
| **Soft delete via `deletedAt`** | Campo `DateTime?` — `null` = ativo, `Date` = deletado |
| **Enums no schema** | Declare enums no `schema.prisma` — Prisma gera os tipos TypeScript automaticamente |
| **`onDelete: Cascade`** | Declare explicitamente o comportamento de deleção nas relações |
| **Índices com `@@index`** | Para queries frequentes — análogo ao `@Index()` do TypeORM |

### ❌ Anti-padrões no Schema

```prisma
// ❌ ERRADO — nome da tabela sem @@map (banco fica com "User" em PascalCase)
model User {
  id String @id @default(uuid())
}

// ✅ CORRETO
model User {
  id String @id @default(uuid())
  @@map("users")
}

// ❌ ERRADO — campo sem @map (banco fica com "passwordHash" em camelCase)
model User {
  passwordHash String
}

// ✅ CORRETO
model User {
  passwordHash String @map("password_hash")
}
```

---

## 2. PrismaService — O Singleton do Cliente

O `PrismaService` é o coração da integração Prisma + NestJS. É uma classe `@Injectable()` que estende o `PrismaClient` e gerencia o ciclo de vida da conexão.

### Localização

```
src/infra/database/
├── database.module.ts     ← registra e exporta PrismaService globalmente
└── prisma.service.ts      ← singleton do Prisma Client
```

### Implementação do PrismaService

```typescript
// src/infra/database/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor() {
    super({
      log: process.env.NODE_ENV === 'development'
        ? ['query', 'warn', 'error']
        : ['error'],
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

> **Por que `extends PrismaClient`?** Diferente do TypeORM que usa `@InjectRepository()`, o Prisma Client é usado diretamente. Estender `PrismaClient` no service permite que o NestJS gerencie o ciclo de vida da conexão e torna o `PrismaService` testável via mock.

### DatabaseModule (Global)

```typescript
// src/infra/database/database.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()          // ← disponível em todos os módulos sem importar DatabaseModule
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class DatabaseModule {}
```

> **`@Global()` é obrigatório para o PrismaService.** Diferente do TypeORM onde cada módulo importa `TypeOrmModule.forFeature([Entity])`, o Prisma não tem esse conceito — o mesmo `PrismaService` serve todos os repositórios.

### Registro no AppModule

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { DatabaseModule } from '@infra/database/database.module';
import { AuthModule } from '@modules/auth/auth.module';
import { MealsModule } from '@modules/meals/meals.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    DatabaseModule,    // ← registra PrismaService globalmente — sempre antes dos módulos de domínio
    AuthModule,
    MealsModule,
  ],
})
export class AppModule {}
```

### Variável de ambiente

O Prisma usa uma única string de conexão `DATABASE_URL`. Atualize o `env.config.ts` (cf. `config-management.md`):

```typescript
// src/config/env.config.ts — adicione DATABASE_URL
const envSchema = z.object({
  // ... outros campos
  DATABASE_URL: z.string().url(),   // ← string de conexão completa (postgres://user:pass@host:5432/db)
});

// .env.example
// DATABASE_URL="postgresql://postgres:password@localhost:5432/myapp_dev?schema=public"
```

> **Diferença do TypeORM:** A `config-management.md` usa variáveis separadas (`DATABASE_HOST`, `DATABASE_PORT`, etc.) para o TypeORM. Com Prisma, use **apenas `DATABASE_URL`** — o Prisma não consome variáveis separadas nativamente.

---

## 3. Migrations — Versionamento do Schema

O Prisma tem seu próprio sistema de migrations — nunca use `db push` em produção.

### Fluxo de Development

```bash
# 1. Após alterar o schema.prisma, gere a migration
npx prisma migrate dev --name add-display-name-to-users

# 2. O Prisma irá:
#    - Gerar o arquivo SQL em prisma/migrations/<timestamp>_<name>/migration.sql
#    - Aplicar a migration no banco de desenvolvimento
#    - Regenerar o Prisma Client (equivalente ao prisma generate)

# 3. Inspecionar o SQL gerado (sempre revise antes de commitar)
cat prisma/migrations/20240215000000_add_display_name_to_users/migration.sql

# 4. Verificar status das migrations
npx prisma migrate status
```

### Fluxo de Produção / CI

```bash
# Em produção, NUNCA use migrate dev — use migrate deploy
# migrate deploy NÃO gera novas migrations, apenas aplica as pendentes
npx prisma migrate deploy

# Regenerar o cliente após deploy (geralmente no build step do CI)
npx prisma generate
```

### Scripts no `package.json`

```json
{
  "scripts": {
    "prisma:migrate:dev":    "prisma migrate dev",
    "prisma:migrate:deploy": "prisma migrate deploy",
    "prisma:migrate:reset":  "prisma migrate reset",
    "prisma:migrate:status": "prisma migrate status",
    "prisma:generate":       "prisma generate",
    "prisma:studio":         "prisma studio",
    "prisma:seed":           "ts-node prisma/seed.ts"
  }
}
```

### Convenções de Migrations

| Regra | Detalhe |
|-------|---------|
| **Nunca edite `.sql` manualmente** | Recrie a migration com `migrate dev` se necessário |
| **Nome descritivo no `--name`** | `add-display-name-to-users`, `create-meals-table` |
| **Comite as migrations** | A pasta `prisma/migrations/` deve estar no git |
| **`migrate deploy` no CI/CD** | Nunca `migrate dev` em ambientes não-locais |
| **`prisma generate` no build** | Regenerar o cliente após deploy para refletir o schema atual |
| **Nunca use `db push` em produção** | `db push` ignora o sistema de migrations — apenas para prototipagem |

---

## 4. Padrão de Repositório — A Única Abstração de Banco

O padrão é idêntico ao do TypeORM (`typeorm-patterns.md`): **classes `@Injectable()` que encapsulam o `PrismaService`**. É o único lugar onde o Prisma é usado diretamente.

### Estrutura de um Repository

```typescript
// src/modules/users/repositories/users.repository.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '@infra/database/prisma.service';
import { User, Prisma } from '@prisma/client';

// Tipos locais de input — desacopla o Repository dos DTOs da API
type CreateUserData = {
  email: string;
  passwordHash: string;
  displayName?: string;
};

type FindUsersFilter = {
  status?: 'active' | 'inactive' | 'suspended';
  createdAfter?: Date;
};

@Injectable()
export class UsersRepository {
  constructor(private readonly prisma: PrismaService) {}

  // ─── CREATE ────────────────────────────────────────────────────────────────

  async create(data: CreateUserData): Promise<User> {
    return this.prisma.user.create({ data });
  }

  // ─── READ ──────────────────────────────────────────────────────────────────

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async findByFilter(filter: FindUsersFilter): Promise<User[]> {
    const where: Prisma.UserWhereInput = {
      deletedAt: null,                                        // sempre exclui soft-deletados
    };

    if (filter.status) {
      where.status = filter.status;
    }
    if (filter.createdAfter) {
      where.createdAt = { gte: filter.createdAfter };
    }

    return this.prisma.user.findMany({
      where,
      orderBy: { createdAt: 'desc' },
    });
  }

  // ─── UPDATE ────────────────────────────────────────────────────────────────

  async updateById(id: string, data: Partial<CreateUserData>): Promise<User | null> {
    try {
      return await this.prisma.user.update({
        where: { id },
        data,
      });
    } catch (error) {
      // Prisma lança P2025 quando o registro não é encontrado
      if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === 'P2025') {
        return null;
      }
      throw error;
    }
  }

  // ─── DELETE ────────────────────────────────────────────────────────────────

  async deleteById(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } });
  }

  async softDeleteById(id: string): Promise<void> {
    await this.prisma.user.update({
      where: { id },
      data: { deletedAt: new Date() },
    });
  }
}
```

### Declaração no Módulo

```typescript
// src/modules/users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersRepository } from './repositories/users.repository';
import { GetUserUseCase } from './use-cases/get-user/get-user.use-case';
import { UsersController } from './controllers/users.controller';

@Module({
  // Sem TypeOrmModule.forFeature() — PrismaService já é global via DatabaseModule
  controllers: [UsersController],
  providers: [GetUserUseCase, UsersRepository],
  exports: [GetUserUseCase],   // exporta apenas o contrato público
})
export class UsersModule {}
```

> **Diferença chave do TypeORM:** Módulos com Prisma **não precisam** de `TypeOrmModule.forFeature([Entity])`. O `PrismaService` já está disponível globalmente via `DatabaseModule` com `@Global()`.

### Regras do Repository

| Regra | Justificativa |
|-------|---------------|
| **Injeta apenas `PrismaService`** | Nunca o `PrismaClient` diretamente — permite mock em testes |
| **Retorna sempre tipo do modelo Prisma** | `User`, `Meal` — tipos gerados automaticamente pelo `prisma generate` |
| **Nunca recebe DTO como parâmetro** | Crie tipos locais (`CreateUserData`) para desacoplar de DTOs da API |
| **Nunca contém lógica de negócio** | `if (user.age < 18) throw` → pertence ao UseCase |
| **Trata `PrismaClientKnownRequestError`** | Código `P2025` = registro não encontrado — retorne `null` |
| **Sempre filtra `deletedAt: null`** | Em projetos com soft delete, toda query deve excluir registros deletados explicitamente |

---

## 5. Tipagem Segura em Queries Complexas

O Prisma gera um namespace completo de tipos a partir do schema. Use-os para garantir segurança em queries dinâmicas.

### 5.1 Tipos Gerados para Filtros e Ordenação

```typescript
import { Prisma } from '@prisma/client';

// ✅ Use Prisma.ModelWhereInput para filtros tipados
async findPaginated(params: {
  page: number;
  limit: number;
  where?: Prisma.MealWhereInput;       // ← tipo gerado, seguro em tempo de compilação
  orderBy?: Prisma.MealOrderByWithRelationInput;  // ← também gerado
}): Promise<{ items: Meal[]; total: number }> {
  const { page, limit, where, orderBy } = params;

  const [items, total] = await this.prisma.$transaction([
    this.prisma.meal.findMany({
      where: { deletedAt: null, ...where },
      orderBy: orderBy ?? { createdAt: 'desc' },
      skip: (page - 1) * limit,
      take: limit,
    }),
    this.prisma.meal.count({
      where: { deletedAt: null, ...where },
    }),
  ]);

  return { items, total };
}
```

### 5.2 `Prisma.validator` — Objetos Reutilizáveis e Tipados

Use `Prisma.validator` para criar objetos de query reutilizáveis com tipagem validada em compile-time:

```typescript
import { Prisma } from '@prisma/client';

// Define uma seleção reutilizável com tipagem garantida pelo Prisma
const userWithMeals = Prisma.validator<Prisma.UserDefaultArgs>()({
  include: { meals: true },
});

// O tipo é inferido automaticamente a partir do validator
type UserWithMeals = Prisma.UserGetPayload<typeof userWithMeals>;

// Uso no repository
async findWithMeals(id: string): Promise<UserWithMeals | null> {
  return this.prisma.user.findUnique({
    where: { id },
    ...userWithMeals,              // ← spread do validator tipado
  });
}
```

### 5.3 `Prisma.UserSelect` — Seleção Parcial de Campos

```typescript
import { Prisma, User } from '@prisma/client';

// Define uma seleção reutilizável que omite campos sensíveis
const publicUserSelect = {
  id: true,
  email: true,
  displayName: true,
  status: true,
  createdAt: true,
  // passwordHash: false ← omitido implicitamente (campo ausente = não selecionado)
} satisfies Prisma.UserSelect;  // ← satisfies garante que os campos existem no model

// Tipo inferido a partir da seleção
type PublicUser = Prisma.UserGetPayload<{ select: typeof publicUserSelect }>;

async findPublicById(id: string): Promise<PublicUser | null> {
  return this.prisma.user.findUnique({
    where: { id },
    select: publicUserSelect,    // ← retorna apenas os campos seguros
  });
}
```

> **`satisfies` vs `as`:** Use `satisfies Prisma.UserSelect` — o TypeScript valida que os campos do objeto existem no modelo sem perder a inferência de tipo. Nunca use `as Prisma.UserSelect` (silencia erros de tipo).

### 5.4 `$queryRaw` Tipado — Queries SQL Brutas

Quando o Prisma Client não for suficiente, use `$queryRaw` com tipagem explícita:

```typescript
import { User, Prisma } from '@prisma/client';

// ✅ $queryRaw com tipo explícito
async findActiveUsersCreatedThisMonth(): Promise<User[]> {
  return this.prisma.$queryRaw<User[]>`
    SELECT id, email, display_name, status, created_at
    FROM users
    WHERE status = 'active'
      AND deleted_at IS NULL
      AND DATE_TRUNC('month', created_at) = DATE_TRUNC('month', NOW())
    ORDER BY created_at DESC
  `;
}

// ✅ Query condicional com Prisma.sql (evita concatenação de strings — SQL injection safe)
async findByStatusAndDate(status?: string, afterDate?: Date): Promise<User[]> {
  return this.prisma.$queryRaw<User[]>`
    SELECT id, email, status, created_at
    FROM users
    WHERE deleted_at IS NULL
    ${status ? Prisma.sql`AND status = ${status}` : Prisma.empty}
    ${afterDate ? Prisma.sql`AND created_at > ${afterDate}` : Prisma.empty}
    ORDER BY created_at DESC
  `;
}
```

> **Atenção com `$queryRaw`:** O tipo passado em `$queryRaw<T>` é uma **asserção** — o Prisma não valida se os campos retornados pelo SQL correspondem ao tipo `T`. Prefira sempre o Prisma Client tipado; use `$queryRaw` apenas quando necessário.

### 5.5 Queries com Relações — `include` vs `select`

```typescript
// ✅ include — carrega a relação inteira
const mealWithUser = await this.prisma.meal.findUnique({
  where: { id },
  include: { user: true },    // inclui todos os campos de User
});

// ✅ select aninhado — mais eficiente, seleciona apenas o necessário
const mealWithUserEmail = await this.prisma.meal.findUnique({
  where: { id },
  select: {
    id: true,
    blocks: true,
    user: {
      select: { email: true, displayName: true },   // apenas 2 campos do User
    },
  },
});

// ❌ ERRADO — include e select no mesmo nível NÃO funciona no Prisma
const invalid = await this.prisma.meal.findUnique({
  where: { id },
  select: { id: true },
  include: { user: true },   // ← TypeScript Error: não é permitido combinar na raiz
});
```

---

## 6. Soft Delete — Exclusão Lógica

O Prisma não tem suporte nativo a soft delete como decorator. A estratégia é convencional — campo `deletedAt` + filtro manual em todas as queries.

### Schema

```prisma
model User {
  // ...
  deletedAt DateTime? @map("deleted_at")    // null = ativo, Date = deletado logicamente

  @@map("users")
}
```

### Repository com Soft Delete

```typescript
// Soft delete — preenche deletedAt
async softDeleteById(id: string): Promise<void> {
  await this.prisma.user.update({
    where: { id },
    data: { deletedAt: new Date() },
  });
}

// Restore — limpa deletedAt
async restoreById(id: string): Promise<void> {
  await this.prisma.user.update({
    where: { id },
    data: { deletedAt: null },
  });
}

// findById respeitando soft delete
async findById(id: string): Promise<User | null> {
  return this.prisma.user.findFirst({
    where: {
      id,
      deletedAt: null,      // ← obrigatório — exclui soft-deletados
    },
  });
}

// Para incluir soft-deletados (auditoria/admin)
async findByIdIncludingDeleted(id: string): Promise<User | null> {
  return this.prisma.user.findUnique({ where: { id } }); // sem filtro deletedAt
}
```

> **Regra:** Toda query de leitura em um modelo com soft delete **deve** incluir `deletedAt: null` no `where`. Use `findFirst` em vez de `findUnique` quando combinar `deletedAt: null` com outros filtros (o `findUnique` aceita apenas campos marcados com `@unique` ou `@id`).

---

## 7. Transações — Operações Atômicas

O Prisma oferece duas formas de transações: **batch (array)** e **interativa (callback)**.

### Transação Batch — Múltiplas operações simples

```typescript
// ✅ Batch transaction — use quando as operações são independentes e simples
async createUserWithProfile(data: {
  email: string;
  passwordHash: string;
  displayName: string;
}): Promise<User> {
  const [user] = await this.prisma.$transaction([
    this.prisma.user.create({
      data: {
        email: data.email,
        passwordHash: data.passwordHash,
      },
    }),
    this.prisma.profile.create({
      data: {
        // ..., userId será preenchido após o user ser criado
        // ⚠️ Limitação: na batch transaction, não há acesso ao resultado das outras queries
      },
    }),
  ]);

  return user;
}
```

### Transação Interativa — Operações dependentes

```typescript
// ✅ Transação interativa com callback — use quando uma operação depende do resultado de outra
async createUserWithProfile(data: {
  email: string;
  passwordHash: string;
  displayName: string;
}): Promise<User> {
  return this.prisma.$transaction(async (tx) => {
    // tx é um PrismaClient transacional — use-o para todas as operações dentro da transação
    const user = await tx.user.create({
      data: {
        email: data.email,
        passwordHash: data.passwordHash,
      },
    });

    await tx.profile.create({
      data: {
        userId: user.id,          // ← usa o id do user criado na mesma transação
        displayName: data.displayName,
      },
    });

    return user;
  });
}
```

> **Regras de Transações:**
> - Use **batch** (`$transaction([...])`) para operações independentes simultâneas (ex: `findMany` + `count` para paginação)
> - Use **interativa** (`$transaction(async (tx) => {...})`) quando uma operação depende do resultado de outra
> - Transações **sempre ficam no Repository**, nunca no UseCase
> - Dentro do callback interativo, **sempre use `tx`** (o client transacional), nunca `this.prisma`

---

## 8. Configuração e Testes

### Installação

```bash
npm install prisma @prisma/client
npx prisma init    # cria prisma/schema.prisma e .env
```

### Testando com Mock do PrismaService

Como os Use Cases injetam o **Repository** (não o PrismaService diretamente), o mock é feito no Repository — idêntico ao padrão do TypeORM:

```typescript
// src/modules/users/__tests__/get-user.use-case.spec.ts
import { Test } from '@nestjs/testing';
import { GetUserUseCase } from '../use-cases/get-user/get-user.use-case';
import { UsersRepository } from '../repositories/users.repository';

describe('GetUserUseCase', () => {
  let useCase: GetUserUseCase;
  let usersRepository: jest.Mocked<UsersRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        GetUserUseCase,
        {
          provide: UsersRepository,
          useValue: {
            findById: jest.fn(),  // mock do repository — sem acoplamento ao Prisma
          },
        },
      ],
    }).compile();

    useCase = module.get(GetUserUseCase);
    usersRepository = module.get(UsersRepository);
  });

  it('deve retornar null quando o usuário não existe', async () => {
    usersRepository.findById.mockResolvedValue(null);
    const result = await useCase.execute({ id: 'non-existent-id' });
    expect(result).toBeNull();
  });
});
```

### Testando o Repository com PrismaService Mockado

Para testar o próprio Repository, mock o `PrismaService`:

```typescript
// Crie um factory helper reutilizável
export const mockPrismaService = () => ({
  user: {
    create: jest.fn(),
    findUnique: jest.fn(),
    findFirst: jest.fn(),
    findMany: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
    count: jest.fn(),
  },
  meal: {
    create: jest.fn(),
    findMany: jest.fn(),
  },
  $transaction: jest.fn(),
});

// No spec do repository
const module = await Test.createTestingModule({
  providers: [
    UsersRepository,
    { provide: PrismaService, useFactory: mockPrismaService },
  ],
}).compile();
```

---

## 9. Checklist de Validação

### Schema
- [ ] Todos os models têm `@@map("nome_tabela")` em `snake_case` plural?
- [ ] Todos os campos `camelCase` têm `@map("snake_case")`?
- [ ] Chave primária usa `@id @default(uuid())`?
- [ ] Relações declaradas nos dois lados do modelo?
- [ ] `@@index` criado para colunas usadas em `WHERE` e `ORDER BY` frequentes?
- [ ] Soft delete com `deletedAt DateTime?` declarado quando necessário?

### PrismaService e DatabaseModule
- [ ] `PrismaService` implementa `OnModuleInit` e `OnModuleDestroy`?
- [ ] `DatabaseModule` usa `@Global()` e exporta `PrismaService`?
- [ ] `DatabaseModule` registrado **antes** dos módulos de domínio no `AppModule`?

### Migrations
- [ ] Migrations geradas com `prisma migrate dev --name <descricao>`?
- [ ] Pasta `prisma/migrations/` está no git?
- [ ] `prisma migrate deploy` configurado no CI/CD (nunca `migrate dev`)?
- [ ] `prisma generate` roda no step de build do CI?
- [ ] `db push` nunca usado em produção?

### Repository
- [ ] Injeta apenas `PrismaService` (não `PrismaClient` diretamente)?
- [ ] Todos os métodos retornam tipo gerado pelo Prisma (`User`, `Meal`) ou `null`?
- [ ] Tipos locais criados para inputs (`CreateUserData`) — sem receber DTO diretamente?
- [ ] Toda query de leitura inclui `deletedAt: null` em modelos com soft delete?
- [ ] `Prisma.PrismaClientKnownRequestError` com código `P2025` tratado em `update`?
- [ ] Módulo **não** importa `TypeOrmModule.forFeature()` (incompatível com Prisma)?

### Tipagem
- [ ] Filtros dinâmicos usam `Prisma.ModelWhereInput`?
- [ ] Seleções parciais usam `satisfies Prisma.ModelSelect` (não `as`)?
- [ ] `$queryRaw` tem tipo explícito `$queryRaw<Model[]>`?
- [ ] Queries condicionais em `$queryRaw` usam `Prisma.sql`/`Prisma.empty` (não concatenação de strings)?

### Exposição de Dados
- [ ] Model Prisma nunca retornado diretamente em response HTTP?
- [ ] Controller sempre converte Model → ViewModel antes de retornar?
- [ ] `UsersRepository` **não** exportado no `exports` do módulo (a não ser que estritamente necessário)?

---

## Referências

- [Prisma Docs — NestJS Integration](https://www.prisma.io/docs/guides/frameworks/nestjs)
- [Prisma Docs — Prisma Schema Reference](https://www.prisma.io/docs/orm/reference/prisma-schema-reference)
- [Prisma Docs — Migrations](https://www.prisma.io/docs/orm/prisma-migrate)
- [Prisma Docs — Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions)
- [Prisma Docs — Raw Queries](https://www.prisma.io/docs/orm/prisma-client/using-raw-sql/raw-queries)
- [Prisma Docs — Relation Queries](https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries)
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Regras de camadas, onde o Model Prisma vive e o papel do Repository
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura de pastas, use-cases e convenções de nomenclatura
- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — Validação de `DATABASE_URL` com Zod e `env.config.ts`
- [typeorm-patterns.md](.agent/skills/backend/database/typeorm-patterns.md) — Alternativa com TypeORM — use um OU outro por projeto
