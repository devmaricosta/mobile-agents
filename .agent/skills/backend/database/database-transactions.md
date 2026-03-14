---
name: database-transactions
description: Gerenciamento de transações no NestJS — Unit of Work Pattern, transações no TypeORM (EntityManager.transaction e QueryRunner), transações no Prisma ($transaction batch e interativa), AsyncLocalStorage para propagação de contexto transacional sem prop drilling, rollback em falhas de operações compostas, transações cross-repository, e como o NestJS gerencia o ciclo de vida da conexão.
---

# Database Transactions — Gerenciamento de Transações no NestJS

Você é um Engenheiro de Software Senior especialista em banco de dados e NestJS. Esta skill é a **referência aprofundada de transações** para o projeto. Ela complementa `typeorm-patterns.md` (seção 6 — transações básicas), `prisma-patterns.md` (seção 7 — transações básicas) e `clean-architecture.md` (regras de camada). Consulte esta skill sempre que uma operação envolver múltiplas escritas no banco que devem ser atômicas.

> **Regra fundamental:** Transações gerenciam o *quando* os dados são confirmados no banco — elas ficam na **camada de Infrastructure (Repository)**, nunca no UseCase. O UseCase coordena *o que* acontece, o Repository garante *que acontece de forma atômica*.

---

## 1. O que é uma Transação e Quando Usar

### Conceito ACID

Uma transação garante as propriedades ACID:

| Propriedade | Significado |
|-------------|-------------|
| **Atomicidade** | Todas as operações do bloco são confirmadas ou nenhuma é |
| **Consistência** | O banco passa de um estado válido para outro estado válido |
| **Isolamento** | Transações concorrentes não interferem entre si |
| **Durabilidade** | Uma vez confirmadas, as mudanças persistem mesmo em falha de sistema |

### Quando usar transação (obrigatório)

```
✅ Criar usuário + perfil + configurações padrão (3 registros ligados)
✅ Transferir saldo: debitar conta A + creditar conta B
✅ Criar pedido + itens do pedido + decrementar estoque
✅ Registrar pagamento + atualizar status do pedido + gerar nota fiscal
✅ Qualquer operação onde "metade feito" é pior que "não feito"
```

### Quando NÃO usar transação

```
❌ Operações de leitura pura (SELECT) — não modifica estado
❌ Operações de escrita únicas e independentes — já são atômicas por natureza
❌ Operações que podem ser compensadas por outra operação posterior (Saga Pattern)
```

> **Custo:** Transações bloqueiam recursos do banco e aumentam a latência. Use apenas quando a atomicidade for realmente necessária.

---

## 2. Unit of Work Pattern no NestJS

### O que é o Unit of Work

O **Unit of Work** (UoW) é um padrão que agrupa múltiplas operações de banco em uma única unidade de trabalho atômica. Ele rastreia as mudanças feitas durante uma operação de negócio e as confirma (ou reverte) de uma vez ao final.

```
UseCase.execute()
    ├── userRepo.create(user)       ──┐
    ├── profileRepo.create(profile)  │ → UoW: confirma tudo ou reverte tudo
    └── settingsRepo.create(settings)┘
```

### Responsabilidades por camada

| Camada | Responsabilidade |
|--------|-----------------|
| **UseCase** | Orquestrar a sequência de operações, decidir *o que* fazer |
| **Repository** | Executar as operações dentro da transação, decidir *como* é atômico |
| **Infrastructure (DataSource/PrismaService)** | Fornecer o mecanismo de transação (commit/rollback) |

### Padrão de implementação no projeto

```typescript
// ✅ UseCase delega todo o controle transacional para o Repository
@Injectable()
export class RegisterUseCase {
  constructor(private readonly authRepository: AuthRepository) {}

  async execute(input: RegisterInput): Promise<UserEntity> {
    // UseCase valida regras de negócio
    const existingUser = await this.authRepository.findByEmail(input.email);
    if (existingUser) {
      throw new ConflictException('E-mail já cadastrado');
    }

    const passwordHash = await bcrypt.hash(input.password, 10);

    // UseCase delega a criação atômica para o Repository
    // O Repository gerencia internamente a transação
    return this.authRepository.createWithProfile({
      email: input.email,
      passwordHash,
      displayName: input.displayName,
    });
  }
}
```

