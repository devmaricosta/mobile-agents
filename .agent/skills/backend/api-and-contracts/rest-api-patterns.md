---
name: rest-api-patterns
description: Padrões REST no NestJS — versionamento de API (/v1/), convenções de nomenclatura de rotas e controllers, uso correto de status HTTP, padronização do envelope de resposta ({ data, meta, error }) via TransformInterceptor e ExceptionFilter, e paginação com cursor vs offset. Complementa clean-architecture.md (Controllers finos, ViewModels como shape interno) e nest-project-structure.md (onde ficam interceptors e filters em src/common/).
---

# Padrões REST — NestJS

Você é um Arquiteto de Software Senior. Esta skill define os **contratos de API** que todo Controller deve seguir. Ela complementa `clean-architecture.md` (Controllers finos, ViewModel como shape interno antes do envelope) e `nest-project-structure.md` (onde ficam os arquivos `src/common/interceptors/` e `src/common/filters/`).

> **Regra fundamental:** O Controller é responsável por receber a requisição e delegar ao UseCase. O envelope `{ data, meta, error }` é aplicado **globalmente** por um interceptor e um filter — o Controller em si retorna apenas o ViewModel ou lança uma exceção.

---

## 1. Versionamento de API — URI Versioning (`/v1/`)

### Estratégia adotada: URI Versioning

O projeto usa **URI versioning** como estratégia de versionamento. É a estratégia mais explícita, transparente e amplamente adotada pelo mercado. A versão fica vísivel na URL, sem necessidade de headers customizados.

```
GET /v1/meals
GET /v1/users/me
POST /v1/auth/login
```

### Configuração global no `main.ts`

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { VersioningType, ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { env } from '@config/env.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // ← Prefixo global de API
  app.setGlobalPrefix('api');

  // ← Versionamento via URI: /api/v1/...
  app.enableVersioning({
    type: VersioningType.URI,
  });

  // ← Validação automática dos DTOs
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,         // remove campos não declarados no DTO
      forbidNonWhitelisted: true,
      transform: true,         // transforma query params e path params para os tipos do DTO
    }),
  );

  await app.listen(env.PORT);
}
bootstrap();
```

### Declaração de versão no Controller

```typescript
// src/modules/meals/controllers/meals.controller.ts
import { Controller, Get, Post, Body, Version } from '@nestjs/common';

// ✅ PADRÃO: versão declarada no nível do controller
@Controller({ path: 'meals', version: '1' })
export class MealsController {
  // Todos os endpoints neste controller respondem em /api/v1/meals
}

// ✅ Alternativa: versão no nível do endpoint (quando o controller tem versões mistas)
@Controller('meals')
export class MealsController {
  @Version('1')
  @Get()
  findAll() { /* /api/v1/meals */ }

  @Version('2')
  @Get()
  findAllV2() { /* /api/v2/meals */ }
}
```

### Regras de versionamento

| Situação | Como fazer |
|----------|-----------|
| **Nova API (do zero)** | Sempre comece em `v1` |
| **Mudança não-breaking** (novo campo opcional na resposta) | Não requer nova versão |
| **Mudança breaking** (remover campo, mudar tipo, renomear rota) | Criar `v2` e manter `v1` por período de deprecação |
| **Rotas de saúde/infra** (`/health`, `/metrics`) | Use `VERSION_NEUTRAL` — sem versão |
| **Quando deprecar uma versão** | Adicionar header `Deprecation: true` e `Sunset: <data>` na resposta |

```typescript
// Rota sem versão (health check, readiness probe)
import { VERSION_NEUTRAL } from '@nestjs/common';

