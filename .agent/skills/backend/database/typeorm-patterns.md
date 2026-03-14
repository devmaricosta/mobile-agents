---
name: typeorm-patterns
description: Padrões TypeORM no NestJS — entidades como classes puras sem lógica de negócio, repositories customizados como única abstração de acesso ao banco, migrations com versionamento, índices e otimizações de query, e a regra absoluta de nunca vazar entidades ORM para fora da camada de repositório.
---

# TypeORM Patterns — Banco de Dados no NestJS

Você é um Engenheiro de Software Senior especialista em TypeORM e NestJS. Esta skill complementa `clean-architecture.md` (regras de camada) e `nest-project-structure.md` (estrutura de pastas). Aqui o foco é **como usar TypeORM corretamente** dentro da arquitetura já estabelecida.

> **Regra absoluta:** Entidades TypeORM são detalhes de infraestrutura. Elas **nunca** saem da camada de repositório em direção às camadas superiores (Use Cases, Controllers, API responses).

---

## 1. Entidades — Classes Puras de Mapeamento

Entidades representam o schema do banco de dados. Sua única responsabilidade é **mapear colunas e relações** via decorators TypeORM. Nenhuma lógica de negócio.

### Anatomia de uma Entity

```typescript
// src/modules/users/entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  Index,
  OneToMany,
  Relation,
} from 'typeorm';

@Entity('users')
@Index(['email'], { unique: true })        // índice na coluna isolada
@Index(['createdAt', 'status'])            // índice composto para queries frequentes
export class UserEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255, unique: true })
  email: string;

  @Column({ type: 'varchar', length: 255 })
  passwordHash: string;                    // nome deixa claro que é hash

  @Column({ type: 'varchar', length: 20, default: 'active' })
  status: string;

  @Column({ type: 'varchar', length: 100, nullable: true })
  displayName: string | null;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @OneToMany(() => MealEntity, (meal) => meal.user, { lazy: true })
  meals: Relation<MealEntity[]>;           // use Relation<T> para evitar problemas circulares
}
```

### Convenções de Entity

| Regra | Detalhe |
|-------|---------|
| **Nome da classe** | `PascalCase` + sufixo `Entity` — ex: `UserEntity`, `MealEntity` |
| **Nome da tabela** | `snake_case` plural no `@Entity('nome_tabela')` — ex: `@Entity('users')` |
| **Nomes de colunas** | TypeORM converte `camelCase` → `snake_case` automaticamente com `namingStrategy` configurado |
| **Sem lógica de negócio** | Apenas getters simples são permitidos; cálculos e validações vão no UseCase |
| **Sem métodos de domínio** | `user.activate()`, `user.calculateAge()` — PROIBIDO na Entity |
| **Tipo explícito em `@Column`** | Sempre especifique `type` explicitamente para evitar surpresas por banco |
| **Arquivo de relações com `Relation<T>`** | Sempre envolva relações em `Relation<T>` para evitar circular imports TypeScript |

### ❌ Anti-padrões em Entities

```typescript
// ❌ ERRADO — Lógica de negócio na Entity
@Entity('users')
export class UserEntity {
  @Column()
  passwordHash: string;

  // Nunca coloque lógica de negócio na entity
  async validatePassword(raw: string): Promise<boolean> {  // ← PROIBIDO
    return bcrypt.compare(raw, this.passwordHash);         // pertence ao UseCase
  }

  getFullName(): string {                                  // ← PROIBIDO se tiver lógica
    return `${this.firstName} ${this.lastName}`.trim();    // getter puro é tolerável
  }

  @BeforeInsert()                                          // ← PROIBIDO — hooks de ciclo de vida
  async hashPasswordBeforeInsert() {                       // oculta efeitos colaterais
    this.passwordHash = await bcrypt.hash(this.password, 10);
  }
}

// ✅ CORRETO — Entity como mapeamento puro
@Entity('users')
export class UserEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255 })
  passwordHash: string;                    // só o dado, sem comportamento
}
```

---

## 2. Repositories Customizados — A Única Abstração de Banco

O padrão estabelecido neste projeto usa **repositories customizados como classes `@Injectable()`** que encapsulam o `Repository<Entity>` do TypeORM. Este é o **único lugar** onde TypeORM é usado diretamente.