```typescript
// ✅ Repository gerencia a transação — UseCase não sabe que existe transação
@Injectable()
export class AuthRepository {
  constructor(private readonly dataSource: DataSource) {}

  async createWithProfile(data: CreateUserWithProfileData): Promise<UserEntity> {
    return this.dataSource.transaction(async (manager) => {
      // Todas as operações usam o manager transacional
      const user = manager.create(UserEntity, { ... });
      await manager.save(user);

      const profile = manager.create(UserProfileEntity, { userId: user.id, ... });
      await manager.save(profile);

      return user;
    });
    // commit automático se nenhuma exceção → rollback automático se exceção
  }
}
```

---

## 3. TypeORM — `EntityManager.transaction` (Caso Simples)

A forma mais simples e segura de gerenciar transações no TypeORM. O commit e rollback são automáticos.

### Injetando o DataSource

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
    private readonly dataSource: DataSource,  // ← injeta o DataSource para transações
  ) {}

  // ─── Operação simples (sem transação) ──────────────────────────────────────

  async findByEmail(email: string): Promise<UserEntity | null> {
    return this.userRepo.findOne({ where: { email } });  // usa o repo normal
  }

  // ─── Operação composta (com transação) ─────────────────────────────────────

  async createWithProfile(data: {
    email: string;
    passwordHash: string;
    displayName: string;
  }): Promise<UserEntity> {
    return this.dataSource.transaction(async (manager) => {
      // ⚠️ REGRA CRÍTICA: dentro do callback, use SEMPRE o `manager`
      // Nunca use `this.userRepo` dentro de uma transação — ele não participa da transação!

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
      // ✅ commit automático aqui — se chegou até aqui sem exceção
    });
    // ✅ rollback automático se qualquer await acima lançar exceção
  }
}
```

### Regras do `EntityManager.transaction`

| Regra | Justificativa |
|-------|---------------|
| **Use `manager`, nunca `this.repo` dentro do callback** | `this.repo` opera fora da transação — operações com ele não fazem parte do commit/rollback |
| **Não faça try/catch dentro do callback para silenciar erros** | Erros silenciados impedem o rollback automático |
| **Não execute chamadas externas (HTTP, email) dentro do callback** | Se a chamada externa falhar, o banco já pode ter feito commit de parte dos dados |
| **Mantenha o callback curto** | Quanto mais tempo a transação dura, mais recursos do banco ficam bloqueados |

### ❌ Anti-padrão: usar `this.repo` dentro da transação

```typescript
// ❌ ERRADO — this.userRepo opera fora da transação
async createWithProfile(data: CreateData): Promise<UserEntity> {
  return this.dataSource.transaction(async (manager) => {
    const user = await this.userRepo.save({ email: data.email }); // ← fora da transação!
    const profile = manager.create(UserProfileEntity, { userId: user.id });
    await manager.save(profile);
    // Se manager.save(profile) falhar, o user JÁ foi persistido (não faz rollback)
    return user;
  });
}

// ✅ CORRETO — apenas manager dentro do callback
async createWithProfile(data: CreateData): Promise<UserEntity> {
  return this.dataSource.transaction(async (manager) => {
    const user = manager.create(UserEntity, { email: data.email });
    await manager.save(user);  // ← dentro da transação
    const profile = manager.create(UserProfileEntity, { userId: user.id });
    await manager.save(profile);  // ← dentro da transação
    return user;
  });
}
```

---

## 4. TypeORM — `QueryRunner` (Controle Granular)

Use o `QueryRunner` quando precisar de **controle explícito** sobre o ciclo de vida da transação: transaction savepoints, múltiplos commits, ou integração com sistemas externos que precisam acontecer *dentro* do bloco transacional.

### Anatomia completa do QueryRunner

```typescript
// src/modules/orders/repositories/orders.repository.ts
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { OrderEntity } from '../entities/order.entity';
import { OrderItemEntity } from '../entities/order-item.entity';
import { InventoryEntity } from '../entities/inventory.entity';

@Injectable()
export class OrdersRepository {
  constructor(private readonly dataSource: DataSource) {}

