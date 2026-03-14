---
name: graphql-pattern
description: Quando e como usar GraphQL com NestJS — abordagem code-first (recomendada para este projeto), estrutura de resolvers por domínio alinhada ao Use Cases Pattern, DataLoader para resolver o problema N+1 em relações, e como GraphQL coexiste com a API REST no mesmo projeto NestJS. Complementa clean-architecture.md (UseCase como camada de aplicação, ViewModel como shape de saída), nest-project-structure.md (estrutura de pastas e path aliases), nest-modules-pattern.md (GraphQLModule como módulo global), e rest-api-patterns.md (os dois transports coexistem sem conflito).
---

# GraphQL com NestJS — Code-First, Resolvers por Domínio e DataLoader

Você é um Arquiteto de Software Senior. Esta skill define **quando e como usar GraphQL** no projeto NestJS, sempre alinhado com a Clean Architecture (`clean-architecture.md`) e o Use Cases Pattern (`nest-project-structure.md`). A API REST continua sendo o transport padrão — GraphQL é um transport adicional para casos onde a flexibilidade de queries vale o custo.

> **Regra fundamental:** A lógica de negócio nunca fica no Resolver — fica no UseCase. O Resolver é para o GraphQL o que o Controller é para o REST: uma camada fina de apresentação que delega ao UseCase e mapeia o resultado para um tipo GraphQL (equivalente ao ViewModel).

---

## 1. Quando Usar GraphQL vs REST

### Tabela de decisão

| Situação | Use GraphQL | Use REST |
|----------|:-----------:|:--------:|
| Cliente mobile precisa de queries flexíveis (subcampos variáveis) | ✅ | ❌ |
| Dados fortemente relacionados (usuário + refeições + nutrição) | ✅ | ❌ |
| API pública consumida por múltiplos clientes com necessidades diferentes | ✅ | ❌ |
| Operações CRUD simples sem relações | ❌ | ✅ |
| Upload de arquivos | ❌ | ✅ |
| Webhooks / callbacks | ❌ | ✅ |
| Streaming / SSE | ❌ | ✅ |
| Integração com terceiros que esperam REST padrão | ❌ | ✅ |

> **Regra prática neste projeto:** REST é o padrão para todos os módulos. GraphQL é adicionado quando o cliente mobile precisa de queries compostas em uma única chamada (ex: buscar usuário + suas últimas refeições + totais nutricionais).

---

## 2. Instalação

```bash
npm install @nestjs/graphql @nestjs/apollo @apollo/server graphql dataloader
npm install -D @types/dataloader
```

> **Atenção:** Use `@nestjs/apollo` com `@apollo/server` — é a combinação moderna recomendada para NestJS v10+. Não use `apollo-server-express` diretamente (versão legada).

---

## 3. Code-First vs Schema-First — Qual Adotar

### Por que code-first é a escolha deste projeto

O projeto já usa TypeScript rigoroso em toda a stack (DTOs com `class-validator`, Entities com TypeORM, ViewModels tipados). A abordagem **code-first** elimina a duplicação de tipos — o mesmo TypeScript gera o schema GraphQL e serve como contrato da API.

| Critério | Code-First | Schema-First |
|----------|-----------|-------------|
| **Consistência com o projeto** | ✅ Tudo em TypeScript | ❌ Duplica definições TypeScript + SDL |
| **Type safety end-to-end** | ✅ Compilador TypeScript valida | ⚠️ Code-gen necessário |
| **Colaboração de equipe** | ✅ Um arquivo, uma fonte da verdade | ✅ SDL é legível por qualquer stack |
| **DX (Developer Experience)** | ✅ Sem context-switch | ❌ Edita `.graphql` + TypeScript separados |
| **Controle fino do SDL** | ❌ Gerado automaticamente | ✅ SDL escrito manualmente |

> **Decisão deste projeto: code-first.** O SDL é gerado automaticamente em `schema.gql` — trate-o como artefato de build, não como arquivo de edição.

