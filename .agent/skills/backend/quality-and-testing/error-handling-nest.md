---
name: error-handling-nest
description: Tratamento centralizado de erros no NestJS — hierarquia de exceções customizadas (DomainException, BusinessException, exceções específicas de domínio), ExceptionFilter global que captura HttpException + erros de banco (TypeORM e Prisma), mapeamento de erros de infraestrutura para HTTP, e integração com o envelope de erro { error } já definido em rest-api-patterns.md. Consultar esta skill sempre que: criar uma nova classe de exceção, precisar mapear erros de banco para HTTP, ou definir um novo código de erro semântico de negócio.
---

# Tratamento de Erros — NestJS

Você é um Arquiteto de Software Senior. Esta skill define **como estruturar, lançar e capturar erros** em toda a aplicação NestJS. Ela é diretamente dependente de:

- `rest-api-patterns.md` — o `HttpExceptionFilter` global e o envelope `{ error: { code, message, details, path, timestamp } }` já estão definidos lá. **Esta skill não redefine o filter — ela o expande e fornece o contexto de exceções que ele vai capturar.**
- `clean-architecture.md` — exceções de domínio nascem nos UseCases; o Controller nunca constrói erros de negócio.
- `nest-project-structure.md` — onde ficam os arquivos de exceção e filtros no projeto.

> **Regra fundamental:** Cada camada lança o tipo certo de exceção. O `ExceptionFilter` global é o único responsável por transformar qualquer exceção em resposta HTTP — nenhum outro ponto do código deve construir objetos de erro diretamente.

---

## 1. Categorias de Erros e Onde Cada Uma Nasce

### Tabela de responsabilidade por camada

| Camada | O que pode lançar | Tipo de exceção |
|--------|-------------------|-----------------|
| **DTO / ValidationPipe** | Payload inválido (campos faltando, tipo errado) | `BadRequestException` (automático) |
| **Guard / Interceptor** | Token inválido, sem permissão | `UnauthorizedException`, `ForbiddenException` |
| **UseCase** | Regras de negócio violadas | `BusinessException` e subclasses |
| **Repository** | Erros de banco mapeados para HTTP | Relança como `ConflictException`, `NotFoundException`, etc. |
| **ExceptionFilter** | Qualquer exceção não tratada | Captura e formata como `{ error }` |

> **Regra:** O Controller **nunca** lança exceções de negócio. Ele delega ao UseCase. Se o UseCase lançar, o Filter captura.

---

## 2. Hierarquia de Exceções Customizadas

### 2.1 Estrutura de arquivos

```
src/common/exceptions/
├── domain.exception.ts        ← Classe base abstrata de todos os erros de domínio
├── business.exception.ts      ← Erros de regra de negócio (422)
├── resource-not-found.exception.ts  ← Recurso não encontrado (404)
├── resource-conflict.exception.ts   ← Conflito de dados (409)
├── resource-forbidden.exception.ts  ← Sem permissão de negócio (403)
└── index.ts                   ← Re-exporta tudo
```

### 2.2 `DomainException` — Classe Base

A raiz da hierarquia. Toda exceção de domínio deste projeto estende esta classe, o que permite ao `ExceptionFilter` distinguir erros de negócio de erros inesperados.

```typescript
// src/common/exceptions/domain.exception.ts
import { HttpException, HttpStatus } from '@nestjs/common';

/**
 * Classe base para todas as exceções de domínio do projeto.
 *
 * Estende HttpException para que o NestJS reconheça automaticamente
 * a exceção e o ExceptionFilter saiba o status HTTP correto.
 *
 * Regras:
 * - Nunca instancie DomainException diretamente — use uma subclasse.
 * - O campo `errorCode` é o código semântico retornado no envelope { error.code }.
 * - O campo `details` é opcional e deve ser usado apenas quando o cliente
 *   precisa tratar campos específicos (ex: erros de validação de negócio por campo).
 */
export abstract class DomainException extends HttpException {
  constructor(
    message: string,
    statusCode: HttpStatus,
    public readonly errorCode: string,
    public readonly details?: unknown,
  ) {
    super(
      {
        message,
        errorCode,
        details: details ?? undefined,
      },
      statusCode,
    );
  }
}
```