  async createOrder(data: CreateOrderData): Promise<OrderEntity> {
    const queryRunner = this.dataSource.createQueryRunner();

    // 1. Conecta o queryRunner ao pool de conexões
    await queryRunner.connect();

    // 2. Inicia a transação explicitamente
    await queryRunner.startTransaction();

    try {
      // 3. Executa as operações usando queryRunner.manager
      const order = queryRunner.manager.create(OrderEntity, {
        userId: data.userId,
        totalAmount: data.totalAmount,
      });
      await queryRunner.manager.save(order);

      for (const item of data.items) {
        const orderItem = queryRunner.manager.create(OrderItemEntity, {
          orderId: order.id,
          productId: item.productId,
          quantity: item.quantity,
          unitPrice: item.unitPrice,
        });
        await queryRunner.manager.save(orderItem);

        // Decrementa estoque
        await queryRunner.manager.decrement(
          InventoryEntity,
          { productId: item.productId },
          'quantity',
          item.quantity,
        );
      }

      // 4. Confirma a transação
      await queryRunner.commitTransaction();

      return order;

    } catch (error) {
      // 5. Reverte em caso de qualquer erro
      await queryRunner.rollbackTransaction();
      throw error;  // ← sempre relance o erro — nunca silencie

    } finally {
      // 6. SEMPRE libera o queryRunner — mesmo em caso de erro
      //    Sem isso, a conexão fica ocupada no pool indefinidamente
      await queryRunner.release();
    }
  }
}
```

### `EntityManager.transaction` vs `QueryRunner` — Quando usar cada um

| Critério | `EntityManager.transaction` | `QueryRunner` |
|----------|----------------------------|---------------|
| **Simplicidade** | ✅ Mais simples — commit/rollback automático | ❌ Mais verboso — manual |
| **Controle** | ❌ Sem acesso ao runner | ✅ Acesso total ao ciclo de vida |
| **Savepoints** | ❌ Não suporta | ✅ Suporta com `queryRunner.startTransaction('SAVEPOINT')` |
| **Isolation level** | ✅ Via segundo parâmetro | ✅ Via `startTransaction('SERIALIZABLE')` |
| **Casos de uso** | 90% dos casos — operações atômicas simples | Controle de isolation level, savepoints, integração complexa |

### Isolation Levels no TypeORM

```typescript
// EntityManager.transaction com isolation level
return this.dataSource.transaction('SERIALIZABLE', async (manager) => {
  // Operações aqui são SERIALIZÁVEIS — máxima proteção contra race conditions
  const balance = await manager.findOne(AccountEntity, { where: { id } });
  if (balance.amount < data.amount) throw new Error('Saldo insuficiente');
  await manager.decrement(AccountEntity, { id }, 'amount', data.amount);
  await manager.increment(AccountEntity, { id: data.targetId }, 'amount', data.amount);
});

// QueryRunner com isolation level
await queryRunner.startTransaction('READ COMMITTED');
```

| Isolation Level | Garante | Quando usar |
|----------------|---------|-------------|
| `READ UNCOMMITTED` | Nada | Nunca em produção |
| `READ COMMITTED` (padrão PostgreSQL) | Sem dirty reads | Operações de leitura comuns |
| `REPEATABLE READ` | Sem non-repeatable reads | Relatórios e agregações |
| `SERIALIZABLE` | Execução totalmente sequencial | Transferências financeiras, reservas, estoque |

---

## 5. TypeORM — AsyncLocalStorage para Propagação de Contexto

### O Problema: Prop Drilling do EntityManager

Quando múltiplos repositories precisam participar da mesma transação, surge o problema do "prop drilling" — passar o `manager`/`queryRunner` como parâmetro por todas as camadas:

```typescript
// ❌ Prop drilling — UseCase precisa conhecer EntityManager (viola Clean Architecture)
async execute(input: RegisterInput): Promise<void> {
  return this.dataSource.transaction(async (manager) => {
    // UseCase gerencia a transação diretamente — VIOLAÇÃO DA CLEAN ARCHITECTURE
    await this.userRepo.createWithManager(manager, {...}); // ← prop drilling
    await this.profileRepo.createWithManager(manager, {...}); // ← prop drilling
  });
}
```

### A Solução: AsyncLocalStorage

O `AsyncLocalStorage` (ALS) do Node.js permite armazenar dados acessíveis em toda a pilha de chamadas assíncronas de uma requisição, sem precisar passá-los como parâmetro.

```typescript
// src/infra/database/transaction.storage.ts
import { AsyncLocalStorage } from 'async_hooks';
import { EntityManager } from 'typeorm';

// Storage global — compartilhado em toda a aplicação
export const transactionStorage = new AsyncLocalStorage<EntityManager>();
```

```typescript
// src/infra/database/transaction.service.ts
import { Injectable } from '@nestjs/common';
import { DataSource, EntityManager } from 'typeorm';
import { transactionStorage } from './transaction.storage';