@Controller({ path: 'health', version: VERSION_NEUTRAL })
export class HealthController {
  @Get()
  check() { return { status: 'ok' }; }
}
```

---

## 2. Nomenclatura de Rotas e Controllers

### Rotas: kebab-case, plural, recursos aninhados

```
# ✅ CORRETO
GET    /api/v1/meals              ← plural, recurso raiz
POST   /api/v1/meals
GET    /api/v1/meals/:id
PATCH  /api/v1/meals/:id
DELETE /api/v1/meals/:id
GET    /api/v1/meals/:id/logs     ← sub-recurso aninhado
POST   /api/v1/auth/login         ← ação como sub-recurso (verbo no path só em casos especiais)
POST   /api/v1/auth/refresh-token ← kebab-case para palavras compostas

# ❌ ERRADO
GET /api/v1/getMeal               ← verbo no path (não REST)
GET /api/v1/Meal                  ← PascalCase
GET /api/v1/meal_history          ← underscore
GET /api/v1/meal                  ← singular
```

### Controller: um por domínio, nome no plural

```typescript
// ✅ CORRETO
@Controller({ path: 'meals', version: '1' })     // → /api/v1/meals
export class MealsController {}

@Controller({ path: 'meal-logs', version: '1' }) // → /api/v1/meal-logs
export class MealLogsController {}

// ❌ ERRADO
@Controller('getMeal')   // verbo na rota
@Controller('Meal')      // singular, PascalCase
```

### Tabela de métodos HTTP e ações CRUD

| Ação | Método HTTP | Rota | Status de Sucesso |
|------|-------------|------|------------------|
| Listar todos | `GET` | `/meals` | `200 OK` |
| Buscar um | `GET` | `/meals/:id` | `200 OK` |
| Criar | `POST` | `/meals` | `201 Created` |
| Atualizar parcialmente | `PATCH` | `/meals/:id` | `200 OK` |
| Sobrescrever | `PUT` | `/meals/:id` | `200 OK` |
| Deletar | `DELETE` | `/meals/:id` | `204 No Content` |
| Ação específica | `POST` | `/meals/:id/share` | `200 OK` ou `204` |

### Convenção de nomenclatura dos arquivos de Controller

Consistente com `nest-project-structure.md`:

| Item | Convenção | Exemplo |
|------|-----------|---------|
| Arquivo | kebab-case + `.controller.ts` | `meals.controller.ts` |
| Classe | PascalCase + `Controller` | `MealsController` |
| Pasta | `controllers/` dentro do módulo | `src/modules/meals/controllers/` |

---

## 3. Uso Correto de Status HTTP

### Status de sucesso

```typescript
import {
  Controller, Get, Post, Patch, Delete, Param, Body,
  HttpCode, HttpStatus,
} from '@nestjs/common';

@Controller({ path: 'meals', version: '1' })
export class MealsController {

  @Get()                                    // ← 200 é o default do GET
  async findAll() { ... }

  @Get(':id')
  async findOne(@Param('id') id: string) { ... }

  @Post()
  @HttpCode(HttpStatus.CREATED)             // ← 201 para criação
  async create(@Body() dto: CreateMealDto) { ... }

  @Patch(':id')                             // ← 200 para update parcial
  async update(@Param('id') id: string, @Body() dto: UpdateMealDto) { ... }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)          // ← 204 para deleção sem body
  async remove(@Param('id') id: string) { ... }
}
```

### Status de erro — Use as exceções do NestJS

**Nunca** construa manualmente objetos de erro no Controller. Use as exceções HTTP do `@nestjs/common`:

```typescript
import {
  NotFoundException,
  BadRequestException,
  UnauthorizedException,
  ForbiddenException,
  ConflictException,
  UnprocessableEntityException,
  InternalServerErrorException,
} from '@nestjs/common';