---

## 4. Configuração do GraphQLModule

### 4.1 `graphql.config.ts` — Configuração centralizada

Consistente com `nest-project-structure.md` (configurações em `src/config/`):

```typescript
// src/config/graphql.config.ts
import { ApolloDriverConfig } from '@nestjs/apollo';
import { join } from 'path';

/**
 * Configuração do GraphQLModule para uso em GraphQLModule.forRootAsync().
 * Sempre code-first — o schema é gerado automaticamente a partir dos ObjectTypes.
 */
export const graphqlConfig: ApolloDriverConfig = {
  // ← Code-first: schema gerado no build, salvo para inspeção
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  sortSchema: true,             // schema SDL em ordem alfabética (diff mais limpo)

  // ← Playground disponível apenas fora de produção
  playground: process.env.NODE_ENV !== 'production',

  // ← Contexto por requisição: onde os DataLoaders são instanciados
  context: ({ req }) => ({ req }),

  // ← Formata erros antes de enviar ao cliente (não expõe stack em produção)
  formatError: (error) => {
    if (process.env.NODE_ENV === 'production') {
      return {
        message: error.message,
        extensions: {
          code: error.extensions?.code ?? 'INTERNAL_SERVER_ERROR',
        },
      };
    }
    return error; // desenvolvimento: retorna tudo incluindo stack
  },
};
```

### 4.2 Registrando no `AppModule`

O `GraphQLModule` é registro **global**, assim como `DatabaseModule` — conforme o padrão de `nest-modules-pattern.md`:

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver } from '@nestjs/apollo';
import { DatabaseModule } from '@infra/database/database.module';
import { AuthModule } from '@modules/auth/auth.module';
import { MealsModule } from '@modules/meals/meals.module';
import { UsersModule } from '@modules/users/users.module';
import { graphqlConfig } from '@config/graphql.config';

@Module({
  imports: [
    // ← Infraestrutura (sempre antes dos módulos de domínio)
    DatabaseModule,

    // ← GraphQL: um único GraphQLModule global
    GraphQLModule.forRoot({
      driver: ApolloDriver,
      ...graphqlConfig,
    }),

    // ← Módulos de domínio (REST + resolvers GraphQL coexistem)
    AuthModule,
    UsersModule,
    MealsModule,
  ],
})
export class AppModule {}
```

> **Por que não usar `forRootAsync`?** Para este projeto, `graphqlConfig` já lê `process.env` no `graphql.config.ts` de forma segura. Use `forRootAsync` apenas se precisar injetar `ConfigService` do `@nestjs/config` na configuração.

---

## 5. Coexistência com REST no Mesmo Projeto

### Como funciona

REST e GraphQL são transports independentes no NestJS — eles **não conflitam**. O Controller REST e o Resolver GraphQL de um mesmo domínio podem coexistir no mesmo módulo, compartilhando os mesmos Use Cases.

```
HTTP Request REST:    /api/v1/meals   → MealsController    → CreateMealUseCase
HTTP Request GraphQL: POST /graphql   → MealsResolver      → CreateMealUseCase
                                                   ↑
                                    mesmo UseCase, dois transports