### 2.3 `BusinessException` — Regra de Negócio Violada (422)

Use quando a entrada é estruturalmente válida, mas viola uma regra de negócio. Equivale a "a requisição passou na validação do DTO, mas não pode ser processada semanticamente."

```typescript
// src/common/exceptions/business.exception.ts
import { HttpStatus } from '@nestjs/common';
import { DomainException } from './domain.exception';

/**
 * Exceção para regras de negócio violadas.
 * HTTP 422 Unprocessable Entity.
 *
 * Use quando:
 * - Limite diário de blocos atingido
 * - Tentativa de editar dieta travada por nutricionista
 * - Refeição incompleta tentando ser publicada na comunidade
 * - Qualquer invariante de negócio (seção 8 de business-rules.md) violada
 *
 * NÃO use para:
 * - Campos faltando ou tipo errado → BadRequestException (ValidationPipe)
 * - Recurso não encontrado → ResourceNotFoundException
 * - Conflito de unicidade → ResourceConflictException
 */
export class BusinessException extends DomainException {
  constructor(message: string, errorCode: string, details?: unknown) {
    super(message, HttpStatus.UNPROCESSABLE_ENTITY, errorCode, details);
  }
}
```

**Uso no UseCase:**

```typescript
// src/modules/meals/use-cases/create-meal/create-meal.use-case.ts
import { BusinessException } from '@common/exceptions';

@Injectable()
export class CreateMealUseCase {
  async execute(input: CreateMealInput): Promise<MealEntity> {
    // ← Regra de negócio: dieta travada por nutricionista (invariante I-05)
    if (input.isDietLocked) {
      throw new BusinessException(
        'Configurações de dieta bloqueadas pelo nutricionista vinculado.',
        'DIET_LOCKED',
      );
    }

    // ← Regra de negócio: blocos devem ser positivos (invariante I-04)
    if (input.blocks <= 0) {
      throw new BusinessException(
        'A quantidade de blocos deve ser maior que zero.',
        'INVALID_BLOCK_COUNT',
      );
    }

    return this.mealsRepository.create(input);
  }
}
```

### 2.4 `ResourceNotFoundException` — Recurso Não Encontrado (404)

```typescript
// src/common/exceptions/resource-not-found.exception.ts
import { HttpStatus } from '@nestjs/common';
import { DomainException } from './domain.exception';

/**
 * Exceção para recursos não encontrados.
 * HTTP 404 Not Found.
 *
 * Use quando:
 * - Refeição com ID inexistente
 * - Usuário não encontrado
 * - Alimento não encontrado no banco
 *
 * Prefira esta classe ao `NotFoundException` do NestJS nos UseCases
 * para manter o código semântico `RESOURCE_NOT_FOUND` consistente.
 */
export class ResourceNotFoundException extends DomainException {
  constructor(resourceType: string, identifier?: string | number) {
    const message = identifier
      ? `${resourceType} com identificador '${identifier}' não encontrado.`
      : `${resourceType} não encontrado.`;

    super(message, HttpStatus.NOT_FOUND, 'RESOURCE_NOT_FOUND');
  }
}
```

**Uso no UseCase:**

```typescript
import { ResourceNotFoundException } from '@common/exceptions';

async execute({ id, userId }: GetMealInput): Promise<MealEntity> {
  const meal = await this.mealsRepository.findById(id);

  if (!meal) {
    throw new ResourceNotFoundException('Refeição', id);
  }

  // ← Garante que a refeição pertence ao usuário autenticado
  if (meal.userId !== userId) {
    throw new ResourceForbiddenException('Refeição');
  }

  return meal;
}
```

### 2.5 `ResourceConflictException` — Conflito de Dados (409)

```typescript
// src/common/exceptions/resource-conflict.exception.ts
import { HttpStatus } from '@nestjs/common';
import { DomainException } from './domain.exception';

/**
 * Exceção para conflitos de dados.
 * HTTP 409 Conflict.
 *
 * Use quando:
 * - E-mail já cadastrado
 * - Alimento customizado com nome duplicado
 * - Qualquer violação de unicidade que pode ser comunicada ao usuário
 *
 * Na maioria dos casos, esta exceção é lançada no UseCase ANTES de ir ao banco,
 * ou no Repository ao capturar um erro de constraint e relançar como semântico.
 */
export class ResourceConflictException extends DomainException {
  constructor(resourceType: string, field: string, value?: string) {
    const message = value
      ? `Já existe um(a) ${resourceType} com ${field} '${value}'.`
      : `Já existe um(a) ${resourceType} com este ${field}.`;

    super(message, HttpStatus.CONFLICT, 'RESOURCE_CONFLICT');
  }
}
```