@Injectable()
export class TransactionService {
  constructor(private readonly dataSource: DataSource) {}

  // Obtém o EntityManager ativo (transacional ou global)
  getManager(): EntityManager {
    return transactionStorage.getStore() ?? this.dataSource.manager;
  }

  // Executa um callback dentro de uma transação com ALS
  async runInTransaction<T>(callback: () => Promise<T>): Promise<T> {
    return this.dataSource.transaction(async (manager) => {
      // Armazena o manager transacional no ALS
      return transactionStorage.run(manager, () => callback());
    });
  }
}
```

```typescript
// src/modules/users/repositories/users.repository.ts
@Injectable()
export class UsersRepository {
  constructor(
    private readonly transactionService: TransactionService,
  ) {}

  async create(data: CreateUserData): Promise<UserEntity> {
    // Obtém o manager correto: transacional (se em transação) ou global
    const manager = this.transactionService.getManager();
    const user = manager.create(UserEntity, data);
    return manager.save(user);
  }
}
```

```typescript
// src/modules/auth/repositories/auth.repository.ts
@Injectable()
export class AuthRepository {
  constructor(
    private readonly transactionService: TransactionService,
    private readonly usersRepository: UsersRepository,
    private readonly profilesRepository: ProfilesRepository,
  ) {}

  // O UseCase chama este método sem saber que existe transação
  async createUserWithProfile(data: CreateData): Promise<UserEntity> {
    return this.transactionService.runInTransaction(async () => {
      // Ambos os repositories obtêm o mesmo manager transacional via ALS
      const user = await this.usersRepository.create({ ... });
      await this.profilesRepository.create({ userId: user.id, ... });
      return user;
    });
  }
}
```

> **Quando usar ALS:** Em operações que envolvem múltiplos repositories na mesma transação, onde o prop drilling comprometeria a separação de camadas. Para operações simples dentro de um único repository, `dataSource.transaction()` é suficiente.

---

## 6. Prisma — `$transaction` Batch (Operações Independentes)

A transação batch do Prisma executa um array de operações em paralelo dentro de uma única transação. **Limitação crítica:** as operações não têm acesso aos resultados das outras — ideal para operações independentes.

### Caso de uso principal: paginação (findMany + count)

```typescript
// ✅ Batch transaction — executa findMany e count ATOMICAMENTE
// Garante que os dois números vieram do mesmo estado do banco
async findPaginated(params: {
  page: number;
  limit: number;
  where?: Prisma.MealWhereInput;
}): Promise<{ items: Meal[]; total: number }> {
  const { page, limit, where } = params;

  const baseWhere: Prisma.MealWhereInput = {
    deletedAt: null,
    ...where,
  };

  const [items, total] = await this.prisma.$transaction([
    this.prisma.meal.findMany({
      where: baseWhere,
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * limit,
      take: limit,
    }),
    this.prisma.meal.count({ where: baseWhere }),
  ]);

  return { items, total };
}
```

### ❌ Limitação do batch: não pode usar resultado de uma operação na outra

```typescript
// ❌ ERRADO — batch não permite usar o resultado do user.create no profile.create
const [user] = await this.prisma.$transaction([
  this.prisma.user.create({ data: { email: '...' } }),
  this.prisma.profile.create({
    data: {
      userId: ???,  // ← IMPOSSÍVEL — não temos o id do user ainda
      displayName: '...',
    },
  }),
]);

// ✅ CORRETO — use transação interativa quando depender do resultado de outra operação
```

---

## 7. Prisma — `$transaction` Interativo (Operações Dependentes)

A transação interativa usa um callback com um cliente Prisma transacional (`tx`). Ela permite que operações dependam dos resultados de operações anteriores dentro da mesma transação.

### Anatomia da transação interativa

```typescript
// src/modules/auth/repositories/auth.repository.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '@infra/database/prisma.service';
import { User, Prisma } from '@prisma/client';

@Injectable()
export class AuthRepository {
  constructor(private readonly prisma: PrismaService) {}