```

### Estrutura do módulo com REST + GraphQL

```
src/modules/meals/
├── controllers/
│   └── meals.controller.ts       ← REST (transport HTTP clássico)
├── resolvers/                    ← GraphQL (transport alternativo)
│   └── meals.resolver.ts
├── use-cases/                    ← Lógica de negócio (compartilhada entre os dois)
│   ├── create-meal/
│   │   └── create-meal.use-case.ts
│   └── get-meal-history/
│       └── get-meal-history.use-case.ts
├── data-loaders/                 ← DataLoaders (apenas para GraphQL)
│   └── meal.loader.ts
├── repositories/
│   └── meals.repository.ts
├── entities/
│   └── meal.entity.ts
├── dtos/                         ← DTOs REST (class-validator)
│   └── create-meal.dto.ts
├── graphql/                      ← Tipos GraphQL (ObjectType, InputType, Args)
│   ├── meal.type.ts
│   └── create-meal.input.ts
├── view-models/                  ← ViewModels (resposta REST)
│   └── meal.view-model.ts
└── meals.module.ts
```

> **Convenção:** Arquivos específicos de GraphQL ficam em `graphql/` dentro do módulo — `ObjectType`, `InputType` e `ArgsType`. O `Resolver` fica em `resolvers/`, paralelo à pasta `controllers/`, seguindo a mesma convenção do projeto.

### Registrando resolver no módulo

```typescript
// src/modules/meals/meals.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { MealsController } from './controllers/meals.controller';
import { MealsResolver } from './resolvers/meals.resolver';      // ← novo
import { CreateMealUseCase } from './use-cases/create-meal/create-meal.use-case';
import { GetMealHistoryUseCase } from './use-cases/get-meal-history/get-meal-history.use-case';
import { MealsRepository } from './repositories/meals.repository';
import { MealEntity } from './entities/meal.entity';

@Module({
  imports: [TypeOrmModule.forFeature([MealEntity])],
  controllers: [MealsController],         // ← REST
  providers: [
    MealsResolver,                        // ← GraphQL
    CreateMealUseCase,
    GetMealHistoryUseCase,
    MealsRepository,
  ],
})
export class MealsModule {}
```

---

## 6. Definindo Tipos GraphQL (Code-First)

### 6.1 `ObjectType` — Shape da resposta GraphQL

O `ObjectType` é o equivalente GraphQL do `ViewModel` no REST. Ele define o que o cliente GraphQL pode consultar.

```typescript
// src/modules/meals/graphql/meal.type.ts
import { ObjectType, Field, ID, Int, registerEnumType } from '@nestjs/graphql';

export enum MealModeGql {
  QUICK = 'quick',
  CUSTOM = 'custom',
}

// ← Registra o enum no schema GraphQL
registerEnumType(MealModeGql, {
  name: 'MealMode',
  description: 'Modo de registro da refeição',
});

@ObjectType({ description: 'Refeição registrada pelo usuário' })
export class MealType {
  @Field(() => ID, { description: 'ID único da refeição (UUID v4)' })
  id: string;

  @Field(() => Int, { description: 'Número de blocos nutricionais' })
  blocks: number;

  @Field(() => MealModeGql, { description: 'Modo de registro' })
  mode: MealModeGql;

  @Field({ description: 'Data/hora de criação (ISO 8601)' })
  createdAt: string;

  @Field({ nullable: true, description: 'Observações opcionais' })
  notes?: string;

  // ← Campo de relação resolvido por DataLoader (veja seção 8)
  @Field(() => UserType, { description: 'Usuário dono da refeição' })
  user?: UserType; // resolvido pelo @ResolveField no Resolver
}
```

> **Regra:** O `ObjectType` é o contrato de saída do Resolver — equivalente ao `ViewModel` no REST. Não use a `Entity` diretamente como tipo GraphQL (mesmas razões que no REST: expõe campos internos do banco).

### 6.2 `InputType` — Payload de mutação

```typescript
// src/modules/meals/graphql/create-meal.input.ts
import { InputType, Field, Int } from '@nestjs/graphql';
import { IsEnum, IsInt, IsOptional, Min, Max, MaxLength } from 'class-validator';
import { MealModeGql } from './meal.type';

@InputType({ description: 'Dados para registrar uma nova refeição' })
export class CreateMealInput {
  @Field(() => Int, { description: 'Número de blocos (1–20)' })
  @IsInt()
  @Min(1)
  @Max(20)
  blocks: number;

  @Field(() => MealModeGql)
  @IsEnum(MealModeGql)
  mode: MealModeGql;