### 2.6 `ResourceForbiddenException` — Sem Permissão (403)

```typescript
// src/common/exceptions/resource-forbidden.exception.ts
import { HttpStatus } from '@nestjs/common';
import { DomainException } from './domain.exception';

/**
 * Exceção para acesso proibido a um recurso.
 * HTTP 403 Forbidden.
 *
 * Diferente do ForbiddenException do NestJS (que geralmente é lançado por Guards),
 * esta exceção é lançada pelo UseCase quando a autenticação é válida,
 * mas o usuário não tem permissão de negócio para aquele recurso específico
 * (ex: tentar acessar a refeição de outro usuário).
 */
export class ResourceForbiddenException extends DomainException {
  constructor(resourceType: string) {
    super(
      `Você não tem permissão para acessar este(a) ${resourceType}.`,
      HttpStatus.FORBIDDEN,
      'ACCESS_FORBIDDEN',
    );
  }
}
```

### 2.7 Barrel de Exceções — `src/common/exceptions/index.ts`

```typescript
// src/common/exceptions/index.ts
export { DomainException } from './domain.exception';
export { BusinessException } from './business.exception';
export { ResourceNotFoundException } from './resource-not-found.exception';
export { ResourceConflictException } from './resource-conflict.exception';
export { ResourceForbiddenException } from './resource-forbidden.exception';
```

---

## 3. Catálogo de Códigos de Erro (`errorCode`)

O `errorCode` é o identificador semântico retornado no campo `error.code` da resposta. O cliente mobile pode usar este código para exibir mensagens específicas ao usuário.

### Convenção de nomenclatura

```
SCREAMING_SNAKE_CASE
Prefixo do domínio + descrição do erro

Exemplos:
  DIET_LOCKED             → dieta travada por nutricionista
  INVALID_BLOCK_COUNT     → quantidade de blocos inválida
  RESOURCE_NOT_FOUND      → recurso genérico não encontrado
  RESOURCE_CONFLICT       → conflito de dados
  ACCESS_FORBIDDEN        → sem permissão de negócio
  MEAL_INCOMPLETE         → refeição incompleta
  DAILY_LIMIT_EXCEEDED    → limite diário atingido
```

### Tabela de códigos — CAB 2.0

| `errorCode` | Status HTTP | Contexto |
|---|---|---|
| `RESOURCE_NOT_FOUND` | 404 | Recurso qualquer não encontrado |
| `RESOURCE_CONFLICT` | 409 | Unicidade violada (e-mail, nome) |
| `ACCESS_FORBIDDEN` | 403 | Usuário sem permissão de negócio |
| `DIET_LOCKED` | 422 | Dieta travada por nutricionista (I-05) |
| `INVALID_BLOCK_COUNT` | 422 | Blocos <= 0 (I-04) |
| `MEAL_INCOMPLETE` | 422 | Refeição incompleta tentando publicar na comunidade (I-09) |
| `DAILY_LIMIT_EXCEEDED` | 422 | Consumo acima do total diário |
| `FOOD_MISSING_CATEGORY` | 422 | Alimento sem categoria predominante (I-03) |
| `NUTRITIONIST_LINK_REQUIRED` | 422 | Operação requer nutricionista vinculado |
| `VALIDATION_ERROR` | 400 | Payload inválido (gerado pelo ValidationPipe) |
| `UNAUTHORIZED` | 401 | Token ausente ou inválido |
| `INTERNAL_SERVER_ERROR` | 500 | Erro inesperado |

> **Regra:** Ao criar um novo `errorCode`, adicione-o nesta tabela antes de usá-lo no código. Codes não documentados aqui são proibidos.

---

## 4. `HttpExceptionFilter` Expandido