  async createUserWithProfile(data: {
    email: string;
    passwordHash: string;
    displayName: string;
  }): Promise<User> {
    return this.prisma.$transaction(async (tx) => {
      // ⚠️ REGRA CRÍTICA: use SEMPRE `tx` (o client transacional)
      // Nunca use `this.prisma` dentro do callback — operações com this.prisma
      // não fazem parte da transação e não serão revertidas em caso de erro

      const user = await tx.user.create({
        data: {
          email: data.email,
          passwordHash: data.passwordHash,
        },
      });

      // user.id está disponível aqui — operações são sequenciais, diferente do batch
      await tx.userProfile.create({
        data: {
          userId: user.id,   // ← usa o id do user criado acima
          displayName: data.displayName,
        },
      });

      return user;
      // ✅ commit automático ao sair do callback sem exceção
    });
    // ✅ rollback automático se qualquer await acima lançar exceção
  }
}
```

### Isolation Level e Timeout no Prisma

```typescript
// Configuração avançada de transação interativa
return this.prisma.$transaction(
  async (tx) => {
    // Operações da transação
    const order = await tx.order.create({ data: { userId } });
    await tx.inventory.update({
      where: { productId },
      data: { quantity: { decrement: quantity } },
    });
    return order;
  },
  {
    // Isolation level — padrão do banco se omitido
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable,

    // Timeout em ms — transação é encerrada se demorar mais que isso
    // Padrão: 5000ms. Importante em operações longas para evitar deadlocks
    timeout: 10_000,

    // maxWait: tempo máximo para aguardar uma conexão disponível (padrão: 2000ms)
    maxWait: 5_000,
  },
);
```

### Regras da transação interativa

| Regra | Justificativa |
|-------|---------------|
| **Use `tx`, nunca `this.prisma` dentro do callback** | `this.prisma` opera fora da transação — não faz parte do rollback |
| **Não chame APIs externas dentro do `$transaction`** | APIs externas não fazem rollback. Se o banco falhar após a chamada externa, os efeitos da API já ocorreram |
| **Sempre relance exceções** | Exceções silenciadas impedem o rollback automático |
| **Configure `timeout` para operações demoradas** | Evita deadlocks por transações abertas indefinidamente |
| **Prefira `Serializable` para operações financeiras** | Evita race conditions em transferências e reservas |

---

## 8. Rollback em Falha — Boas Práticas

### Rollback automático vs. manual

```typescript
// ✅ Rollback AUTOMÁTICO — use quando possível (EntityManager.transaction / Prisma.$transaction)
async doAtomicOperation(): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    await manager.save(entityA);
    throw new Error('Algo falhou');
    await manager.save(entityB); // ← nunca chega aqui
    // → rollback automático de entityA
  });
}

// ✅ Rollback MANUAL — use apenas com QueryRunner (quando precisar de controle granular)
async doAtomicOperationManual(): Promise<void> {
  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    await queryRunner.manager.save(entityA);
    await queryRunner.manager.save(entityB);
    await queryRunner.commitTransaction();

  } catch (error) {
    await queryRunner.rollbackTransaction(); // ← rollback explícito
    throw error;  // ✅ SEMPRE relance — nunca silencie o erro

  } finally {
    await queryRunner.release(); // ← SEMPRE libere, mesmo após rollback
  }
}
```

### ❌ Anti-padrões de tratamento de erro em transações

```typescript
// ❌ ERRADO 1 — Silenciar exceção dentro do callback (impede rollback)
await this.dataSource.transaction(async (manager) => {
  try {
    await manager.save(user);
    await manager.save(profile);  // falha aqui
  } catch (error) {
    console.error(error);  // ← silencia! rollback NÃO acontece
    // Resultado: user foi salvo, profile não — dados inconsistentes
  }
});

// ❌ ERRADO 2 — Não liberar o QueryRunner no finally
async create(): Promise<void> {
  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    await queryRunner.commitTransaction();
  } catch (error) {
    await queryRunner.rollbackTransaction();
    throw error;
  }
  // ← FALTA queryRunner.release() — connection leak!
  // O pool de conexões esgotará com o tempo
}

// ❌ ERRADO 3 — Usar this.prisma dentro do $transaction callback
await this.prisma.$transaction(async (tx) => {
  const user = await this.prisma.user.create({ data }); // ← usa this.prisma, não tx
  await tx.profile.create({ data: { userId: user.id } }); // se falhar, user NÃO é revertido
});