// Em qualquer camada (UseCase, Guard, Pipe):
throw new NotFoundException('Refeição não encontrada');
throw new BadRequestException('Parâmetros inválidos');
throw new UnauthorizedException('Token inválido ou expirado');
throw new ForbiddenException('Sem permissão para esta ação');
throw new ConflictException('E-mail já cadastrado');
```

### Mapeamento de status de erro por situação

| Situação | Exceção NestJS | Status HTTP |
|----------|---------------|-------------|
| Recurso não encontrado | `NotFoundException` | `404 Not Found` |
| Payload inválido (validação DTO) | `BadRequestException` | `400 Bad Request` |
| Token ausente / inválido | `UnauthorizedException` | `401 Unauthorized` |
| Usuário sem permissão | `ForbiddenException` | `403 Forbidden` |
| Conflito (email duplicado) | `ConflictException` | `409 Conflict` |
| Entidade não processável (regra de negócio) | `UnprocessableEntityException` | `422 Unprocessable Entity` |
| Erro inesperado de servidor | `InternalServerErrorException` | `500 Internal Server Error` |

> **Regra sobre 422 vs 400:** Use `BadRequestException` (400) para erros de **formato** (campo faltando, tipo errado — capturado pelo `ValidationPipe`). Use `UnprocessableEntityException` (422) para erros de **regra de negócio** que passam na validação estrutural mas falham semanticamente (ex: "data de nascimento no futuro", "limite de refeições diárias atingido").

---

## 4. Envelope de Resposta — `{ data, meta, error }`

### Estratégia: TransformInterceptor global + HttpExceptionFilter global

O envelope é padronizado em **dois pontos globais** da aplicação, sem nenhuma mudança nos Controllers ou Use Cases:

1. **`TransformInterceptor`** → envolve respostas de sucesso em `{ data, meta }`
2. **`HttpExceptionFilter`** → formata erros em `{ error }` consistente

```
Controller retorna: MealViewModel (ou lança exception)
         ↓
TransformInterceptor: { data: MealViewModel, meta: {...} }    ← sucesso
HttpExceptionFilter: { error: { code, message, details } }    ← erro
```

### 4.1 Shape da Resposta de Sucesso

```typescript
// Resposta de item único:
{
  "data": {
    "id": "meal-1",
    "blocks": 4,
    "mode": "quick",
    "createdAt": "2024-01-15T10:00:00.000Z"
  }
}

// Resposta de lista com paginação:
{
  "data": [
    { "id": "meal-1", "blocks": 4, "mode": "quick", ... },
    { "id": "meal-2", "blocks": 3, "mode": "custom", ... }
  ],
  "meta": {
    "total": 42,          // total de registros (apenas em offset pagination)
    "page": 1,            // página atual (apenas em offset pagination)
    "lastPage": 5,        // última página (apenas em offset pagination)
    "limit": 10,          // itens por página
    "nextCursor": "abc",  // cursor para próxima página (apenas em cursor pagination)
    "prevCursor": "xyz"   // cursor para página anterior (apenas em cursor pagination)
  }
}
```

### 4.2 `TransformInterceptor` — Resposta de Sucesso

```typescript
// src/common/interceptors/transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface ApiResponse<T> {
  data: T;
  meta?: Record<string, unknown>;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, ApiResponse<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<ApiResponse<T>> {
    return next.handle().pipe(
      map((payload) => {
        // Se o Controller já retornou { data, meta } (caso de lista paginada),
        // preserva a estrutura. Caso contrário, envolve em { data }.
        if (payload && typeof payload === 'object' && 'data' in payload) {
          return payload as ApiResponse<T>;
        }
        return { data: payload };
      }),
    );
  }
}
```

### 4.3 `HttpExceptionFilter` — Resposta de Erro

```typescript
// src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