O `HttpExceptionFilter` já definido em `rest-api-patterns.md` captura `HttpException`. Como toda `DomainException` estende `HttpException`, ele as captura automaticamente e retorna o `errorCode` correto.

O filter **não precisa ser alterado** — ele já funciona com a hierarquia definida nesta skill. Mas aqui está como ele deve se comportar com as DomainExceptions:

```typescript
// src/common/filters/http-exception.filter.ts
// (definido em rest-api-patterns.md — esta seção apenas documenta o comportamento)

// Quando o Filter recebe uma DomainException:
// exception.getResponse() → { message, errorCode, details }
// exception.getStatus()   → o status HTTP correto (422, 404, 409, etc.)

// O filter extrai:
// - code: exception.getResponse().errorCode ?? statusToCode(status)
// - message: exception.getResponse().message
// - details: exception.getResponse().details (opcional)
```

### Como o filter deve extrair o `errorCode` de uma `DomainException`

Para garantir que `errorCode` apareça corretamente no campo `error.code`, o filter existente em `rest-api-patterns.md` precisa desta lógica adicional ao extrair o `code`:

```typescript
// Trecho adicional para o HttpExceptionFilter de rest-api-patterns.md
// Dentro do método catch(), na extração do code:

const exceptionResponse = exception.getResponse();

// Se a exceção carrega um errorCode semântico (DomainException),
// use-o. Caso contrário, usa o mapeamento padrão de status → code.
const code =
  typeof exceptionResponse === 'object' && (exceptionResponse as any).errorCode
    ? (exceptionResponse as any).errorCode
    : HttpExceptionFilter.statusToCode(status);

// O details também deve ser propagado (validação por campo, etc.)
const details =
  typeof exceptionResponse === 'object'
    ? (exceptionResponse as any).details ?? (exceptionResponse as any).message
    : undefined;
```

**Exemplo de resposta para `ResourceNotFoundException`:**

```json
// GET /api/v1/meals/meal-inexistente → 404
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Refeição com identificador 'meal-inexistente' não encontrado.",
    "path": "/api/v1/meals/meal-inexistente",
    "timestamp": "2025-03-15T10:00:00.000Z"
  }
}
```

**Exemplo de resposta para `BusinessException`:**

```json
// PATCH /api/v1/users/me/blocks → 422
{
  "error": {
    "code": "DIET_LOCKED",
    "message": "Configurações de dieta bloqueadas pelo nutricionista vinculado.",
    "path": "/api/v1/users/me/blocks",
    "timestamp": "2025-03-15T10:00:00.000Z"
  }
}
```

---

## 5. Mapeamento de Erros de Banco para HTTP

Erros de banco (TypeORM e Prisma) são detalhes de infraestrutura e **nunca devem vazar para o cliente**. O Repository é responsável por capturá-los e relançá-los como exceções semânticas.

### 5.1 Estratégia: Repository captura, UseCase não sabe

```
Repository.create()
    ↓ QueryFailedError (TypeORM) / PrismaClientKnownRequestError (Prisma)
    ↓ catch interno no Repository
    ↓ relança como ResourceConflictException / ResourceNotFoundException
    ↓
UseCase recebe a exceção semântica
    ↓ (pode ou não capturar — geralmente não precisa)
    ↓
ExceptionFilter captura e formata
```

> **Regra:** UseCases nunca fazem `catch` de erros de banco. Só os Repositories fazem.

### 5.2 Mapeamento TypeORM

