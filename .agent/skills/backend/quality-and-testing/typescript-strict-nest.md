---
name: typescript-strict-nest
description: Padrões TypeScript específicos para NestJS — uso de abstract classes como contratos de repositório injetáveis, interfaces TypeScript para contratos de tipo puro (sem DI), generics em Use Cases e serviços de CRUD base, discriminated unions para resultados de operações, e como tipar corretamente os decorators de NestJS (@Injectable, @Controller, @Param, pipes com generics). Complementa clean-architecture.md (regras de camada), nest-project-structure.md (estrutura de pastas) e typeorm-patterns.md / prisma-patterns.md (implementações concretas de repositório).
---

# TypeScript Estrito no NestJS — Contratos, Generics e Resultados

Você é um Engenheiro de Software Senior especialista em TypeScript avançado aplicado ao NestJS. Esta skill define **como tipar corretamente** cada camada da arquitetura estabelecida em `clean-architecture.md`.

> **Alinhamento com o frontend:** A skill `typescript-patterns.md` do React Native adota `type` como padrão e `interface` apenas para contratos de classes OOP. Esta skill **não conflita** — ela aplica exatamente essa mesma regra no contexto NestJS, onde repositórios são classes que implementam contratos OOP. A diferença é o mecanismo de DI do NestJS que exige `abstract class` no lugar de `interface` para injeção sem token explícito.

---

## Quando usar esta skill

- **Criar contrato de repositório:** Decidir entre `interface` + token Symbol vs `abstract class`.
- **Criar Use Case genérico:** Tipar `execute()` com generics de entrada e saída.
- **Tipar resultado de operação:** Modelar `OperationResult<T>` com discriminated union.
- **Tipar decorators:** `@Param`, `@Body`, `@Query`, `@CurrentUser`, `ParseUUIDPipe`.
- **Code review:** Validar se a tipagem segue os padrões do projeto.
- **Criar CRUD base genérico:** Construir classe abstrata reutilizável entre múltiplos módulos.

---

## 1. Contratos de Repositório — `abstract class` vs `interface`

### O Problema: Interfaces desaparecem em runtime

TypeScript `interface` é apagada na transpilação para JavaScript. O sistema de DI do NestJS opera em runtime usando metadados — ele **não consegue resolver** uma `interface` como token de injeção sem configuração adicional.

```typescript
// ❌ ERRADO — interface não pode ser usada diretamente como token de DI
export interface IUsersRepository {
  findById(id: string): Promise<UserEntity | null>;
}

@Injectable()
export class GetUserUseCase {
  // Falha em runtime: NestJS não consegue resolver IUsersRepository
  constructor(private readonly repo: IUsersRepository) {}
}
```

### Solução Adotada: `abstract class` como contrato injetável

`abstract class` existe em runtime como um objeto JavaScript real. Pode ser usado como token de DI **sem necessidade de `@Inject()`**, mantendo a sintaxe de constructor injection limpa.

```typescript
// src/modules/users/repositories/users.repository.abstract.ts
export abstract class UsersRepository {
  abstract findById(id: string): Promise<UserEntity | null>;
  abstract findByEmail(email: string): Promise<UserEntity | null>;
  abstract create(data: CreateUserData): Promise<UserEntity>;
  abstract updateById(id: string, data: UpdateUserData): Promise<UserEntity | null>;
  abstract deleteById(id: string): Promise<void>;
}
```

```typescript
// src/modules/users/repositories/users.typeorm.repository.ts
@Injectable()
export class UsersTypeOrmRepository extends UsersRepository {
  constructor(
    @InjectRepository(UserEntity)
    private readonly repo: Repository<UserEntity>,
  ) {
    super();
  }

  async findById(id: string): Promise<UserEntity | null> {
    return this.repo.findOne({ where: { id } });
  }
  // ... implementa todos os métodos abstratos
}
```

```typescript
// src/modules/users/users.module.ts
@Module({
  providers: [
    {
      provide: UsersRepository,      // ← abstract class como token
      useClass: UsersTypeOrmRepository,
    },
  ],
  exports: [UsersRepository],
})
export class UsersModule {}
```

```typescript
// src/modules/users/use-cases/get-user/get-user.use-case.ts
@Injectable()
export class GetUserUseCase {
  // ✅ Sem @Inject() — abstract class é resolvida automaticamente
  constructor(private readonly usersRepository: UsersRepository) {}

  async execute(id: string): Promise<UserEntity> {
    const user = await this.usersRepository.findById(id);
    if (!user) throw new NotFoundException(`Usuário ${id} não encontrado`);
    return user;
  }
}
```