  @Field({ nullable: true, description: 'Observações (máx. 500 caracteres)' })
  @IsOptional()
  @MaxLength(500)
  notes?: string;
}
```

> **`InputType` vs `ArgsType`:** Use `@InputType()` quando o argumento é um objeto composto (ex: `createMeal(input: CreateMealInput)`). Use `@ArgsType()` quando os argumentos são campos planos passados diretamente (ex: `meal(id: ID!)`). **Prefira `InputType` para mutations** — é mais fácil de versionar e evoluir.

---

## 7. Estrutura de Resolvers por Domínio

### 7.1 Resolver como Controller GraphQL

O Resolver segue exatamente as mesmas regras do Controller REST definidas em `clean-architecture.md`:
- **Delega ao UseCase** — sem lógica de negócio no Resolver
- **Mapeia para ObjectType** — nunca retorna Entity diretamente
- **Usa o mesmo UseCase** que o Controller REST

```typescript
// src/modules/meals/resolvers/meals.resolver.ts
import {
  Resolver,
  Query,
  Mutation,
  Args,
  Context,
  ID,
} from '@nestjs/graphql';
import { UseGuards } from '@nestjs/common';
import { GqlAuthGuard } from '@common/guards/gql-auth.guard';
import { CurrentUser } from '@common/decorators/current-user.decorator';
import { UserPayload } from '@common/types/user-payload.type';

import { MealType } from '../graphql/meal.type';
import { CreateMealInput } from '../graphql/create-meal.input';
import { CreateMealUseCase } from '../use-cases/create-meal/create-meal.use-case';
import { GetMealHistoryUseCase } from '../use-cases/get-meal-history/get-meal-history.use-case';
import { MealEntity } from '../entities/meal.entity';

// ← @Resolver() alinhado com o ObjectType principal do domínio
@Resolver(() => MealType)
@UseGuards(GqlAuthGuard)  // ← guard equivalente ao JwtAuthGuard do REST
export class MealsResolver {
  constructor(
    private readonly createMealUseCase: CreateMealUseCase,
    private readonly getMealHistoryUseCase: GetMealHistoryUseCase,
  ) {}

  // ─── Queries ─────────────────────────────────────────────────────────────

  @Query(() => [MealType], { name: 'myMeals', description: 'Histórico de refeições do usuário autenticado' })
  async myMeals(
    @CurrentUser() user: UserPayload,
  ): Promise<MealType[]> {
    const result = await this.getMealHistoryUseCase.execute({ userId: user.id });
    // ← Mapeia Entity[] → ObjectType[] (equivalente ao MealViewModel.fromEntities())
    return result.items.map(MealType.fromEntity);
  }

  // ─── Mutations ────────────────────────────────────────────────────────────

  @Mutation(() => MealType, { name: 'createMeal', description: 'Registra uma nova refeição' })
  async createMeal(
    @Args('input') input: CreateMealInput,
    @CurrentUser() user: UserPayload,
  ): Promise<MealType> {
    const meal = await this.createMealUseCase.execute({
      ...input,
      userId: user.id,  // ← enriquece com dados do contexto (JWT), igual ao Controller REST
    });
    return MealType.fromEntity(meal);
  }
}
```

### 7.2 Estático `fromEntity` no `ObjectType`

Para manter o padrão de mapeamento consistente com o `ViewModel` do REST:

```typescript
// src/modules/meals/graphql/meal.type.ts (complemento)
@ObjectType()
export class MealType {
  // ... campos @Field ...