export interface ApiError {
  error: {
    code: string;       // ex: 'NOT_FOUND', 'BAD_REQUEST', 'UNAUTHORIZED'
    message: string;    // mensagem legível para o desenvolvedor / cliente
    details?: unknown;  // detalhes de validação (campos com erro), apenas em 400/422
    path: string;       // endpoint que originou o erro
    timestamp: string;  // ISO 8601
  };
}

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    // O NestJS ValidationPipe retorna { message: string[], error: string }
    const message =
      typeof exceptionResponse === 'string'
        ? exceptionResponse
        : (exceptionResponse as any).message;

    const details =
      typeof exceptionResponse === 'object' &&
      Array.isArray((exceptionResponse as any).message)
        ? (exceptionResponse as any).message   // array de erros de validação
        : undefined;

    const code = HttpExceptionFilter.statusToCode(status);

    if (status >= HttpStatus.INTERNAL_SERVER_ERROR) {
      this.logger.error(`[${code}] ${request.method} ${request.url}`, exception.stack);
    }

    const body: ApiError = {
      error: {
        code,
        message: Array.isArray(message) ? message[0] : message,
        details: details ?? undefined,
        path: request.url,
        timestamp: new Date().toISOString(),
      },
    };

    response.status(status).json(body);
  }

  private static statusToCode(status: number): string {
    const map: Record<number, string> = {
      400: 'BAD_REQUEST',
      401: 'UNAUTHORIZED',
      403: 'FORBIDDEN',
      404: 'NOT_FOUND',
      409: 'CONFLICT',
      422: 'UNPROCESSABLE_ENTITY',
      429: 'TOO_MANY_REQUESTS',
      500: 'INTERNAL_SERVER_ERROR',
    };
    return map[status] ?? `HTTP_${status}`;
  }
}
```

### 4.4 Registrando globalmente no `main.ts`

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { VersioningType, ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { TransformInterceptor } from '@common/interceptors/transform.interceptor';
import { HttpExceptionFilter } from '@common/filters/http-exception.filter';
import { env } from '@config/env.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.setGlobalPrefix('api');
  app.enableVersioning({ type: VersioningType.URI });

  // ← Validação automática dos DTOs (400 automático para campos inválidos)
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  // ← Envelope de resposta global
  app.useGlobalInterceptors(new TransformInterceptor());

  // ← Formato de erro global
  app.useGlobalFilters(new HttpExceptionFilter());

  await app.listen(env.PORT);
  console.log(`🚀 API rodando em http://localhost:${env.PORT}/api/v1`);
}
bootstrap();
```

### 4.5 Como o Controller interage com o envelope

O Controller **não sabe** que existe um envelope — ele apenas retorna o ViewModel. O interceptor aplica o envelope automaticamente:

```typescript
// src/modules/meals/controllers/meals.controller.ts

@Controller({ path: 'meals', version: '1' })
@UseGuards(JwtAuthGuard)
export class MealsController {
  constructor(
    private readonly createMealUseCase: CreateMealUseCase,
    private readonly getMealHistoryUseCase: GetMealHistoryUseCase,
  ) {}

  // ✅ Controller retorna MealViewModel — o interceptor envolve em { data: ... }
  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(
    @Body() dto: CreateMealDto,
    @CurrentUser() user: UserPayload,
  ): Promise<MealViewModel> {
    const meal = await this.createMealUseCase.execute({ ...dto, userId: user.id });
    return MealViewModel.fromEntity(meal);
    // Cliente recebe: { "data": { "id": "...", "blocks": 4, ... } }
  }

  // ✅ Para listas paginadas, o Controller retorna o objeto com { data, meta }
  // e o interceptor o detecta e não envolve novamente
  @Get()
  async findAll(
    @CurrentUser() user: UserPayload,
    @Query() query: PaginationDto,
  ): Promise<{ data: MealViewModel[]; meta: PaginationMeta }> {
    const result = await this.getMealHistoryUseCase.execute({
      userId: user.id,
      ...query,
    });
    return {
      data: MealViewModel.fromEntities(result.items),
      meta: result.meta,
    };
    // Cliente recebe: { "data": [...], "meta": { "nextCursor": "abc", ... } }
  }
}
```

### 4.6 Shape de erro — exemplos práticos

```json
// 400 — Validação de DTO (ValidationPipe)
{
  "error": {
    "code": "BAD_REQUEST",
    "message": "blocks must be a positive number",
    "details": [
      "blocks must be a positive number",
      "mode must be one of the following values: quick, custom"
    ],
    "path": "/api/v1/meals",
    "timestamp": "2024-01-15T10:00:00.000Z"
  }
}