```typescript
// src/common/filters/typeorm-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  Logger,
  HttpStatus,
} from '@nestjs/common';
import { Response, Request } from 'express';
import {
  QueryFailedError,
  EntityNotFoundError,
} from 'typeorm';

/**
 * Captura erros de infraestrutura TypeORM que escaparam do Repository
 * sem serem mapeados para exceções semânticas.
 *
 * Responsabilidade: DEFESA — o Repository deve capturar primeiro.
 * Este filter é a última linha de defesa para não vazar 500s desnecessários.
 *
 * NÃO registre como filter global no main.ts. Registre no DatabaseModule
 * apenas se necessário para garantir que erros de banco não sejam silenciados.
 *
 * Prefira a abordagem de mapear no próprio Repository (seção 5.3).
 */
@Catch(QueryFailedError, EntityNotFoundError)
export class TypeOrmExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(TypeOrmExceptionFilter.name);

  catch(exception: QueryFailedError | EntityNotFoundError, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    this.logger.error(
      `[TypeORM] Erro de banco não tratado no Repository: ${exception.message}`,
      exception.stack,
    );

    // EntityNotFoundError → 404 (lançado por findOneOrFail)
    if (exception instanceof EntityNotFoundError) {
      response.status(HttpStatus.NOT_FOUND).json({
        error: {
          code: 'RESOURCE_NOT_FOUND',
          message: 'Recurso não encontrado.',
          path: request.url,
          timestamp: new Date().toISOString(),
        },
      });
      return;
    }

    // QueryFailedError → verificar código de constraint do driver
    if (exception instanceof QueryFailedError) {
      const driverError = (exception as any).driverError ?? {};
      const pgCode = driverError.code; // PostgreSQL error codes

      // 23505: unique_violation (e-mail duplicado, constraint única)
      if (pgCode === '23505') {
        response.status(HttpStatus.CONFLICT).json({
          error: {
            code: 'RESOURCE_CONFLICT',
            message: 'Já existe um registro com estes dados.',
            path: request.url,
            timestamp: new Date().toISOString(),
          },
        });
        return;
      }

      // 23503: foreign_key_violation (referência a ID inexistente)
      if (pgCode === '23503') {
        response.status(HttpStatus.BAD_REQUEST).json({
          error: {
            code: 'INVALID_REFERENCE',
            message: 'Referência a recurso inexistente.',
            path: request.url,
            timestamp: new Date().toISOString(),
          },
        });
        return;
      }
    }

    // Fallback: erro desconhecido de banco → 500
    response.status(HttpStatus.INTERNAL_SERVER_ERROR).json({
      error: {
        code: 'INTERNAL_SERVER_ERROR',
        message: 'Erro interno do servidor.',
        path: request.url,
        timestamp: new Date().toISOString(),
      },
    });
  }
}
```

### 5.3 Padrão recomendado: mapeamento no Repository (TypeORM)

A abordagem preferida é mapear dentro do próprio Repository, sem depender do `TypeOrmExceptionFilter`:

```typescript
// src/modules/users/repositories/users.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, QueryFailedError } from 'typeorm';
import { UserEntity } from '../entities/user.entity';
import {
  ResourceConflictException,
  ResourceNotFoundException,
} from '@common/exceptions';

@Injectable()
export class UsersRepository {
  constructor(
    @InjectRepository(UserEntity)
    private readonly repo: Repository<UserEntity>,
  ) {}

  async create(data: Partial<UserEntity>): Promise<UserEntity> {
    const entity = this.repo.create(data);
    try {
      return await this.repo.save(entity);
    } catch (error) {
      // ✅ Mapeamento dentro do Repository — UseCase não sabe do banco
      if (error instanceof QueryFailedError) {
        const pgCode = (error as any).driverError?.code;
        if (pgCode === '23505') {
          throw new ResourceConflictException('Usuário', 'e-mail', data.email);
        }
      }
      throw error; // Relança erros desconhecidos para o filter capturar
    }
  }

  async findByIdOrFail(id: string): Promise<UserEntity> {
    try {
      return await this.repo.findOneOrFail({ where: { id } });
    } catch {
      // EntityNotFoundError → semântica de negócio
      throw new ResourceNotFoundException('Usuário', id);
    }
  }
}
```

### 5.4 Mapeamento de Erros Prisma

```typescript
// src/common/utils/prisma-error-mapper.ts
import { Prisma } from '@prisma/client';
import {
  ResourceConflictException,
  ResourceNotFoundException,
} from '@common/exceptions';

/**
 * Utilitário para mapear PrismaClientKnownRequestError para DomainException.
 * Use dentro dos Repositories Prisma ao capturar erros.
 *
 * Referência de códigos: https://www.prisma.io/docs/reference/api-reference/error-reference
 */
export function mapPrismaError(
  error: unknown,
  resourceType: string,
  field?: string,
): never {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    switch (error.code) {
      // P2002: Unique constraint violation
      case 'P2002': {
        const fields = (error.meta?.target as string[])?.join(', ') ?? field ?? 'campo';
        throw new ResourceConflictException(resourceType, fields);
      }
      // P2025: Record not found (update/delete em registro inexistente)
      case 'P2025':
        throw new ResourceNotFoundException(resourceType);
      // P2003: Foreign key constraint violation
      case 'P2003':
        throw new ResourceNotFoundException('Referência', String(error.meta?.field_name));
      default:
        throw error; // Relança para o filter capturar como 500
    }
  }
  throw error;
}
```