  // ← Mesmo padrão do ViewModel REST: MealViewModel.fromEntity()
  static fromEntity(entity: MealEntity): MealType {
    const type = new MealType();
    type.id = entity.id;
    type.blocks = entity.blocks;
    type.mode = entity.mode as MealModeGql;
    type.createdAt = entity.createdAt.toISOString();
    type.notes = entity.notes;
    return type;
  }
}
```

### 7.3 Convenção de nomenclatura

Consistente com `nest-project-structure.md`:

| Item | Convenção | Exemplo |
|------|-----------|---------|
| Arquivo | kebab-case + `.resolver.ts` | `meals.resolver.ts` |
| Classe | PascalCase + `Resolver` | `MealsResolver` |
| Pasta | `resolvers/` dentro do módulo | `src/modules/meals/resolvers/` |
| ObjectType | PascalCase + `Type` | `MealType` |
| InputType | PascalCase + `Input` | `CreateMealInput` |
| ArgsType | PascalCase + `Args` | `PaginateMealsArgs` |

---

## 8. DataLoader — Resolvendo o Problema N+1

### O que é o problema N+1

Quando um campo de relação é resolvido por um `@ResolveField`, o GraphQL chama o resolver para cada item da lista separadamente:

```graphql
# Esta query dispara 1 + N queries ao banco (N+1):
query {
  myMeals {          # 1 query: buscar 20 refeições
    id
    user {           # 20 queries: buscar usuário de cada refeição separadamente
      name
    }
  }
}
```

### Solução: DataLoader com instância por request

O DataLoader **batcha** e **deduplica** as requisições dentro de um event loop tick, transformando N queries em 1.

#### 8.1 Criando o DataLoader

```typescript
// src/modules/users/data-loaders/user.loader.ts
import * as DataLoader from 'dataloader';
import { Injectable, Scope } from '@nestjs/common';
import { UsersRepository } from '@modules/users/repositories/users.repository';
import { UserEntity } from '@modules/users/entities/user.entity';

// ← Scope.REQUEST garante uma instância nova POR REQUISIÇÃO GraphQL
// Isso evita cache compartilhado entre usuários diferentes (segurança!)
@Injectable({ scope: Scope.REQUEST })
export class UserLoader {
  constructor(private readonly usersRepository: UsersRepository) {}

  // ← Um loader = uma relação (ex: carregar User por ID)
  readonly batchLoadUsers = new DataLoader<string, UserEntity>(
    async (userIds: readonly string[]) => {
      // 1. Busca todos de uma vez (1 query ao invés de N)
      const users = await this.usersRepository.findByIds([...userIds]);

      // 2. Cria um Map para lookup O(1)
      const userMap = new Map<string, UserEntity>(
        users.map((user) => [user.id, user]),
      );

      // ⚠️ CRÍTICO: retorna na MESMA ORDEM dos IDs de entrada
      // DataLoader exige isso para mapear corretamente os resultados
      return userIds.map((id) => userMap.get(id) ?? new Error(`Usuário ${id} não encontrado`));
    },
    {
      // ← Caching por request (já garantido por Scope.REQUEST, mas explícito aqui)
      cache: true,
    },
  );
}
```

#### 8.2 Usando DataLoader no `@ResolveField`

```typescript
// src/modules/meals/resolvers/meals.resolver.ts (complemento)
import { Resolver, ResolveField, Parent } from '@nestjs/graphql';
import { UserLoader } from '@modules/users/data-loaders/user.loader';
import { UserType } from '@modules/users/graphql/user.type';
import { MealType } from '../graphql/meal.type';

@Resolver(() => MealType)
export class MealsResolver {
  constructor(
    // ... use cases ...
    private readonly userLoader: UserLoader,  // ← injetado via DI
  ) {}

  // ← @ResolveField resolve o campo 'user' de MealType
  // Chamado para cada item da lista — DataLoader faz o batching automaticamente
  @ResolveField(() => UserType, { name: 'user' })
  async resolveUser(
    @Parent() meal: MealType,  // ← o MealType pai
  ): Promise<UserType> {
    const user = await this.userLoader.batchLoadUsers.load(meal.userId);
    return UserType.fromEntity(user as UserEntity);
  }
}
```

#### 8.3 Registrando o DataLoader no módulo

```typescript
// src/modules/users/users.module.ts
import { Module } from '@nestjs/common';
import { UserLoader } from './data-loaders/user.loader';