// ✅ CORRETO — capturar error para logar e relançar transformado
await this.dataSource.transaction(async (manager) => {
  try {
    await manager.save(user);
    await manager.save(profile);
  } catch (error) {
    // ✅ Pode logar, transformar, então RELANÇAR
    this.logger.error('Falha ao criar usuário com perfil', { error, data });
    throw new InternalServerErrorException('Falha ao criar conta'); // ← relança
  }
});
```

---

## 9. Ciclo de Vida da Conexão no NestJS

### TypeORM — DataSource e Connection Pool

O TypeORM gerencia um **pool de conexões** via `DataSource`. O NestJS inicializa o `DataSource` durante o bootstrap e o fecha ao desligar.

```typescript
// src/infra/database/database.module.ts
// O TypeOrmModule.forRootAsync() inicializa o DataSource automaticamente
TypeOrmModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    type: 'postgres',
    url: config.get<string>('DATABASE_URL'),
    // Pool de conexões — cada request transacional usa uma conexão do pool
    extra: {
      max: config.get('DATABASE_POOL_SIZE', 10),    // máximo de conexões simultâneas
      min: 2,                                        // mínimo mantido aquecido
      idleTimeoutMillis: 30_000,                     // fecha conexão ociosa após 30s
      connectionTimeoutMillis: 5_000,                // timeout para obter conexão do pool
    },
  }),
});
```

**Ciclo de vida de uma transação TypeORM:**
```
Request chega → NestJS route handler
    ↓
Repository.someTransactionalMethod()
    ↓
dataSource.transaction() ou queryRunner.connect()
    → Obtém conexão do pool
    → Inicia transação (BEGIN)
    ↓
Operações de banco (INSERT, UPDATE, DELETE)
    ↓
commit() ou rollback()
    → Finaliza transação (COMMIT ou ROLLBACK)
    ↓
queryRunner.release() ou callback termina
    → Devolve conexão ao pool
```

> **Regra crítica:** Sempre libere o `QueryRunner` no `finally`. Uma conexão não liberada fica retida no pool, causando `connection pool exhausted` em produção.

### Prisma — PrismaService e Conexão

O `PrismaService` gerencia o ciclo de vida da conexão via `OnModuleInit`/`OnModuleDestroy` (cf. `prisma-patterns.md` seção 2):

```typescript
// src/infra/database/prisma.service.ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit(): Promise<void> {
    await this.$connect();     // ← NestJS chama ao inicializar o módulo
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();  // ← NestJS chama ao desligar a aplicação (SIGTERM, SIGINT)
  }
}
```

**Ciclo de vida de uma transação Prisma:**
```
$transaction() chamado
    → Prisma obtém conexão do pool interno
    → BEGIN TRANSACTION
    ↓
Operações dentro do callback (tx.model.create, tx.model.update, ...)
    ↓
Callback retorna sem exceção → COMMIT
    OU
Callback lança exceção     → ROLLBACK
    ↓
Conexão devolvida ao pool automaticamente
```

> **Diferença chave:** Com Prisma, você **nunca gerencia conexões manualmente** — o `$transaction` cuida de tudo. Com TypeORM e QueryRunner, a liberação manual no `finally` é obrigatória.

### Shutdown Gracioso no NestJS

Para garantir que transações em andamento sejam finalizadas antes do processo encerrar:

```typescript
// src/main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Habilita shutdown hooks — chama OnModuleDestroy nos providers
  app.enableShutdownHooks();  // ← obrigatório para PrismaService desconectar corretamente

  await app.listen(3000);
}
```

---

## 10. Transações Cross-Repository (UseCase Orquestrando)

### Cenário: UseCase precisa que dois repositories trabalhem na mesma transação

Quando uma operação de negócio envolve repositories de **domínios diferentes** (ex: criar um pedido e decrementar estoque), há duas abordagens.

### Abordagem 1: Método de Repository dedicado (preferida para casos simples)

Crie um método no repository que coordena a operação atômica internamente:

```typescript
// src/modules/orders/repositories/orders.repository.ts
// Este repository coordena a operação cross-entity internamente
@Injectable()
export class OrdersRepository {
  constructor(private readonly dataSource: DataSource) {}

  async createWithInventoryDecrement(data: CreateOrderData): Promise<OrderEntity> {
    return this.dataSource.transaction(async (manager) => {
      // 1. Cria o pedido
      const order = manager.create(OrderEntity, { userId: data.userId });
      await manager.save(order);

      // 2. Verifica e decrementa estoque (mesmo domínio de infra = OK no repository)
      for (const item of data.items) {
        const inventory = await manager.findOne(InventoryEntity, {
          where: { productId: item.productId },
          lock: { mode: 'pessimistic_write' }, // ← lock para evitar race condition
        });

        if (!inventory || inventory.quantity < item.quantity) {
          throw new BadRequestException(`Estoque insuficiente para produto ${item.productId}`);
        }

        await manager.decrement(InventoryEntity, { productId: item.productId }, 'quantity', item.quantity);
      }

      return order;
    });
  }
}
```

```typescript
// UseCase — não sabe que existe transação nem cross-repository
@Injectable()
export class CreateOrderUseCase {
  constructor(private readonly ordersRepository: OrdersRepository) {}