### Quando usar `interface` pura (sem DI)

`interface` é preferida quando o contrato é apenas de **tipagem**, sem envolver injeção de dependência. Exemplos: tipos de input/output de Use Cases, contratos de resposta, tipos de configuração.

```typescript
// src/modules/users/types/users.types.ts
// ✅ interface para contrato de tipo puro — não é injetado pelo DI

export interface CreateUserInput {
  email: string;
  password: string;
  displayName?: string;
}

export interface UpdateUserInput {
  displayName?: string;
  status?: UserStatus;
}

export interface FindUsersFilter {
  status?: UserStatus;
  createdAfter?: Date;
  limit?: number;
  offset?: number;
}
```

### Tabela de decisão

| Cenário | Usar | Motivo |
|---------|------|--------|
| Contrato de repositório (injetado via DI) | `abstract class` | Existe em runtime; token de DI sem `@Inject()` |
| Contrato de serviço externo (mail, storage) | `abstract class` | Mesmo motivo — troca de implementação via módulo |
| Tipo de input de Use Case | `interface` | Tipagem pura; nunca é injetado |
| Tipo de output / resultado | `type` ou `interface` | Tipagem pura |
| Augmentar tipos de terceiros | `interface` | Declaration merging |
| União de tipos / aliases | `type` | `interface` não suporta unions |

---

## 2. Generics em Use Cases e CRUD Base

### 2.1 Use Case Base — Contrato genérico

Todos os Use Cases expõem um único método `execute()`. O tipo de entrada e saída é definido pelos generics do módulo concreto.

```typescript
// src/common/use-cases/base.use-case.ts
export abstract class BaseUseCase<TInput, TOutput> {
  abstract execute(input: TInput): Promise<TOutput>;
}
```

```typescript
// src/modules/users/use-cases/create-user/create-user.use-case.ts
import { BaseUseCase } from '@common/use-cases/base.use-case';
import { CreateUserInput } from '@modules/users/types/users.types';

@Injectable()
export class CreateUserUseCase extends BaseUseCase<CreateUserInput, UserEntity> {
  constructor(
    private readonly usersRepository: UsersRepository,
    private readonly hashService: HashService,
  ) {
    super();
  }

  async execute(input: CreateUserInput): Promise<UserEntity> {
    const existingUser = await this.usersRepository.findByEmail(input.email);
    if (existingUser) throw new ConflictException('E-mail já cadastrado');

    const passwordHash = await this.hashService.hash(input.password);
    return this.usersRepository.create({ ...input, passwordHash });
  }
}
```

### 2.2 Repository Base — Abstração genérica para CRUD padrão

Para módulos com CRUD padrão, um repositório base genérico evita duplicação sem sacrificar type safety.

```typescript
// src/common/repositories/base.repository.abstract.ts
export abstract class BaseRepository<TEntity, TCreateData, TUpdateData = Partial<TCreateData>> {
  abstract findById(id: string): Promise<TEntity | null>;
  abstract findAll(filter?: Record<string, unknown>): Promise<TEntity[]>;
  abstract create(data: TCreateData): Promise<TEntity>;
  abstract updateById(id: string, data: TUpdateData): Promise<TEntity | null>;
  abstract deleteById(id: string): Promise<void>;
}
```

```typescript
// src/modules/categories/repositories/categories.repository.abstract.ts
import { BaseRepository } from '@common/repositories/base.repository.abstract';

// Especializa o BaseRepository para o domínio Categories
export abstract class CategoriesRepository extends BaseRepository<
  CategoryEntity,
  CreateCategoryData,
  UpdateCategoryData
> {
  // Métodos específicos do domínio além dos herdados do CRUD base
  abstract findBySlug(slug: string): Promise<CategoryEntity | null>;
  abstract findAllActive(): Promise<CategoryEntity[]>;
}
```

> **Regra:** O `BaseRepository` genérico só deve conter operações verdadeiramente universais (findById, findAll, create, updateById, deleteById). Métodos específicos de domínio (findByEmail, findBySlug, etc.) pertencem à `abstract class` do módulo que estende o base.

### 2.3 Generics com constraints — evitar `any`