@Module({
  providers: [
    // ... use cases e outros providers ...
    UserLoader,   // ← registrado com Scope.REQUEST (declarado na classe)
  ],
  exports: [
    UserLoader,   // ← exportado para ser injetado em outros módulos (ex: MealsResolver)
  ],
})
export class UsersModule {}
```

```typescript
// src/modules/meals/meals.module.ts
import { Module } from '@nestjs/common';
import { UsersModule } from '@modules/users/users.module'; // ← importa o módulo que exporta o loader

@Module({
  imports: [UsersModule],   // ← torna UserLoader disponível no MealsResolver
  providers: [MealsResolver, /* ... */],
})
export class MealsModule {}
```

### 8.4 Diagrama do fluxo com DataLoader

```
Query GraphQL: myMeals { user { name } }
      ↓
MealsResolver.myMeals()         → 1 query: SELECT * FROM meals WHERE userId = ?
      ↓ (para cada MealType)
MealsResolver.resolveUser()     → userLoader.batchLoadUsers.load(meal.userId)
                                   ← DataLoader coleta todos os IDs do tick atual
                                   → 1 query: SELECT * FROM users WHERE id IN (ids...)
                                   ← mapeia e retorna para cada resolver
```

**Resultado:** 2 queries totais, independente de quantas refeições foram retornadas.

---

## 9. Guard GraphQL — Autenticação com JWT

O `JwtAuthGuard` do REST não funciona diretamente no contexto GraphQL porque o `ExecutionContext` é diferente. Crie um guard que detecta o contexto:

```typescript
// src/common/guards/gql-auth.guard.ts
import { ExecutionContext, Injectable } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  // ← Sobrescreve getRequest() para extrair req do contexto GraphQL
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req;  // ← req injetado no context do GraphQLModule
  }
}
```

> **Por que um guard separado?** O `GqlAuthGuard` reutiliza a mesma estratégia Passport JWT (`AuthGuard('jwt')`), apenas adaptando como extrai o objeto `request` do contexto de execução. **A validação JWT é a mesma** — não há duplicação de lógica.

---

## 10. Tratamento de Erros no GraphQL

### 10.1 Erros GraphQL padrão

No GraphQL, erros são retornados no campo `errors` da resposta, nunca como HTTP status codes (a resposta HTTP sempre é `200 OK`). Use as exceções NestJS normalmente nos UseCases — o `@nestjs/graphql` as converte automaticamente para o formato GraphQL.

```typescript
// UseCase: mesmo padrão do REST — exceções do @nestjs/common
throw new NotFoundException('Refeição não encontrada');
// → response: { data: null, errors: [{ message: "Refeição não encontrada", extensions: { code: "NOT_FOUND" } }] }
```

### 10.2 Erros customizados com código semântico

Para erros de negócio que precisam de código customizado:

```typescript
// src/common/errors/business.error.ts
import { GraphQLError } from 'graphql';

// Extensão de GraphQLError para erros de negócio
export class BusinessError extends GraphQLError {
  constructor(message: string, code: string) {
    super(message, {
      extensions: {
        code,        // ex: 'DAILY_LIMIT_EXCEEDED', 'MEAL_LOCKED'
      },
    });
  }
}

// Uso no UseCase:
throw new BusinessError('Limite diário de refeições atingido', 'DAILY_LIMIT_EXCEEDED');
```

> **Regra:** Use as exceções do `@nestjs/common` (NotFoundException, ForbiddenException, etc.) para erros estruturais. Use `BusinessError` apenas para erros de negócio que precisam de código semântico customizado — e apenas quando o cliente GraphQL precisa tratar esse código específico.

---

## 11. Paginação GraphQL — Connections (Relay-style)

Para listas paginadas no GraphQL, o padrão da indústria é **Relay Connections** (cursor-based). Isso é consistente com a paginação por cursor já definida em `rest-api-patterns.md`.

```typescript
// src/common/graphql/pagination.type.ts
import { ObjectType, Field, Int } from '@nestjs/graphql';