  async execute(input: CreateOrderInput): Promise<OrderEntity> {
    return this.ordersRepository.createWithInventoryDecrement(input);
  }
}
```

### Abordagem 2: AsyncLocalStorage para múltiplos repositories (TypeORM)

Quando a coordenação cross-repository seria muito complexa em um único método, use ALS (ver seção 5):

```typescript
// UseCase usa um "facade" de repository que coordena via ALS
@Injectable()
export class TransferUseCase {
  constructor(
    private readonly transactionService: TransactionService,
    private readonly accountsRepository: AccountsRepository,
    private readonly ledgerRepository: LedgerRepository,
  ) {}

  async execute(input: TransferInput): Promise<void> {
    await this.transactionService.runInTransaction(async () => {
      // Ambos usam o mesmo EntityManager via ALS — mesma transação
      await this.accountsRepository.debit(input.fromId, input.amount);
      await this.accountsRepository.credit(input.toId, input.amount);
      await this.ledgerRepository.record({
        type: 'transfer',
        fromId: input.fromId,
        toId: input.toId,
        amount: input.amount,
      });
    });
  }
}
```

### Abordagem 3: Prisma — passando `tx` explicitamente (aceito quando necessário)

Para Prisma, em casos cross-repository, passe o cliente `tx` como parâmetro dos métodos do repository:

```typescript
// src/modules/auth/repositories/auth.repository.ts
@Injectable()
export class AuthRepository {
  constructor(private readonly prisma: PrismaService) {}

  // Método principal — gerencia a transação
  async createWithProfile(data: CreateData): Promise<User> {
    return this.prisma.$transaction(async (tx) => {
      // Passa `tx` para os métodos que precisam participar da transação
      const user = await this.createUser(tx, data);
      await this.createProfile(tx, { userId: user.id, displayName: data.displayName });
      return user;
    });
  }

  // Métodos privados aceitam `tx` como parâmetro
  private async createUser(tx: Prisma.TransactionClient, data: CreateUserData): Promise<User> {
    return tx.user.create({
      data: { email: data.email, passwordHash: data.passwordHash },
    });
  }

  private async createProfile(
    tx: Prisma.TransactionClient,
    data: CreateProfileData,
  ): Promise<void> {
    await tx.userProfile.create({ data });
  }
}
```

> **Trade-off:** Passar `tx` explicitamente é simples e transparente, mas cria acoplamento de assinatura. O ALS resolve o acoplamento mas adiciona complexidade de setup. Escolha com base na escala e complexidade do projeto.

---

## 11. Anti-Padrões e Decisões Erradas

### ❌ Gerenciar transação no UseCase

```typescript
// ❌ ERRADO — UseCase injeta DataSource e gerencia transação diretamente
@Injectable()
export class RegisterUseCase {
  constructor(
    private readonly dataSource: DataSource,  // ← PROIBIDO no UseCase
    private readonly usersRepository: UsersRepository,
  ) {}

  async execute(input: RegisterInput): Promise<void> {
    // UseCase nunca deve saber que existe DataSource, EntityManager ou $transaction
    await this.dataSource.transaction(async (manager) => {  // ← VIOLAÇÃO
      await this.usersRepository.create(manager, input);    // ← prop drilling + violação
    });
  }
}

// ✅ CORRETO — Repository encapsula a transação
@Injectable()
export class RegisterUseCase {
  constructor(private readonly authRepository: AuthRepository) {}

  async execute(input: RegisterInput): Promise<void> {
    await this.authRepository.createWithProfile(input); // ← transação invisível para o UseCase
  }
}
```

### ❌ Operação externa dentro da transação

```typescript
// ❌ ERRADO — chamada de API/email dentro da transação
async register(data: RegisterData): Promise<User> {
  return this.dataSource.transaction(async (manager) => {
    const user = manager.create(UserEntity, data);
    await manager.save(user);

    // Se o saveProfile falhar, user é revertido — mas o email JÁ FOI ENVIADO
    await this.mailerService.sendWelcomeEmail(user.email);  // ← PERIGO!

    await manager.save(profile);
    return user;
  });
}