### Estrutura de um Repository Customizado

```typescript
// src/modules/users/repositories/users.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, FindOptionsWhere } from 'typeorm';
import { UserEntity } from '../entities/user.entity';

// Tipos de input/output do repository — sem vazar Entity para o UseCase como "parâmetro de query"
type CreateUserData = {
  email: string;
  passwordHash: string;
  displayName?: string;
};

type FindUsersFilter = {
  status?: string;
  createdAfter?: Date;
};

@Injectable()
export class UsersRepository {
  constructor(
    @InjectRepository(UserEntity)
    private readonly repo: Repository<UserEntity>,
  ) {}

  // ─── CREATE ────────────────────────────────────────────────────────────────

  async create(data: CreateUserData): Promise<UserEntity> {
    const entity = this.repo.create(data);       // instancia sem salvar
    return this.repo.save(entity);               // persiste
  }

  // ─── READ ──────────────────────────────────────────────────────────────────

  async findById(id: string): Promise<UserEntity | null> {
    return this.repo.findOne({ where: { id } });
  }

  async findByEmail(email: string): Promise<UserEntity | null> {
    return this.repo.findOne({ where: { email } });
  }

  async findByFilter(filter: FindUsersFilter): Promise<UserEntity[]> {
    const qb = this.repo.createQueryBuilder('user');

    if (filter.status) {
      qb.andWhere('user.status = :status', { status: filter.status });
    }
    if (filter.createdAfter) {
      qb.andWhere('user.createdAt > :date', { date: filter.createdAfter });
    }

    return qb.orderBy('user.createdAt', 'DESC').getMany();
  }

  // ─── UPDATE ────────────────────────────────────────────────────────────────

  async updateById(id: string, data: Partial<CreateUserData>): Promise<UserEntity | null> {
    await this.repo.update({ id }, data);
    return this.findById(id);
  }

  // ─── DELETE ────────────────────────────────────────────────────────────────

  async deleteById(id: string): Promise<void> {
    await this.repo.delete({ id });
  }

  async softDeleteById(id: string): Promise<void> {
    await this.repo.softDelete({ id });      // requer @DeleteDateColumn() na Entity
  }
}
```

### Regras do Repository

| Regra | Justificativa |
|-------|---------------|
| **Sempre retorna `Entity` ou `null`** | O UseCase mapeia o resultado para tipos de domínio ou ViewModels |
| **Nunca recebe DTO como parâmetro** | Crie tipos locais (`CreateUserData`) para desacoplar de DTOs da API |
| **Nunca contém lógica de negócio** | `if (user.age < 18) throw` → pertence ao UseCase |
| **Usa `QueryBuilder` para queries complexas** | `find()` para casos simples, `createQueryBuilder()` para filtros dinâmicos |
| **É registrado como `@Injectable()`** | Nunca estenda `Repository<Entity>` diretamente — use composição |
| **Nunca acessa outro Repository** | Relações entre domínios são responsabilidade do UseCase |

### Como o Repository é declarado no módulo

```typescript
// src/modules/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserEntity } from './entities/user.entity';
import { UsersRepository } from './repositories/users.repository';

@Module({
  imports: [
    TypeOrmModule.forFeature([UserEntity]),   // registra a Entity para injeção
  ],
  providers: [
    UsersRepository,                          // repository visível apenas dentro do módulo
  ],
  exports: [
    // Exporte apenas se outro módulo precisar → preferência: exporte o UseCase, não o Repository
    // UsersRepository,  ← raramente necessário
  ],
})
export class UsersModule {}
```

> **Preferência:** Nunca exporte o Repository diretamente. Se outro módulo precisa de dados do domínio `users`, exporte o UseCase correspondente. O Repository é um detalhe de implementação interno.

---

## 3. A Regra de Ouro — Entidade Nunca Vaza para Fora do Repository

A Entity é o contrato entre o Repository e a camada de domínio (UseCase). Ela **nunca sobe além do UseCase** — nunca chega ao Controller e nunca aparece numa response HTTP.

```
Repository → retorna Entity
    ↓
UseCase → recebe Entity, processa, retorna Entity
    ↓
Controller → recebe Entity, CONVERTE para ViewModel → response HTTP
```

### Por que isso importa