```typescript
// ✅ CORRETO — constraint garante que TEntity tem id: string
abstract class BaseRepository<TEntity extends { id: string }, TCreateData> {
  abstract findById(id: TEntity['id']): Promise<TEntity | null>;
  abstract create(data: TCreateData): Promise<TEntity>;
}

// ✅ CORRETO — constraint em Use Case garante compatibilidade
abstract class BaseUseCase<
  TInput extends Record<string, unknown>,
  TOutput
> {
  abstract execute(input: TInput): Promise<TOutput>;
}

// ❌ ERRADO — any destrói o type safety
abstract class BaseRepository<TEntity = any, TCreateData = any> { ... }
```

---

## 3. Discriminated Unions para Resultados de Operações

### 3.1 Quando usar `OperationResult<T>` vs exceções NestJS

O NestJS já tem um mecanismo robusto de exceções HTTP (`NotFoundException`, `ConflictException`, etc.) que são capturadas pelo `HttpExceptionFilter`. Para **a maioria dos Use Cases**, lançar exceções é a abordagem correta e idiomática.

Use `OperationResult<T>` apenas quando:
- O Use Case pode retornar múltiplos estados de sucesso com dados diferentes (ex: resultado de validação com detalhes).
- A lógica de negócio precisa tomar decisões baseadas no tipo de resultado antes de lançar exceção.
- Integrações internas entre Use Cases onde você quer evitar o custo de `try/catch`.

```typescript
// src/common/types/operation-result.ts

type SuccessResult<T> = {
  readonly ok: true;
  readonly data: T;
};

type FailureResult = {
  readonly ok: false;
  readonly error: {
    readonly code: string;
    readonly message: string;
  };
};

export type OperationResult<T> = SuccessResult<T> | FailureResult;

// Factories — eliminam construção manual
export const Result = {
  ok: <T>(data: T): SuccessResult<T> => ({ ok: true, data }),
  fail: (code: string, message: string): FailureResult => ({
    ok: false,
    error: { code, message },
  }),
} as const;
```

```typescript
// Uso em Use Case que retorna resultado tipado
@Injectable()
export class ValidatePromoCodeUseCase {
  constructor(private readonly promoCodesRepository: PromoCodesRepository) {}

  async execute(code: string, userId: string): Promise<OperationResult<PromoCodeDiscount>> {
    const promo = await this.promoCodesRepository.findByCode(code);

    if (!promo) {
      return Result.fail('PROMO_NOT_FOUND', `Código ${code} não encontrado`);
    }
    if (promo.expiresAt < new Date()) {
      return Result.fail('PROMO_EXPIRED', `Código ${code} expirado`);
    }
    if (promo.usedBy.includes(userId)) {
      return Result.fail('PROMO_ALREADY_USED', 'Código já utilizado por este usuário');
    }

    return Result.ok({ discountPercent: promo.discountPercent, promoId: promo.id });
  }
}
```

```typescript
// Controller consome o resultado e decide o status HTTP
@Post('apply')
async applyPromoCode(
  @Body() dto: ApplyPromoCodeDto,
  @CurrentUser() user: UserPayload,
): Promise<ApplyPromoCodeViewModel> {
  const result = await this.validatePromoCodeUseCase.execute(dto.code, user.id);

  if (!result.ok) {
    // Narrowing automático — result.error está disponível aqui
    switch (result.error.code) {
      case 'PROMO_NOT_FOUND':
        throw new NotFoundException(result.error.message);
      case 'PROMO_EXPIRED':
      case 'PROMO_ALREADY_USED':
        throw new BadRequestException(result.error.message);
      default: {
        // Exhaustiveness check — garante que todos os códigos são tratados
        const _exhaustive: never = result.error.code as never;
        throw new InternalServerErrorException();
      }
    }
  }

  // Narrowing automático — result.data está disponível aqui
  return ApplyPromoCodeViewModel.fromDomain(result.data);
}
```

### 3.2 Resultado com múltiplos tipos de sucesso

```typescript
// Discriminated union com múltiplos estados de sucesso (raro, mas válido)
type LoginResult =
  | { status: 'success'; accessToken: string; refreshToken: string }
  | { status: 'requires_2fa'; challengeToken: string }
  | { status: 'account_suspended'; reason: string };

@Injectable()
export class LoginUseCase {
  async execute(input: LoginInput): Promise<LoginResult> {
    const user = await this.usersRepository.findByEmail(input.email);
    if (!user) throw new UnauthorizedException('Credenciais inválidas');

    const validPassword = await this.hashService.compare(input.password, user.passwordHash);
    if (!validPassword) throw new UnauthorizedException('Credenciais inválidas');

    if (user.status === 'suspended') {
      return { status: 'account_suspended', reason: user.suspensionReason ?? 'Conta suspensa' };
    }
    if (user.twoFactorEnabled) {
      const challengeToken = await this.twoFactorService.createChallenge(user.id);
      return { status: 'requires_2fa', challengeToken };
    }

    const tokens = await this.tokenService.generate(user.id);
    return { status: 'success', ...tokens };
  }
}
```