// ← Função genérica para gerar tipos de Connection
export function Paginated<T>(classRef: Type<T>) {
  @ObjectType(`${classRef.name}Edge`)
  abstract class EdgeType {
    @Field()
    cursor: string;

    @Field(() => classRef)
    node: T;
  }

  @ObjectType({ isAbstract: true })
  abstract class PaginatedType {
    @Field(() => [EdgeType])
    edges: EdgeType[];

    @Field(() => Int)
    totalCount: number;

    @Field()
    hasNextPage: boolean;

    @Field({ nullable: true })
    nextCursor?: string;
  }

  return PaginatedType;
}

// Uso para o domínio de refeições:
// @ObjectType() export class PaginatedMeals extends Paginated(MealType) {}
```

---

## 12. Estrutura de Pastas Resumida

```
src/
├── config/
│   └── graphql.config.ts          ← configuração ApolloDriverConfig
│
└── modules/
    └── meals/
        ├── controllers/
        │   └── meals.controller.ts     ← REST transport (inalterado)
        ├── resolvers/
        │   └── meals.resolver.ts       ← GraphQL transport
        ├── graphql/
        │   ├── meal.type.ts            ← @ObjectType (shape de saída GraphQL)
        │   └── create-meal.input.ts    ← @InputType (payload de mutation)
        ├── data-loaders/
        │   └── meal.loader.ts          ← DataLoader com Scope.REQUEST
        ├── use-cases/                  ← compartilhados entre REST e GraphQL
        │   └── ...
        └── meals.module.ts
```

---

## 13. Anti-Padrões Comuns

### ❌ Lógica de negócio no Resolver

```typescript
// ❌ ERRADO — regra de negócio no Resolver
@Mutation(() => MealType)
async createMeal(@Args('input') input: CreateMealInput) {
  if (input.blocks <= 0) throw new Error('Blocos inválidos'); // ← no UseCase
  const meal = await this.mealsRepository.create(input);      // ← repositório direto
  return meal;
}

// ✅ CORRETO — Resolver delega ao UseCase
@Mutation(() => MealType)
async createMeal(@Args('input') input: CreateMealInput, @CurrentUser() user: UserPayload) {
  const meal = await this.createMealUseCase.execute({ ...input, userId: user.id });
  return MealType.fromEntity(meal);
}
```

### ❌ Entity retornada diretamente no Resolver

```typescript
// ❌ ERRADO — Entity como ObjectType GraphQL
@Query(() => MealEntity)  // ← expõe campos internos do banco
async meal(@Args('id') id: string): Promise<MealEntity> { ... }

// ✅ CORRETO — ObjectType dedicado
@Query(() => MealType)
async meal(@Args('id') id: string): Promise<MealType> {
  const entity = await this.getMealUseCase.execute({ id });
  return MealType.fromEntity(entity);
}
```

### ❌ DataLoader com Scope singleton

```typescript
// ❌ ERRADO — singleton: cache compartilhado entre requests (vazamento de dados entre usuários!)
@Injectable() // ← Scope.DEFAULT = singleton
export class UserLoader { ... }

// ✅ CORRETO — uma instância por request
@Injectable({ scope: Scope.REQUEST })
export class UserLoader { ... }
```

### ❌ DataLoader retornando em ordem incorreta

```typescript
// ❌ ERRADO — retorno sem garantia de ordem
async (userIds) => {
  const users = await this.repo.findByIds([...userIds]);
  return users; // ← ordem pode diferir dos IDs de entrada
}