```typescript
// ❌ ERRADO — Entity exposta diretamente como resposta HTTP
@Get(':id')
async findOne(@Param('id') id: string): Promise<UserEntity> {  // ← RISCO: expõe passwordHash!
  return this.getUserUseCase.execute({ id });                   // ← Entity chega pura na response
}

// ✅ CORRETO — Entity mapeada para ViewModel antes de retornar
@Get(':id')
async findOne(@Param('id') id: string): Promise<UserViewModel> {
  const user = await this.getUserUseCase.execute({ id });
  return UserViewModel.fromEntity(user);     // ← apenas campos seguros expostos
}
```

### ViewModel protege contra vazamento acidental de dados

```typescript
// src/modules/users/view-models/user.view-model.ts
export class UserViewModel {
  id: string;
  email: string;
  displayName: string | null;
  createdAt: string;
  // ❌ passwordHash NÃO está aqui — nunca exposto

  static fromEntity(entity: UserEntity): UserViewModel {
    const vm = new UserViewModel();
    vm.id = entity.id;
    vm.email = entity.email;
    vm.displayName = entity.displayName;
    vm.createdAt = entity.createdAt.toISOString();
    return vm;
  }

  static fromEntities(entities: UserEntity[]): UserViewModel[] {
    return entities.map(UserViewModel.fromEntity);
  }
}
```

---

## 4. Migrations — Versionamento do Schema

**Nunca use `synchronize: true` em produção.** Migrations são a única forma segura de evoluir o schema.

### Configuração do DatabaseModule (em `src/infra/database/`)

```typescript
// src/infra/database/database.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigService } from '@nestjs/config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        url: config.get<string>('DATABASE_URL'),
        entities: [__dirname + '/../../**/*.entity{.ts,.js}'],
        migrations: [__dirname + '/migrations/*{.ts,.js}'],
        migrationsRun: false,              // nunca auto-run em produção
        synchronize: false,               // ← NUNCA true em produção
        logging: config.get('NODE_ENV') === 'development',
        ssl: config.get('NODE_ENV') === 'production'
          ? { rejectUnauthorized: false }
          : false,
      }),
    }),
  ],
})
export class DatabaseModule {}
```

### Data Source para CLI (arquivo separado)

```typescript
// src/infra/database/data-source.ts
// Usado APENAS pelo CLI do TypeORM — não importado em runtime
import { DataSource } from 'typeorm';
import * as dotenv from 'dotenv';

dotenv.config();

export const AppDataSource = new DataSource({
  type: 'postgres',
  url: process.env.DATABASE_URL,
  entities: ['src/**/*.entity.ts'],
  migrations: ['src/infra/database/migrations/*.ts'],
  synchronize: false,
});
```

### Scripts no `package.json`

```json
{
  "scripts": {
    "migration:generate": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:generate src/infra/database/migrations/$npm_config_name",
    "migration:run": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:run",
    "migration:revert": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:revert",
    "migration:show": "typeorm-ts-node-commonjs -d src/infra/database/data-source.ts migration:show"
  }
}
```

```bash
# Gerar uma migration após alterar uma Entity
npm run migration:generate --name=add-display-name-to-users

# Executar migrations pendentes
npm run migration:run

# Reverter a última migration
npm run migration:revert
```

### Estrutura de uma Migration

```typescript
// src/infra/database/migrations/1709900000000-AddDisplayNameToUsers.ts
import { MigrationInterface, QueryRunner, TableColumn } from 'typeorm';

export class AddDisplayNameToUsers1709900000000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.addColumn(
      'users',
      new TableColumn({
        name: 'display_name',
        type: 'varchar',
        length: '100',
        isNullable: true,
        default: null,
      }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropColumn('users', 'display_name');
  }
}
```

### Convenções de Migrations

| Regra | Detalhe |
|-------|---------|
| **Timestamp no nome** | TypeORM gera automaticamente — ex: `1709900000000-NomeDaAlteracao` |
| **Nome descritivo** | `AddDisplayNameToUsers`, `CreateMealsTable`, `AddIndexToUserEmail` |
| **Sempre implementar `down()`** | Permite rollback seguro |
| **Uma mudança por migration** | Facilita rollback granular e revisão de código |
| **Nunca editar migration já executada** | Crie uma nova migration para corrigir |
| **Testar o `down()` localmente** | `migration:revert` deve funcionar sem erros |