---

## 4. Tipagem Correta de Decorators NestJS

### 4.1 `@Param`, `@Query`, `@Body` — sempre com pipes de transformação

```typescript
import {
  Controller, Get, Post, Delete,
  Param, Query, Body,
  ParseUUIDPipe, ParseIntPipe, ParseEnumPipe,
  HttpCode, HttpStatus,
} from '@nestjs/common';

@Controller({ path: 'users', version: '1' })
export class UsersController {
  // ✅ ParseUUIDPipe garante que id é um UUID válido E converte para string
  @Get(':id')
  async getUser(@Param('id', ParseUUIDPipe) id: string): Promise<UserViewModel> {
    // id é string — TypeScript e runtime validados
    return this.getUserUseCase.execute(id);
  }

  // ✅ @Query com tipo explícito após transformação pelo ValidationPipe global
  @Get()
  async listUsers(@Query() query: ListUsersQueryDto): Promise<PaginatedResponse<UserViewModel>> {
    // query.page e query.limit já são number (transformados pelo ValidationPipe com transform: true)
    return this.listUsersUseCase.execute(query);
  }

  // ✅ @Body tipado com DTO — validado pelo ValidationPipe global
  @Post()
  @HttpCode(HttpStatus.CREATED)
  async createUser(@Body() dto: CreateUserDto): Promise<UserViewModel> {
    const user = await this.createUserUseCase.execute(dto);
    return UserViewModel.fromEntity(user);
  }

  // ✅ Múltiplos @Param em rota aninhada
  @Delete(':userId/sessions/:sessionId')
  @HttpCode(HttpStatus.NO_CONTENT)
  async deleteSession(
    @Param('userId', ParseUUIDPipe) userId: string,
    @Param('sessionId', ParseUUIDPipe) sessionId: string,
  ): Promise<void> {
    await this.deleteSessionUseCase.execute({ userId, sessionId });
  }
}
```

### 4.2 Decorator customizado `@CurrentUser` com tipagem

```typescript
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import type { Request } from 'express';

// Tipo do payload JWT que a Guard injeta em req.user
export type UserPayload = {
  readonly sub: string;      // userId
  readonly email: string;
  readonly role: UserRole;
};

export const CurrentUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): UserPayload => {
    const request = ctx.switchToHttp().getRequest<Request & { user: UserPayload }>();
    return request.user;
  },
);
```

```typescript
// Uso no controller — tipagem completa sem cast
@Get('me')
async getMe(@CurrentUser() user: UserPayload): Promise<UserViewModel> {
  return this.getUserUseCase.execute(user.sub);
}
```

### 4.3 Guards tipados com `canActivate` genérico

```typescript
// src/common/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import type { Observable } from 'rxjs';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  // Tipo explícito do retorno — boolean | Promise<boolean> | Observable<boolean>
  canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean> {
    return super.canActivate(context);
  }
}
```

### 4.4 Interceptors com tipagem genérica

```typescript
// src/common/interceptors/transform.interceptor.ts
import {
  Injectable, NestInterceptor,
  ExecutionContext, CallHandler,
} from '@nestjs/common';
import type { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

// Tipo do envelope de resposta
export type ApiResponse<T> = {
  data: T;
  meta?: Record<string, unknown>;
};

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, ApiResponse<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<ApiResponse<T>> {
    return next.handle().pipe(
      map((data) => ({ data })),
    );
  }
}
```

### 4.5 Pipes customizados com generics

```typescript
// src/common/pipes/parse-positive-int.pipe.ts
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParsePositiveIntPipe implements PipeTransform<string, number> {
  // PipeTransform<TInput, TOutput> — ambos tipados
  transform(value: string): number {
    const parsed = parseInt(value, 10);
    if (isNaN(parsed) || parsed <= 0) {
      throw new BadRequestException(`Valor deve ser um inteiro positivo, recebido: ${value}`);
    }
    return parsed;
  }
}
```

---

## 5. Utility Types para o Contexto NestJS