**Uso no Repository Prisma:**

```typescript
// src/modules/users/repositories/users.repository.ts (Prisma)
import { mapPrismaError } from '@common/utils/prisma-error-mapper';

@Injectable()
export class UsersRepository {
  async create(data: CreateUserData): Promise<User> {
    try {
      return await this.prisma.user.create({ data });
    } catch (error) {
      mapPrismaError(error, 'Usuário', 'e-mail'); // ✅ Semântico, sem vazar detalhes
    }
  }
}
```

### 5.5 Tabela de mapeamento: erros de banco → HTTP

| Banco | Código | Significado | HTTP | `errorCode` |
|-------|--------|-------------|------|-------------|
| PostgreSQL | `23505` | `unique_violation` | 409 | `RESOURCE_CONFLICT` |
| PostgreSQL | `23503` | `foreign_key_violation` | 400 | `INVALID_REFERENCE` |
| PostgreSQL | `23502` | `not_null_violation` | 400 | `VALIDATION_ERROR` |
| TypeORM | `EntityNotFoundError` | `findOneOrFail` falhou | 404 | `RESOURCE_NOT_FOUND` |
| Prisma | `P2002` | Unique constraint | 409 | `RESOURCE_CONFLICT` |
| Prisma | `P2025` | Record not found | 404 | `RESOURCE_NOT_FOUND` |
| Prisma | `P2003` | FK constraint failed | 400 | `INVALID_REFERENCE` |
| Qualquer | Desconhecido | Erro interno | 500 | `INTERNAL_SERVER_ERROR` |

---

## 6. Registro Global dos Filtros no `main.ts`

A ordem de registro dos filtros no `main.ts` importa. Filtros são aplicados de baixo para cima (o último registrado é o primeiro a executar).

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { VersioningType, ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { TransformInterceptor } from '@common/interceptors/transform.interceptor';
import { HttpExceptionFilter } from '@common/filters/http-exception.filter';
import { TypeOrmExceptionFilter } from '@common/filters/typeorm-exception.filter'; // ← opcional
import { env } from '@config/env.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.setGlobalPrefix('api');
  app.enableVersioning({ type: VersioningType.URI });

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  app.useGlobalInterceptors(new TransformInterceptor());

  // ← Ordem de registro (aplicados de baixo para cima):
  // 1. TypeOrmExceptionFilter captura erros de banco não mapeados (mais específico)
  // 2. HttpExceptionFilter captura tudo que é HttpException (incluindo DomainException)
  //
  // Se você mapear TODOS os erros nos Repositories (recomendado),
  // o TypeOrmExceptionFilter é desnecessário.
  app.useGlobalFilters(
    new HttpExceptionFilter(),       // ← captura HttpException e DomainException
    // new TypeOrmExceptionFilter(), // ← opcional, apenas como defesa
  );

  await app.listen(env.PORT);
}
bootstrap();
```

> **Nota sobre a ordem:** No NestJS, `useGlobalFilters` registra filtros em ordem decrescente de prioridade (o último vem primeiro). Se usar dois filtros, coloque o mais específico por último.

---

## 7. Integração com a Clean Architecture

### Fluxo completo de uma exceção de negócio

```
HTTP Request → Controller → UseCase.execute()
                                    ↓
                          throw new BusinessException(
                            'Limite diário atingido.',
                            'DAILY_LIMIT_EXCEEDED'
                          )
                                    ↓ (sobe pela call stack)
                          HttpExceptionFilter.catch()
                                    ↓
                          response.status(422).json({
                            error: {
                              code: 'DAILY_LIMIT_EXCEEDED',
                              message: 'Limite diário atingido.',
                              path: '/api/v1/meals',
                              timestamp: '...'
                            }
                          })