---

## 5. Índices e Otimizações de Query

### Índices na Entity (preferido para casos simples)

```typescript
@Entity('meals')
@Index(['userId', 'createdAt'])                      // composto — queries de histórico por usuário
@Index(['status', 'scheduledAt'])                    // composto — queries de agendamento
export class MealEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'uuid' })
  @Index()                                           // índice simples — busca por userId
  userId: string;

  @Column({ type: 'varchar', length: 20 })
  status: string;

  @Column({ type: 'timestamp', nullable: true })
  scheduledAt: Date | null;

  @CreateDateColumn()
  createdAt: Date;
}
```

### Quando criar índices

| Query frequente | Índice recomendado |
|----------------|-------------------|
| `WHERE userId = ?` | `@Index()` na coluna `userId` |
| `WHERE userId = ? ORDER BY createdAt DESC` | Índice composto `(userId, createdAt)` |
| `WHERE email = ?` (busca de login) | `unique: true` na coluna `email` |
| `WHERE status = ? AND scheduledAt < ?` | Índice composto `(status, scheduledAt)` |
| `LIKE '%termo%'` em texto longo | Full-text search (índice GIN no PostgreSQL) |

### QueryBuilder para queries complexas

```typescript
// ✅ QueryBuilder — Use para filtros dinâmicos, JOINs e paginação
async findPaginatedByUser(
  userId: string,
  page: number,
  limit: number,
): Promise<{ items: MealEntity[]; total: number }> {
  const qb = this.repo
    .createQueryBuilder('meal')
    .where('meal.userId = :userId', { userId })
    .orderBy('meal.createdAt', 'DESC')
    .skip((page - 1) * limit)
    .take(limit);

  const [items, total] = await qb.getManyAndCount();
  return { items, total };
}

// ✅ JOIN explícito — carrega relação apenas quando necessário
async findWithFoods(mealId: string): Promise<MealEntity | null> {
  return this.repo
    .createQueryBuilder('meal')
    .leftJoinAndSelect('meal.foods', 'food')   // JOIN apenas nesta query
    .where('meal.id = :mealId', { mealId })
    .getOne();
}
```

### Evitar N+1 — nunca use `eager: true` em relações

```typescript
// ❌ EVITAR — eager loading causa N+1 implícito
@OneToMany(() => FoodEntity, (food) => food.meal, { eager: true })
foods: FoodEntity[];   // carregado em TODA query de MealEntity, mesmo sem precisar

// ✅ CORRETO — lazy por padrão, JOIN explícito quando necessário
@OneToMany(() => FoodEntity, (food) => food.meal, { lazy: true })
foods: Relation<FoodEntity[]>;    // carregado APENAS quando explicitamente pedido

// No repository — join explícito quando necessário:
async findWithFoods(mealId: string): Promise<MealEntity | null> {
  return this.repo.findOne({
    where: { id: mealId },
    relations: { foods: true },     // carrega relação apenas nesta query
  });
}
```

---

## 6. Transactions — Operações Atômicas

Use transações quando múltiplas operações de banco precisam ser atômicas (todas ou nenhuma).

```typescript
// src/modules/auth/repositories/auth.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { DataSource, Repository } from 'typeorm';
import { UserEntity } from '../entities/user.entity';
import { UserProfileEntity } from '../entities/user-profile.entity';

@Injectable()
export class AuthRepository {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepo: Repository<UserEntity>,
    private readonly dataSource: DataSource,     // injetado para transactions
  ) {}

  // Cria usuário + perfil em uma única transação
  async createUserWithProfile(data: {
    email: string;
    passwordHash: string;
    displayName: string;
  }): Promise<UserEntity> {
    return this.dataSource.transaction(async (manager) => {
      const user = manager.create(UserEntity, {
        email: data.email,
        passwordHash: data.passwordHash,
      });
      await manager.save(user);

      const profile = manager.create(UserProfileEntity, {
        userId: user.id,
        displayName: data.displayName,
      });
      await manager.save(profile);

      return user;
    });
  }
}
```

> **Regra:** Transactions sempre ficam no Repository, nunca no UseCase. O UseCase coordena a operação chamando o método do repository que define a transação.

---

## 7. Soft Delete — Exclusão Lógica