// ✅ CORRETO — retorno na mesma ordem dos IDs de entrada
async (userIds) => {
  const users = await this.repo.findByIds([...userIds]);
  const map = new Map(users.map((u) => [u.id, u]));
  return userIds.map((id) => map.get(id) ?? new Error(`User ${id} not found`));
}
```

### ❌ Dois GraphQLModule no mesmo projeto

```typescript
// ❌ ERRADO — registrar GraphQLModule em mais de um lugar
@Module({ imports: [GraphQLModule.forRoot(...)] })
class MealsModule {} // ← GraphQLModule deve ser global, apenas no AppModule

// ✅ CORRETO — um único GraphQLModule no AppModule
@Module({ imports: [GraphQLModule.forRoot(...), MealsModule] })
class AppModule {}
```

### ❌ Ignorar `context({ req })` no GraphQLModule

```typescript
// ❌ ERRADO — sem context, o GqlAuthGuard não consegue acessar o request
GraphQLModule.forRoot({ driver: ApolloDriver, autoSchemaFile: true })

// ✅ CORRETO — req disponível no contexto para o GqlAuthGuard
GraphQLModule.forRoot({
  driver: ApolloDriver,
  autoSchemaFile: true,
  context: ({ req }) => ({ req }), // ← obrigatório para autenticação JWT
})
```

---

## 14. Checklist de Implementação

### GraphQLModule
- [ ] `graphql.config.ts` em `src/config/`?
- [ ] `GraphQLModule.forRoot()` registrado **apenas** no `AppModule`?
- [ ] `context: ({ req }) => ({ req })` configurado (para autenticação)?
- [ ] `playground` condicional ao ambiente (`process.env.NODE_ENV !== 'production'`)?
- [ ] `formatError` configurado para não expor stack em produção?

### Resolver
- [ ] Arquivo em `resolvers/` dentro do módulo, com sufixo `.resolver.ts`?
- [ ] Classe com `@Resolver(() => MealType)`?
- [ ] Sem lógica de negócio (delega ao UseCase)?
- [ ] Em vez de Entity, retorna `ObjectType` (com `fromEntity()`)?
- [ ] `@UseGuards(GqlAuthGuard)` aplicado?

### ObjectType / InputType
- [ ] `ObjectType` em `graphql/<domínio>.type.ts`?
- [ ] `InputType` em `graphql/<domínio>.input.ts`?
- [ ] Todos os campos têm `@Field()` com `description`?
- [ ] Enums registrados com `registerEnumType()`?
- [ ] `fromEntity()` estático definido no `ObjectType`?

### DataLoader
- [ ] `@Injectable({ scope: Scope.REQUEST })` declarado?
- [ ] Batch function retorna na mesma **ordem** dos IDs de entrada?
- [ ] Loader exportado pelo módulo e importado nos módulos que o usam?
- [ ] Campos de relação resolvidos com `@ResolveField` + DataLoader?

### Guard
- [ ] `GqlAuthGuard` em `src/common/guards/gql-auth.guard.ts`?
- [ ] Reutiliza `AuthGuard('jwt')` (mesma estratégia Passport do REST)?

---

## Referências

- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Controller fino, UseCase como camada de aplicação, Entity nunca exposta diretamente
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura de pastas, Use Cases Pattern, path aliases
- [nest-modules-pattern.md](.agent/skills/backend/foundation-and-architecture/nest-modules-pattern.md) — GraphQLModule global no AppModule, Scope.REQUEST para DataLoaders
- [rest-api-patterns.md](.agent/skills/backend/api-and-contracts/rest-api-patterns.md) — REST coexiste com GraphQL, paginação por cursor (base para Relay Connections)
- [dto-validation.md](.agent/skills/backend/api-and-contracts/dto-validation.md) — class-validator em InputType funciona igual ao DTO REST
- [@nestjs/graphql — Documentação Oficial](https://docs.nestjs.com/graphql/quick-start)
- [DataLoader — GitHub](https://github.com/graphql/dataloader)
- [GraphQL Relay Connections spec](https://relay.dev/graphql/connections.htm)