```

### O que cada camada faz com exceções

```typescript
// ✅ CORRETO — UseCase lança exceção semântica
@Injectable()
export class UpdateDailyBlocksUseCase {
  async execute(input: UpdateDailyBlocksInput): Promise<void> {
    const user = await this.usersRepository.findByIdOrFail(input.userId);

    // Invariante I-05: dieta travada impede edição
    if (user.isDietLocked) {
      throw new BusinessException(
        'Não é possível alterar os blocos diários com dieta travada.',
        'DIET_LOCKED',
      );
    }

    await this.usersRepository.updateDailyBlocks(input.userId, input.totalDailyBlocks);
  }
}

// ✅ CORRETO — Controller delega, não trata
@Patch('me/blocks')
async updateBlocks(
  @Body() dto: UpdateDailyBlocksDto,
  @CurrentUser() user: UserPayload,
): Promise<void> {
  await this.updateDailyBlocksUseCase.execute({
    userId: user.id,
    totalDailyBlocks: dto.totalDailyBlocks,
  });
  // Se o UseCase lançar, o Filter captura — Controller não faz try/catch aqui
}

// ❌ ERRADO — Controller fazendo catch e relançando (desnecessário)
@Patch('me/blocks')
async updateBlocks(@Body() dto: UpdateDailyBlocksDto) {
  try {
    await this.updateDailyBlocksUseCase.execute(dto);
  } catch (error) {
    throw new HttpException('Erro', 422); // ← perde o errorCode semântico
  }
}
```

---

## 8. Exceções vs. Retorno Nulo — Quando Usar Cada Um

| Situação | Use | Justificativa |
|---|---|---|
| Recurso não existe e a operação **requer** que exista | `ResourceNotFoundException` | Semântico — o cliente deve saber que não encontrou |
| Busca opcional que pode não retornar resultado | Retorna `null` | É um estado válido, não um erro |
| Tentativa de criar algo que já existe | `ResourceConflictException` | Informa o conflito |
| Usuário sem permissão para acessar | `ResourceForbiddenException` | Diferencia de "não existe" |
| Regra de negócio violada | `BusinessException` | Contexto semântico + errorCode |

```typescript
// ✅ findById: retorna null se não encontrar (busca opcional)
async findById(id: string): Promise<MealEntity | null> {
  return this.repo.findOne({ where: { id } });
}

// ✅ findByIdOrFail: lança se não encontrar (busca obrigatória)
async findByIdOrFail(id: string): Promise<MealEntity> {
  const meal = await this.repo.findOne({ where: { id } });
  if (!meal) throw new ResourceNotFoundException('Refeição', id);
  return meal;
}
```

---

## 9. Anti-Padrões Comuns

### ❌ Lançar exceção genérica do Node sem semântica

```typescript
// ❌ ERRADO — perde o errorCode, vira 500 desnecessário
throw new Error('Dieta bloqueada');

// ✅ CORRETO
throw new BusinessException('Dieta bloqueada pelo nutricionista.', 'DIET_LOCKED');
```

### ❌ Catch no Controller que engole o errorCode

```typescript
// ❌ ERRADO — engole o BusinessException e devolve resposta genérica
@Post()
async create(@Body() dto: CreateMealDto) {
  try {
    return await this.createMealUseCase.execute(dto);
  } catch (error) {
    throw new BadRequestException('Erro ao criar refeição'); // ← perde DIET_LOCKED
  }
}

// ✅ CORRETO — deixa o Filter cuidar
@Post()
async create(@Body() dto: CreateMealDto) {
  return this.createMealUseCase.execute(dto); // sem try/catch
}
```

### ❌ Criar errorCode não documentado

```typescript
// ❌ ERRADO — código não está no catálogo (seção 3)
throw new BusinessException('Problema', 'MINHA_COISA_NOVA');

// ✅ CORRETO — adiciona ao catálogo ANTES de usar no código
// 1. Adiciona 'MINHA_COISA_NOVA' na tabela da seção 3
// 2. Usa no código
```

### ❌ Vazar detalhes de banco para o cliente

```typescript
// ❌ ERRADO — message do erro de banco vira resposta HTTP
catch (error) {
  throw new InternalServerErrorException(error.message); // ex: "duplicate key value violates unique constraint users_email_key"
}