// 404 — Recurso não encontrado
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Refeição não encontrada",
    "path": "/api/v1/meals/meal-999",
    "timestamp": "2024-01-15T10:00:00.000Z"
  }
}

// 422 — Regra de negócio violada
{
  "error": {
    "code": "UNPROCESSABLE_ENTITY",
    "message": "Limite diário de refeições atingido",
    "path": "/api/v1/meals",
    "timestamp": "2024-01-15T10:00:00.000Z"
  }
}
```

---

## 5. Paginação — Cursor vs Offset

### Tabela de decisão

| Critério | Use Cursor | Use Offset |
|----------|-----------|-----------|
| **Dataset grande** (> 10k registros) | ✅ | ❌ |
| **Dados dinâmicos** (inserts/deletes frequentes) | ✅ | ❌ |
| **Scroll infinito no mobile** | ✅ | ❌ |
| **Feed em tempo real** | ✅ | ❌ |
| **Admin/backoffice** (pular para página N) | ❌ | ✅ |
| **Dataset pequeno e estático** (< 1k registros) | ❌ | ✅ |
| **Relatórios com total de páginas** | ❌ | ✅ |

> **Regra prática neste projeto:** Feeds de domínio (refeições, notificações, histórico) usam **cursor**. Painéis administrativos e listagens fixas usam **offset**.

---

### 5.1 Cursor Pagination — Paginação por Cursor

#### DTO de Query

```typescript
// src/common/dtos/cursor-pagination.dto.ts
import { IsOptional, IsInt, Min, Max, IsString } from 'class-validator';
import { Type } from 'class-transformer';

export class CursorPaginationDto {
  @IsOptional()
  @IsString()
  cursor?: string;           // ← ID ou valor codificado do último item recebido

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  @Type(() => Number)
  limit?: number = 20;       // ← padrão: 20 itens por página
}
```

#### Tipo de Metadados de Cursor

```typescript
// src/common/types/pagination.types.ts
export interface CursorPaginationMeta {
  limit: number;
  nextCursor: string | null;  // null = não há próxima página
  prevCursor: string | null;  // null = não há página anterior (opcional para mobile)
  hasNextPage: boolean;
}

export interface CursorPaginatedResult<T> {
  items: T[];
  meta: CursorPaginationMeta;
}
```

#### Implementação no Repository

```typescript
// src/modules/meals/repositories/meals.repository.ts (trecho cursor)
async findByUserIdWithCursor(
  userId: string,
  limit: number,
  cursor?: string,
): Promise<CursorPaginatedResult<MealEntity>> {
  const take = limit + 1; // busca 1 extra para detectar se há próxima página

  const qb = this.repo
    .createQueryBuilder('meal')
    .where('meal.userId = :userId', { userId })
    .orderBy('meal.createdAt', 'DESC')
    .addOrderBy('meal.id', 'DESC') // ← desempate por ID (evita duplicatas em mesmo timestamp)
    .take(take);

  if (cursor) {
    // cursor = base64(createdAt + '|' + id)
    const decoded = Buffer.from(cursor, 'base64').toString('utf-8');
    const [cursorDate, cursorId] = decoded.split('|');
    qb.andWhere(
      '(meal.createdAt < :cursorDate OR (meal.createdAt = :cursorDate AND meal.id < :cursorId))',
      { cursorDate, cursorId },
    );
  }

  const items = await qb.getMany();
  const hasNextPage = items.length > limit;

  if (hasNextPage) items.pop(); // remove o item extra

  const nextCursor =
    hasNextPage && items.length > 0
      ? Buffer.from(
          `${items[items.length - 1].createdAt.toISOString()}|${items[items.length - 1].id}`,
        ).toString('base64')
      : null;

  return {
    items,
    meta: {
      limit,
      nextCursor,
      prevCursor: null, // implementar se necessário (scroll bidirecional)
      hasNextPage,
    },
  };
}
```

#### Use Case com Cursor Pagination

```typescript
// src/modules/meals/use-cases/get-meal-history/get-meal-history.use-case.ts
import { Injectable } from '@nestjs/common';
import { MealsRepository } from '@modules/meals/repositories/meals.repository';
import { CursorPaginatedResult } from '@common/types/pagination.types';
import { MealEntity } from '@modules/meals/entities/meal.entity';