Para entidades que precisam de histórico ou auditoria, use soft delete em vez de exclusão física.

```typescript
// Entity com soft delete
import { Entity, Column, DeleteDateColumn } from 'typeorm';

@Entity('meals')
export class MealEntity {
  // ... outros campos

  @DeleteDateColumn()                           // null = ativo, Date = deletado
  deletedAt: Date | null;
}
```

```typescript
// Repository com soft delete
async softDeleteById(id: string): Promise<void> {
  await this.repo.softDelete({ id });           // preenche deletedAt
}

async restoreById(id: string): Promise<void> {
  await this.repo.restore({ id });              // limpa deletedAt
}

// findOne / find automaticamente excluem registros com deletedAt != null
// Para incluir deletados: { withDeleted: true }
async findDeletedById(id: string): Promise<MealEntity | null> {
  return this.repo.findOne({ where: { id }, withDeleted: true });
}
```

---

## 8. Configuração TypeORM Recomendada

### `database.config.ts` (complementa `config-management.md`)

```typescript
// src/config/database.config.ts
import { registerAs } from '@nestjs/config';
import { z } from 'zod';

export const databaseConfigSchema = z.object({
  DATABASE_URL: z.string().url(),
  DATABASE_POOL_SIZE: z.coerce.number().default(10),
  DATABASE_SSL: z.coerce.boolean().default(false),
});

export const databaseConfig = registerAs('database', () => ({
  url: process.env.DATABASE_URL,
  poolSize: parseInt(process.env.DATABASE_POOL_SIZE ?? '10'),
  ssl: process.env.DATABASE_SSL === 'true',
}));
```

### Checklist de configuração por ambiente

| Configuração | `development` | `staging` | `production` |
|-------------|--------------|-----------|-------------|
| `synchronize` | ❌ `false` | ❌ `false` | ❌ `false` |
| `migrationsRun` | ✅ `true` (conveniente) | ✅ `true` | ✅ `true` (via CI/CD) |
| `logging` | `['query', 'error']` | `['error']` | `['error']` |
| `ssl` | `false` | `true` | `true` |
| `poolSize` | 5 | 10 | 20+ |

---

## 9. Checklist de Validação

### Entity
- [ ] Nome da classe termina com `Entity`?
- [ ] `@Entity('nome_tabela')` especifica o nome da tabela em `snake_case` plural?
- [ ] Nenhum método de negócio na Entity?
- [ ] Tipos explícitos em todos os `@Column({ type: ... })`?
- [ ] Relações usam `Relation<T>` do TypeORM para evitar circular imports?
- [ ] Índices criados para colunas usadas em `WHERE` e `ORDER BY` frequentes?

### Repository
- [ ] Classe é `@Injectable()` (composição, não herança de `Repository<Entity>`)?
- [ ] `@InjectRepository(Entity)` injetado no constructor?
- [ ] Todos os métodos retornam `Entity` ou `null` (nunca DTO ou ViewModel)?
- [ ] Nenhuma lógica de negócio (if de domínio, cálculos)?
- [ ] Relações carregadas com JOIN explícito (nunca `eager: true`)?
- [ ] Queries complexas usam `createQueryBuilder()`?

### Migrations
- [ ] `synchronize: false` em todos os ambientes?
- [ ] Migration gerada com CLI (`migration:generate`), não escrita manualmente?
- [ ] `down()` implementado e testado localmente?
- [ ] Nome da migration é descritivo?

### Exposição de Dados
- [ ] Entity nunca retornada diretamente em response HTTP?
- [ ] Controller sempre converte Entity → ViewModel antes de retornar?
- [ ] Repository nunca exportado diretamente no `exports` do módulo (a não ser que estritamente necessário)?

---

## Referências

- [TypeORM Docs — Entities](https://typeorm.io/entities)
- [TypeORM Docs — Relations](https://typeorm.io/relations)
- [TypeORM Docs — Migrations](https://typeorm.io/migrations)
- [TypeORM Docs — Indices](https://typeorm.io/indices)
- [NestJS Docs — Database (TypeORM)](https://docs.nestjs.com/techniques/database)
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Regras de camadas, onde Entity vive e o papel do Repository
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura de pastas, use-cases e convenções de nomenclatura
- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — Configuração do DatabaseModule com `@nestjs/config`