// ✅ CORRETO — mensagem semântica sem detalhe de banco
catch (error) {
  if (error instanceof QueryFailedError && (error as any).driverError?.code === '23505') {
    throw new ResourceConflictException('Usuário', 'e-mail');
  }
  throw error; // desconhecido: deixa virar 500 limpo
}
```

### ❌ Usar `DomainException` diretamente

```typescript
// ❌ ERRADO — DomainException é abstrata por design
throw new DomainException('Erro', 422, 'ALGUM_CODIGO');

// ✅ CORRETO — use uma subclasse existente ou crie uma nova
throw new BusinessException('Erro de negócio.', 'ALGUM_CODIGO');
```

---

## 10. Organização de Arquivos Final

```
src/
├── common/
│   ├── exceptions/
│   │   ├── domain.exception.ts
│   │   ├── business.exception.ts
│   │   ├── resource-not-found.exception.ts
│   │   ├── resource-conflict.exception.ts
│   │   ├── resource-forbidden.exception.ts
│   │   └── index.ts
│   │
│   ├── filters/
│   │   ├── http-exception.filter.ts     ← definido em rest-api-patterns.md (expandido aqui)
│   │   └── typeorm-exception.filter.ts  ← opcional, defesa contra erros não mapeados
│   │
│   └── utils/
│       └── prisma-error-mapper.ts       ← utilitário para Repositories Prisma
```

---

## 11. Checklist de Validação

### Exceções e hierarquia
- [ ] Toda exceção de negócio estende `DomainException` via uma subclasse?
- [ ] `DomainException` nunca é instanciada diretamente?
- [ ] Todo `errorCode` usado está documentado na tabela da seção 3?
- [ ] Exceções novas foram adicionadas ao barrel `src/common/exceptions/index.ts`?

### Camadas e responsabilidades
- [ ] UseCases lançam `BusinessException`, `ResourceNotFoundException`, `ResourceConflictException` ou `ResourceForbiddenException`?
- [ ] Controllers **não** têm `try/catch` que engolam ou transformam exceções de negócio?
- [ ] Repositories mapeiam erros de banco para exceções semânticas antes de repassar?
- [ ] Erros de banco **nunca** chegam ao cliente com mensagens internas do driver?

### Filter global
- [ ] `HttpExceptionFilter` extrai o `errorCode` de `DomainException` corretamente (`exception.getResponse().errorCode`)?
- [ ] O campo `details` é propagado quando presente?
- [ ] Exceções `>= 500` são logadas com `logger.error` (stack trace completo)?
- [ ] Exceções `< 500` são logadas com `logger.warn` (sem stack trace)?

### Prisma / TypeORM
- [ ] `mapPrismaError` (Prisma) ou `catch(QueryFailedError)` (TypeORM) usado nos Repositories?
- [ ] `P2002`/`23505` → `ResourceConflictException` (não `ConflictException` do NestJS)?
- [ ] `P2025`/`EntityNotFoundError` → `ResourceNotFoundException`?

---

## Referências

- [rest-api-patterns.md](rest-api-patterns.md) — `HttpExceptionFilter` global, envelope `{ error }`, status codes por situação
- [clean-architecture.md](clean-architecture.md) — Controller fino, UseCase com lógica de negócio, onde cada tipo de erro nasce
- [nest-project-structure.md](nest-project-structure.md) — `src/common/exceptions/`, `src/common/filters/`, path aliases `@common/`
- [dto-validation.md](dto-validation.md) — `ValidationPipe` global, `BadRequestException` para erros de formato
- [typeorm-patterns.md](typeorm-patterns.md) — `QueryFailedError`, `EntityNotFoundError`, padrão de Repository
- [prisma-patterns.md](prisma-patterns.md) — `PrismaClientKnownRequestError`, tratamento no Repository
- [business-rules.md](business-rules.md) — Invariantes de negócio (I-03 a I-11) que devem gerar `BusinessException`
- [NestJS Docs — Exception Filters](https://docs.nestjs.com/exception-filters)
- [Prisma Error Reference](https://www.prisma.io/docs/reference/api-reference/error-reference)