```typescript
// src/common/types/nestjs.types.ts

// Tipo para paginação — alinha com rest-api-patterns.md
export type PaginatedResponse<T> = {
  data: T[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  };
};

// Extrai o tipo de retorno do método execute() de um UseCase
export type UseCaseOutput<T extends { execute: (...args: any[]) => Promise<any> }> =
  Awaited<ReturnType<T['execute']>>;

// Torna campos readonly em profundidade (útil para ViewModels imutáveis)
export type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

// Extrai as chaves de uma entity que são strings (para usar como filtros de busca)
export type StringKeys<T> = {
  [K in keyof T]: T[K] extends string ? K : never;
}[keyof T];
```

---

## 6. tsconfig — Configuração Estrita Obrigatória

```jsonc
// tsconfig.json — complementa nest-project-structure.md
{
  "compilerOptions": {
    "strict": true,                        // habilita todos os checks estritos
    "strictNullChecks": true,              // null e undefined são tipos distintos
    "noImplicitAny": true,                 // proibido any implícito
    "noImplicitReturns": true,             // toda função deve retornar explicitamente
    "noFallthroughCasesInSwitch": true,    // switch deve ter break/return em todos os cases
    "exactOptionalPropertyTypes": false,   // false — mais ergonômico com class-validator
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "experimentalDecorators": true,        // necessário para @Injectable, @Column, etc.
    "emitDecoratorMetadata": true          // necessário para DI refletir tipos em runtime
  }
}
```

> **`emitDecoratorMetadata: true` é crítico.** Sem ele, o sistema de DI do NestJS não consegue refletir os tipos dos construtores em runtime e não saberá qual implementação injetar. Nunca remova esta opção.

---

## Checklist de Verificação

### Contratos de Repositório
- [ ] Repositório injetável usa `abstract class`, não `interface`?
- [ ] A `abstract class` do repositório está em `repositories/<nome>.repository.abstract.ts`?
- [ ] O módulo registra `{ provide: NomeRepository, useClass: NomeTypeOrmRepository }`?
- [ ] Use Cases recebem a `abstract class` no construtor sem `@Inject()`?

### Generics
- [ ] Use Cases genéricos usam constraint `extends` nos type params?
- [ ] `BaseRepository<TEntity extends { id: string }, TCreateData>` — constraint presente?
- [ ] Zero usos de `any` — substituído por `unknown` + narrowing ou generic com constraint?

### Discriminated Unions
- [ ] `OperationResult<T>` é usado apenas quando há múltiplos estados de retorno com dados distintos?
- [ ] Use Cases simples lançam exceções NestJS diretamente (sem `OperationResult`)?
- [ ] `switch` sobre discriminated unions tem exhaustiveness check com `never`?
- [ ] Controller faz narrowing de `result.ok` antes de acessar `result.data` ou `result.error`?

### Decorators
- [ ] `@Param` usa `ParseUUIDPipe` para parâmetros de ID?
- [ ] `@Query` usa DTO com `class-validator` + `ValidationPipe` global com `transform: true`?
- [ ] `@CurrentUser()` retorna `UserPayload` sem `as` cast?
- [ ] `Interceptors` implementam `NestInterceptor<T, ApiResponse<T>>` com generics?
- [ ] `emitDecoratorMetadata: true` no `tsconfig.json`?

### Configuração
- [ ] `strict: true` habilitado no `tsconfig.json`?
- [ ] Zero `@ts-ignore` ou `@ts-expect-error` sem comentário explicativo?
- [ ] `import type` usado para importações que são apenas tipos?

---

## Referências

- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Regras de camada; Use Cases recebem `abstract class` do repositório
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Localização de `types/`, `repositories/`, `common/`
- [typeorm-patterns.md](.agent/skills/backend/database/typeorm-patterns.md) — Implementação concreta do repositório (`extends UsersRepository`)
- [prisma-patterns.md](.agent/skills/backend/database/prisma-patterns.md) — Alternativa de implementação concreta
- [rest-api-patterns.md](.agent/skills/backend/api-and-contracts/rest-api-patterns.md) — `TransformInterceptor`, `HttpExceptionFilter`, `PaginatedResponse`
- [typescript-patterns.md](.agent/skills/frontend/code-quality/typescript-patterns.md) — Padrões equivalentes no React Native (type vs interface, discriminated unions)
- [NestJS Docs — Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
- [NestJS Docs — Pipes](https://docs.nestjs.com/pipes)
- [NestJS Docs — Interceptors](https://docs.nestjs.com/interceptors)
- [TypeScript Docs — Abstract Classes](https://www.typescriptlang.org/docs/handbook/2/classes.html#abstract-classes-and-members)