// ✅ CORRETO — operações externas APÓS o commit da transação
async register(data: RegisterData): Promise<User> {
  // 1. Persiste os dados atomicamente
  const user = await this.authRepository.createWithProfile(data);

  // 2. Após commit bem-sucedido, executa efeitos externos
  await this.mailerService.sendWelcomeEmail(user.email);  // ← fora da transação

  return user;
}
```

### ❌ Lock sem isolamento adequado

```typescript
// ❌ ERRADO — leitura sem lock em seção crítica (race condition)
async transferBalance(fromId: string, toId: string, amount: number): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    const fromAccount = await manager.findOne(AccountEntity, { where: { id: fromId } });
    // Entre o findOne e o decrement, outra transação pode modificar fromAccount.balance!
    if (fromAccount.balance < amount) throw new Error('Saldo insuficiente');
    await manager.decrement(AccountEntity, { id: fromId }, 'balance', amount);
  });
}

// ✅ CORRETO — pessimistic lock garante que nenhuma outra transação lê/escreve durante
async transferBalance(fromId: string, toId: string, amount: number): Promise<void> {
  await this.dataSource.transaction('SERIALIZABLE', async (manager) => {
    const fromAccount = await manager.findOne(AccountEntity, {
      where: { id: fromId },
      lock: { mode: 'pessimistic_write' }, // ← bloqueia o registro para outras transações
    });
    if (!fromAccount || fromAccount.balance < amount) {
      throw new BadRequestException('Saldo insuficiente');
    }
    await manager.decrement(AccountEntity, { id: fromId }, 'balance', amount);
    await manager.increment(AccountEntity, { id: toId }, 'balance', amount);
  });
}
```

---

## 12. Checklist de Validação

### Responsabilidade de Camada
- [ ] Transações estão no Repository, nunca no UseCase?
- [ ] UseCase não injeta `DataSource`, `EntityManager`, `QueryRunner` ou `PrismaClient` diretamente?
- [ ] UseCase não injeta `PrismaService` diretamente (apenas o Repository)?

### TypeORM
- [ ] `EntityManager.transaction`: usa `manager` dentro do callback, nunca `this.repo`?
- [ ] `QueryRunner`: tem `try/catch/finally` com `rollbackTransaction()` e `release()` no `finally`?
- [ ] Exceções dentro do callback são relançadas (não silenciadas)?
- [ ] Isolation level adequado para a criticidade da operação?

### Prisma
- [ ] `$transaction` interativo: usa `tx` dentro do callback, nunca `this.prisma`?
- [ ] Timeout configurado para operações longas (`$transaction({}, { timeout: N })`)?
- [ ] Operações batch usadas apenas quando as queries são independentes entre si?

### Rollback e Conexões
- [ ] Nenhuma chamada externa (email, HTTP, fila) dentro do bloco transacional?
- [ ] `app.enableShutdownHooks()` habilitado no `main.ts`?
- [ ] Pool de conexões configurado adequadamente por ambiente?

### Cross-Repository
- [ ] Operações cross-repository encapsuladas em um método de repository dedicado?
- [ ] AsyncLocalStorage ou passagem de `tx`/`manager` usados quando necessário?
- [ ] Locks pessimistas configurados para seções críticas de escrita concorrente?

---

## Referências

- [TypeORM Docs — Transactions](https://typeorm.io/transactions)
- [Prisma Docs — Transactions and batch queries](https://www.prisma.io/docs/guides/performance-and-optimization/prisma-client-transactions-guide)
- [Node.js AsyncLocalStorage](https://nodejs.org/api/async_context.html#class-asynclocalstorage)
- [typeorm-transactional](https://www.npmjs.com/package/typeorm-transactional) — implementação de `@Transactional()` decorator com ALS para TypeORM
- [nestjs-cls](https://papouch.dev/nestjs-cls/docs) — Continuation Local Storage para NestJS com plugin para Prisma
- [typeorm-patterns.md](.agent/skills/backend/database/typeorm-patterns.md) — Entidades, Repositories, Migrations e Soft Delete com TypeORM
- [prisma-patterns.md](.agent/skills/backend/database/prisma-patterns.md) — Schema Prisma, PrismaService, Migrations e Tipagem Segura
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Regras de camada e responsabilidade de cada camada
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura de pastas, path aliases e convenções de nomenclatura