type GetMealHistoryInput = {
  userId: string;
  cursor?: string;
  limit?: number;
};

@Injectable()
export class GetMealHistoryUseCase {
  constructor(private readonly mealsRepository: MealsRepository) {}

  async execute(input: GetMealHistoryInput): Promise<CursorPaginatedResult<MealEntity>> {
    return this.mealsRepository.findByUserIdWithCursor(
      input.userId,
      input.limit ?? 20,
      input.cursor,
    );
  }
}
```

#### Controller com Cursor Pagination

```typescript
@Get()
async findAll(
  @CurrentUser() user: UserPayload,
  @Query() query: CursorPaginationDto,
): Promise<{ data: MealViewModel[]; meta: CursorPaginationMeta }> {
  const result = await this.getMealHistoryUseCase.execute({
    userId: user.id,
    cursor: query.cursor,
    limit: query.limit,
  });

  return {
    data: MealViewModel.fromEntities(result.items),
    meta: result.meta,
  };
}
```

**Chamada do cliente:**
```
GET /api/v1/meals?limit=20
→ { "data": [...], "meta": { "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI0...", "hasNextPage": true } }

GET /api/v1/meals?limit=20&cursor=eyJjcmVhdGVkQXQiOiIyMDI0...
→ { "data": [...], "meta": { "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI1...", "hasNextPage": false } }
```

---

### 5.2 Offset Pagination — Paginação por Página

#### DTO de Query

```typescript
// src/common/dtos/offset-pagination.dto.ts
import { IsOptional, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class OffsetPaginationDto {
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  page?: number = 1;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  @Type(() => Number)
  limit?: number = 20;
}
```

#### Tipo de Metadados de Offset

```typescript
// src/common/types/pagination.types.ts (adicionar)
export interface OffsetPaginationMeta {
  total: number;       // total de registros
  page: number;        // página atual
  limit: number;       // registros por página
  lastPage: number;    // Math.ceil(total / limit)
  hasNextPage: boolean;
  hasPrevPage: boolean;
}

export interface OffsetPaginatedResult<T> {
  items: T[];
  meta: OffsetPaginationMeta;
}
```

#### Utilitário de Paginação

```typescript
// src/common/utils/pagination.util.ts
import { OffsetPaginationMeta } from '@common/types/pagination.types';

export function buildOffsetMeta(
  total: number,
  page: number,
  limit: number,
): OffsetPaginationMeta {
  const lastPage = Math.ceil(total / limit);
  return {
    total,
    page,
    limit,
    lastPage,
    hasNextPage: page < lastPage,
    hasPrevPage: page > 1,
  };
}

export function toSkip(page: number, limit: number): number {
  return (page - 1) * limit;
}
```

#### Implementação no Repository

```typescript
// src/modules/users/repositories/users.repository.ts (trecho offset — ex: admin)
async findAllPaginated(
  page: number,
  limit: number,
): Promise<OffsetPaginatedResult<UserEntity>> {
  const [items, total] = await this.repo.findAndCount({
    order: { createdAt: 'DESC' },
    skip: toSkip(page, limit),
    take: limit,
  });

  return {
    items,
    meta: buildOffsetMeta(total, page, limit),
  };
}
```

---

## 6. Anti-Padrões Comuns

### ❌ Lógica do envelope dentro do Controller

```typescript
// ❌ ERRADO — Controller construindo o envelope manualmente
@Get()
async findAll(@CurrentUser() user: UserPayload) {
  const meals = await this.getMealHistoryUseCase.execute({ userId: user.id });
  return {                             // ← não faça isso
    success: true,
    data: MealViewModel.fromEntities(meals),
    statusCode: 200,
  };
}

// ✅ CORRETO — Controller retorna apenas o ViewModel; o interceptor envolve
@Get()
async findAll(@CurrentUser() user: UserPayload) {
  const result = await this.getMealHistoryUseCase.execute({ userId: user.id });
  return {
    data: MealViewModel.fromEntities(result.items),
    meta: result.meta,
  };
}
```

### ❌ Usar status HTTP errado

```typescript
// ❌ ERRADO
@Post()
async create() { ... } // ← 200 por padrão, mas criação deve ser 201

// ✅ CORRETO
@Post()
@HttpCode(HttpStatus.CREATED)
async create() { ... }
```

### ❌ Rota com verbo no path

```typescript
// ❌ ERRADO — verbo no path
@Controller('getMeals')
@Get('createMeal')

// ✅ CORRETO — verbo no método HTTP
@Controller('meals')
@Post()
```

### ❌ Construir o objeto de erro manualmente

```typescript
// ❌ ERRADO — formato inconsistente
return { success: false, message: 'Not found' }; //← não usa o filter global

// ✅ CORRETO — NestJS + filter global formatam automaticamente
throw new NotFoundException('Refeição não encontrada');
```

### ❌ Expor Entity diretamente na resposta (sem ViewModel)

```typescript
// ❌ ERRADO — Entity contém campos internos do banco
@Get(':id')
async findOne(@Param('id') id: string): Promise<MealEntity> { ... }

// ✅ CORRETO — ViewModel define o contrato da API
@Get(':id')
async findOne(@Param('id') id: string): Promise<MealViewModel> {
  const meal = await this.getMealUseCase.execute({ id });
  return MealViewModel.fromEntity(meal);
}
```

---

## 7. Checklist de Validação

### Versionamento
- [ ] `app.enableVersioning({ type: VersioningType.URI })` está no `main.ts`?
- [ ] `app.setGlobalPrefix('api')` definido?
- [ ] Todo controller de domínio tem `version: '1'` (ou versão correspondente)?
- [ ] Endpoints de infra (`/health`) usam `VERSION_NEUTRAL`?

### Nomenclatura
- [ ] Rotas estão em kebab-case e plural?
- [ ] Nenhum verbo no path (exceto sub-ações como `share`, `publish`)?
- [ ] Classes de Controller em PascalCase com sufixo `Controller`?
- [ ] Arquivos em kebab-case com sufixo `.controller.ts`?

### Status HTTP
- [ ] `@HttpCode(HttpStatus.CREATED)` nos endpoints `@Post()`?
- [ ] `@HttpCode(HttpStatus.NO_CONTENT)` nos endpoints `@Delete()` sem body?
- [ ] Erros lançam exceções do `@nestjs/common` (nunca `return { error }` manual)?

### Envelope
- [ ] `TransformInterceptor` registrado globalmente no `main.ts`?
- [ ] `HttpExceptionFilter` registrado globalmente no `main.ts`?
- [ ] Controllers retornam apenas ViewModels (para itens) ou `{ data, meta }` (para listas)?
- [ ] Nenhum Controller constrói o envelope manualmente?

### Paginação
- [ ] Feeds e histórico usam cursor pagination?
- [ ] Admin e listagens fixas usam offset pagination?
- [ ] Query params validados com `class-validator` + `@Type(() => Number)`?
- [ ] `ValidationPipe` com `transform: true` no `main.ts` para conversão de tipos?
- [ ] Campos de cursor indexados no banco de dados?

---

## Referências

- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — `src/common/interceptors/`, `src/common/filters/`, path aliases, nomenclatura geral
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Controllers finos, ViewModel como shape antes do envelope
- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — `env.PORT` via `@config/env.config`
- [NestJS Docs — Versioning](https://docs.nestjs.com/techniques/versioning)
- [NestJS Docs — Interceptors](https://docs.nestjs.com/interceptors)
- [NestJS Docs — Exception Filters](https://docs.nestjs.com/exception-filters)
- [NestJS Docs — Validation](https://docs.nestjs.com/techniques/validation)
