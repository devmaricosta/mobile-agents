---
name: openapi-docs
description: Documentação automática com @nestjs/swagger — configuração do SwaggerModule no main.ts (separada do swagger.config.ts), decorators em controllers (@ApiTags, @ApiOperation, @ApiResponse, @ApiBearerAuth), decorators em DTOs (@ApiProperty, @ApiPropertyOptional), separação por tags de domínio alinhada ao módulo NestJS, como documentar autenticação JWT, múltiplos documentos por versão e geração automática de SDK cliente. Complementa rest-api-patterns.md (versionamento /v1/, envelope { data, meta, error }), dto-validation.md (DTOs decorados com class-validator e @ApiProperty juntos) e nest-project-structure.md (swagger.config.ts em src/config/).
---

# Documentação Automática com @nestjs/swagger

Você é um Arquiteto de Software Senior. Esta skill define **como documentar automaticamente a API NestJS** com Swagger/OpenAPI. Ela complementa:

- `rest-api-patterns.md` — versionamento URI (`/v1/`), envelope `{ data, meta, error }` e `main.ts`
- `dto-validation.md` — DTOs de entrada com `class-validator`; os mesmos DTOs recebem `@ApiProperty`
- `nest-project-structure.md` — `swagger.config.ts` em `src/config/`, path aliases `@config/`

> **Regra fundamental:** A documentação Swagger deve ser um reflexo fiel do contrato da API. Cada campo do DTO, cada possível resposta e cada requisito de autenticação devem estar documentados. Documentação incompleta é documentação enganosa.

---

## 1. Instalação

```bash
npm install @nestjs/swagger swagger-ui-express
```

> **Atenção:** O `@nestjs/swagger` requer `emitDecoratorMetadata: true` e `experimentalDecorators: true` no `tsconfig.json` — já definidos em `nest-project-structure.md`.

---

## 2. Configuração — `swagger.config.ts`

A configuração do Swagger é centralizada em `src/config/swagger.config.ts`, conforme a estrutura de pastas definida em `nest-project-structure.md`.

```typescript
// src/config/swagger.config.ts
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { INestApplication } from '@nestjs/common';
import { env } from '@config/env.config';

/**
 * Configura e monta o Swagger UI na aplicação NestJS.
 * Chamado no main.ts APENAS em ambientes não-produtivos
 * (ou com feature flag para produção restrita).
 */
export function setupSwagger(app: INestApplication): void {
  const config = new DocumentBuilder()
    .setTitle('CAB API')
    .setDescription(
      'Documentação da API do projeto CAB. ' +
      'Todos os endpoints estão sob `/api/v{n}/`.',
    )
    .setVersion('1.0')
    .setContact(
      'Time de Engenharia',
      'https://github.com/org/repo',
      'engenharia@empresa.com',
    )
    .addServer(`http://localhost:${env.PORT}`, 'Local')
    .addServer('https://staging.api.empresa.com', 'Staging')
    // ← Esquema de autenticação JWT (Bearer Token)
    // O nome 'access-token' é referenciado nos decorators @ApiBearerAuth()
    .addBearerAuth(
      {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
        description: 'Informe o JWT obtido em POST /api/v1/auth/login',
      },
      'access-token', // ← nome do security scheme (referenciado nos decorators)
    )
    // ← Tags de domínio globais — cada módulo NestJS mapeia para uma tag
    .addTag('auth', 'Autenticação e gerenciamento de sessão')
    .addTag('users', 'Cadastro e perfil de usuários')
    .addTag('meals', 'Refeições e histórico nutricional')
    .build();

  const document = SwaggerModule.createDocument(app, config);

  // Swagger disponível em /api/docs
  SwaggerModule.setup('api/docs', app, document, {
    swaggerOptions: {
      persistAuthorization: true,    // mantém o token ao recarregar a página
      defaultModelsExpandDepth: 2,   // expande schemas de modelos automaticamente
      docExpansion: 'list',          // 'list' | 'full' | 'none'
      filter: true,                  // habilita busca de endpoints
      showRequestDuration: true,     // exibe tempo de resposta nas requisições de teste
    },
    customSiteTitle: 'CAB API Docs',
  });
}
```

---

## 3. Integração no `main.ts`

A chamada do Swagger é adicionada ao `main.ts` **depois** de toda a configuração global (pipes, interceptors, filters), conforme o padrão já definido em `rest-api-patterns.md` e `dto-validation.md`.

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { VersioningType, ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { TransformInterceptor } from '@common/interceptors/transform.interceptor';
import { HttpExceptionFilter } from '@common/filters/http-exception.filter';
import { env } from '@config/env.config';
import { setupSwagger } from '@config/swagger.config';

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
  app.useGlobalFilters(new HttpExceptionFilter());

  // ← Swagger apenas fora de produção (ou configurável via env)
  if (env.NODE_ENV !== 'production') {
    setupSwagger(app);
  }

  await app.listen(env.PORT);
  console.log(`🚀 API rodando em http://localhost:${env.PORT}/api/v1`);

  if (env.NODE_ENV !== 'production') {
    console.log(`📖 Swagger em http://localhost:${env.PORT}/api/docs`);
  }
}
bootstrap();
```

> **Por que separar em `swagger.config.ts`?** O `main.ts` já tem responsabilidades suficientes (pipes, guards, interceptors). Separar o Swagger mantém o arquivo enxuto e facilita testes de configuração.

---

## 4. Decorators em Controllers

### 4.1 `@ApiTags` — Separação por Domínio

Cada controller recebe **exatamente uma tag**, alinhada ao domínio do módulo NestJS. A tag deve coincidir com a declarada no `DocumentBuilder.addTag()` em `swagger.config.ts`.

```typescript
// src/modules/meals/controllers/meals.controller.ts
import { Controller, Get, Post, Body, Param, Query, HttpCode, HttpStatus } from '@nestjs/common';
import {
  ApiTags,
  ApiBearerAuth,
  ApiOperation,
  ApiResponse,
  ApiCreatedResponse,
  ApiOkResponse,
  ApiNotFoundResponse,
  ApiUnauthorizedResponse,
  ApiBadRequestResponse,
} from '@nestjs/swagger';
import { CreateMealDto } from '@modules/meals/dtos/create-meal.dto';
import { MealViewModel } from '@modules/meals/view-models/meal.view-model';

@ApiTags('meals')              // ← tag de domínio (1 por controller)
@ApiBearerAuth('access-token') // ← todos os endpoints deste controller requerem JWT
@Controller({ path: 'meals', version: '1' })
export class MealsController {

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({
    summary: 'Registrar refeição',
    description: 'Registra uma nova refeição para o usuário autenticado. '
      + 'Retorna o ViewModel da refeição criada dentro de `{ data: ... }`.',
  })
  @ApiCreatedResponse({
    description: 'Refeição criada com sucesso.',
    type: MealViewModel,
  })
  @ApiBadRequestResponse({ description: 'Payload inválido (campos faltando ou tipo errado).' })
  @ApiUnauthorizedResponse({ description: 'Token ausente, expirado ou inválido.' })
  async create(@Body() dto: CreateMealDto) {
    // ...
  }

  @Get(':id')
  @ApiOperation({ summary: 'Buscar refeição por ID' })
  @ApiOkResponse({ description: 'Refeição encontrada.', type: MealViewModel })
  @ApiNotFoundResponse({ description: 'Refeição não encontrada.' })
  @ApiUnauthorizedResponse({ description: 'Token ausente ou inválido.' })
  async findOne(@Param('id') id: string) {
    // ...
  }

  @Get()
  @ApiOperation({
    summary: 'Listar refeições com paginação por cursor',
    description: 'Retorna a lista de refeições paginada por cursor, '
      + 'conforme padrão definido em `rest-api-patterns.md`.',
  })
  @ApiOkResponse({
    description: 'Lista de refeições. `meta` contém `nextCursor` e `hasNextPage`.',
    schema: {
      properties: {
        data: { type: 'array', items: { $ref: '#/components/schemas/MealViewModel' } },
        meta: {
          type: 'object',
          properties: {
            nextCursor: { type: 'string', nullable: true, example: 'eyJjcmVhdGVkQXQi...' },
            hasNextPage: { type: 'boolean', example: true },
            limit: { type: 'number', example: 20 },
          },
        },
      },
    },
  })
  async findAll(@Query() query: any) {
    // ...
  }
}
```

### 4.2 Regras de uso dos decorators de resposta

| Decorator | Quando usar |
|-----------|-------------|
| `@ApiCreatedResponse` | POST que retorna o recurso criado (201) |
| `@ApiOkResponse` | GET, PATCH, PUT com resposta (200) |
| `@ApiNoContentResponse` | DELETE sem body (204) |
| `@ApiBadRequestResponse` | Erros de validação do `ValidationPipe` (400) |
| `@ApiUnauthorizedResponse` | Token ausente/inválido (401) |
| `@ApiForbiddenResponse` | Usuário sem permissão (403) |
| `@ApiNotFoundResponse` | Recurso inexistente (404) |
| `@ApiConflictResponse` | Conflito de dados, ex: e-mail duplicado (409) |
| `@ApiUnprocessableEntityResponse`| Regra de negócio violada (422) |

> **Regra prática:** Documente pelo menos os códigos de sucesso e os de erro mais prováveis. Não é necessário documentar todos os 4xx/5xx possíveis — foque nos que o cliente precisa tratar.

---

## 5. Decorators em DTOs

### 5.1 `@ApiProperty` e `@ApiPropertyOptional` — Campos do DTO

Os mesmos DTOs já decorados com `class-validator` (conforme `dto-validation.md`) recebem adicionalmente `@ApiProperty` para gerar o schema OpenAPI correto. **Os dois conjuntos de decorators coexistem no mesmo campo.**

```typescript
// src/modules/meals/dtos/create-meal.dto.ts
import {
  IsEnum,
  IsInt,
  IsOptional,
  IsArray,
  ValidateNested,
  Min,
  Max,
} from 'class-validator';
import { Type } from 'class-transformer';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export enum MealMode {
  QUICK = 'quick',
  CUSTOM = 'custom',
}

export class MealFoodItemDto {
  @ApiProperty({ description: 'ID do alimento no banco', example: 'food-uuid-123' })
  foodId: string;

  @ApiProperty({ description: 'Quantidade consumida', example: 150, minimum: 1 })
  @IsInt()
  @Min(1)
  quantity: number;

  @ApiProperty({ description: 'Unidade de medida', example: 'g', enum: ['g', 'ml', 'unidade'] })
  unit: string;
}

export class CreateMealDto {
  @ApiProperty({
    description: 'Modo de registro da refeição',
    enum: MealMode,
    example: MealMode.QUICK,
  })
  @IsEnum(MealMode)
  mode: MealMode;

  @ApiProperty({
    description: 'Número total de blocos nutricionais',
    example: 4,
    minimum: 1,
    maximum: 20,
  })
  @IsInt()
  @Min(1)
  @Max(20)
  blocks: number;

  @ApiProperty({
    description: 'Itens alimentares da refeição',
    type: [MealFoodItemDto],
  })
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => MealFoodItemDto)
  items: MealFoodItemDto[];

  @ApiPropertyOptional({
    description: 'Observações livres sobre a refeição',
    example: 'Refeição pós-treino',
    maxLength: 500,
  })
  @IsOptional()
  notes?: string;
}
```

### 5.2 Documentando tipos complexos

```typescript
// Enum com @ApiProperty
@ApiProperty({
  enum: UserRole,
  enumName: 'UserRole',  // ← gera um schema separado em components/schemas
  example: UserRole.USER,
})
@IsEnum(UserRole)
role: UserRole;

// Array de primitivos
@ApiProperty({
  type: [String],
  description: 'Lista de tags da refeição',
  example: ['café', 'lanche'],
})
@IsArray()
@IsString({ each: true })
tags: string[];

// Objeto aninhado opcional
@ApiPropertyOptional({
  type: MealFoodItemDto,
  description: 'Item principal da refeição (modo quick)',
})
@IsOptional()
@ValidateNested()
@Type(() => MealFoodItemDto)
mainItem?: MealFoodItemDto;
```

### 5.3 `PartialType` + `IntersectionType` do `@nestjs/swagger`

> **Importante:** Para `UpdateXxxDto` usar `PartialType` e manter a documentação Swagger correta, importe `PartialType` de `@nestjs/swagger` **e não de** `@nestjs/mapped-types`. O `@nestjs/swagger` reexporta um `PartialType` que propaga também os decorators `@ApiProperty`.

```typescript
// src/modules/meals/dtos/update-meal.dto.ts
import { PartialType, OmitType } from '@nestjs/swagger'; // ← IMPORTANTE: de @nestjs/swagger
import { CreateMealDto } from './create-meal.dto';

export class UpdateMealDto extends PartialType(
  OmitType(CreateMealDto, ['mode'] as const), // modo não pode mudar após criação
) {}
// Resultado: blocks?, items?, notes? — todos opcionais, com @ApiProperty herdados
```

---

## 6. Documentando Autenticação JWT

### 6.1 Configuração do security scheme (já no swagger.config.ts)

```typescript
// Em swagger.config.ts — DocumentBuilder
.addBearerAuth(
  {
    type: 'http',
    scheme: 'bearer',
    bearerFormat: 'JWT',
    description: 'Informe o JWT obtido em POST /api/v1/auth/login',
  },
  'access-token', // ← nome do security scheme
)
```

### 6.2 Controllers protegidos — `@ApiBearerAuth()` no nível do controller

```typescript
// ✅ CORRETO — aplicar no nível do controller (protege todos os endpoints)
@ApiTags('meals')
@ApiBearerAuth('access-token') // ← nome deve bater com o declarado no DocumentBuilder
@Controller({ path: 'meals', version: '1' })
export class MealsController { ... }
```

### 6.3 Endpoints públicos dentro de controller protegido

Se um controller tem a maioria dos endpoints protegidos mas alguns são públicos (ex: rota de saúde), use o decorator `@ApiSecurity` no nível do método para sobrescrever:

```typescript
@ApiTags('auth')
@Controller({ path: 'auth', version: '1' })
export class AuthController {

  // ← público: sem @ApiBearerAuth no controller nem no método
  @Post('login')
  @ApiOperation({ summary: 'Autenticar usuário' })
  @ApiOkResponse({ type: AuthResponseDto })
  @ApiBadRequestResponse({ description: 'Credenciais inválidas' })
  async login(@Body() dto: LoginDto) { ... }

  // ← protegido: override explícito de segurança no método
  @Get('me')
  @ApiBearerAuth('access-token')
  @ApiOperation({ summary: 'Perfil do usuário autenticado' })
  @ApiOkResponse({ type: UserViewModel })
  @ApiUnauthorizedResponse({ description: 'Token inválido' })
  async me() { ... }
}
```

### 6.4 Resposta de autenticação

```typescript
// src/modules/auth/dtos/auth-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class AuthResponseDto {
  @ApiProperty({ description: 'JWT de acesso (validade: 7d)', example: 'eyJhbGci...' })
  accessToken: string;

  @ApiProperty({ description: 'JWT de refresh (validade: 30d)', example: 'eyJhbGci...' })
  refreshToken: string;

  @ApiProperty({ description: 'Tipo do token', example: 'Bearer' })
  tokenType: string;

  @ApiProperty({ description: 'Validade do accessToken em segundos', example: 604800 })
  expiresIn: number;
}
```

---

## 7. ViewModels como Tipos de Resposta

Os **ViewModels** (camada de apresentação, conforme `clean-architecture.md`) são as classes que representam o shape da resposta. Eles também recebem `@ApiProperty` para que o Swagger os documente corretamente.

```typescript
// src/modules/meals/view-models/meal.view-model.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { MealEntity } from '@modules/meals/entities/meal.entity';

export class MealViewModel {
  @ApiProperty({ description: 'ID único da refeição', example: 'meal-uuid-123' })
  id: string;

  @ApiProperty({ description: 'Número de blocos nutricionais', example: 4 })
  blocks: number;

  @ApiProperty({ description: 'Modo de registro', enum: ['quick', 'custom'], example: 'quick' })
  mode: string;

  @ApiProperty({ description: 'Data/hora de criação (ISO 8601)', example: '2024-01-15T10:00:00.000Z' })
  createdAt: string;

  @ApiPropertyOptional({ description: 'Observações da refeição', example: 'Pós-treino' })
  notes?: string;

  static fromEntity(entity: MealEntity): MealViewModel {
    const vm = new MealViewModel();
    vm.id = entity.id;
    vm.blocks = entity.blocks;
    vm.mode = entity.mode;
    vm.createdAt = entity.createdAt;
    vm.notes = entity.notes;
    return vm;
  }

  static fromEntities(entities: MealEntity[]): MealViewModel[] {
    return entities.map(MealViewModel.fromEntity);
  }
}
```

> **Regra:** Use `type: MealViewModel` nos decorators de resposta (`@ApiOkResponse`, `@ApiCreatedResponse`) — o Swagger inferirá o schema automaticamente a partir dos `@ApiProperty` da classe.

---

## 8. Versionamento da Spec / Múltiplos Documentos

Quando a API tem múltiplas versões com contratos diferentes (ex: `v1` e `v2`), é possível gerar documentos separados para cada versão.

```typescript
// src/config/swagger.config.ts — múltiplos documentos
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { INestApplication } from '@nestjs/common';
import { env } from '@config/env.config';

export function setupSwagger(app: INestApplication): void {
  // Configuração base reutilizável
  const baseConfig = {
    contact: { name: 'Time de Engenharia', email: 'engenharia@empresa.com' },
    bearerAuth: { type: 'http' as const, scheme: 'bearer', bearerFormat: 'JWT' },
  };

  // ─── Documento v1 ────────────────────────────────────────────────────────
  const configV1 = new DocumentBuilder()
    .setTitle('CAB API v1')
    .setDescription('Endpoints estáveis da versão 1.')
    .setVersion('1.0')
    .addBearerAuth(baseConfig.bearerAuth, 'access-token')
    .addTag('auth')
    .addTag('users')
    .addTag('meals')
    .build();

  const documentV1 = SwaggerModule.createDocument(app, configV1, {
    include: [], // ← deixe vazio para incluir tudo; ou informe módulos específicos
    deepScanRoutes: true,
  });

  SwaggerModule.setup('api/docs/v1', app, documentV1, {
    swaggerOptions: { persistAuthorization: true, filter: true },
    customSiteTitle: 'CAB API v1 Docs',
  });

  // ─── Documento v2 (quando existir) ───────────────────────────────────────
  // const configV2 = new DocumentBuilder()
  //   .setTitle('CAB API v2')
  //   .setVersion('2.0')
  //   ...
  //   .build();
  // const documentV2 = SwaggerModule.createDocument(app, configV2, { ... });
  // SwaggerModule.setup('api/docs/v2', app, documentV2);

  // ─── Documento unificado (todas as versões) ────────────────────────────
  const configAll = new DocumentBuilder()
    .setTitle('CAB API — Todas as versões')
    .setVersion('latest')
    .addBearerAuth(baseConfig.bearerAuth, 'access-token')
    .build();

  const documentAll = SwaggerModule.createDocument(app, configAll, {
    deepScanRoutes: true,
  });
  SwaggerModule.setup('api/docs', app, documentAll, {
    swaggerOptions: { persistAuthorization: true, filter: true, docExpansion: 'list' },
    customSiteTitle: 'CAB API Docs',
  });
}
```

### Excluir um endpoint do Swagger

```typescript
import { ApiExcludeEndpoint } from '@nestjs/swagger';

@Get('internal-health')
@ApiExcludeEndpoint() // ← não aparece no Swagger UI (ex: endpoints internos, webhooks)
async internalHealth() { ... }
```

---

## 9. Geração Automática de SDK Cliente

O documento OpenAPI gerado pode ser usado para gerar um SDK TypeScript automaticamente para o frontend ou app mobile.

### 9.1 Exportar o arquivo `openapi.json`

```typescript
// src/config/swagger.config.ts — adicione ao final da função setupSwagger
import * as fs from 'fs';
import * as path from 'path';

export function setupSwagger(app: INestApplication): void {
  // ... configuração existente ...

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);

  // ← Exporta spec para arquivo (útil em CI/CD para geração de SDK)
  if (env.NODE_ENV === 'development') {
    const outputPath = path.join(process.cwd(), 'openapi.json');
    fs.writeFileSync(outputPath, JSON.stringify(document, null, 2));
    console.log(`📄 OpenAPI spec exportada em: ${outputPath}`);
  }
}
```

### 9.2 Geração de SDK com `openapi-typescript-codegen`

```bash
# Instalar gerador (dev dependency)
npm install -D openapi-typescript-codegen

# Gerar SDK a partir do arquivo local
npx openapi-typescript-codegen \
  --input ./openapi.json \
  --output ./sdk \
  --client fetch \
  --useOptions \
  --useUnionTypes

# Ou a partir do servidor rodando
npx openapi-typescript-codegen \
  --input http://localhost:3000/api/docs-json \
  --output ./sdk \
  --client fetch
```

### 9.3 Script no `package.json`

```json
{
  "scripts": {
    "sdk:generate": "openapi-typescript-codegen --input http://localhost:3000/api/docs-json --output ./sdk --client fetch --useOptions --useUnionTypes",
    "sdk:generate:local": "openapi-typescript-codegen --input ./openapi.json --output ./sdk --client fetch"
  }
}
```

> **Regra de integração com o frontend:** O SDK gerado é consumido diretamente no app React Native/Next.js. Alterar o contrato de um endpoint sem atualizar a documentação e regenerar o SDK resultará em erros de runtime. **Trate a spec OpenAPI como código.**

---

## 10. Documentando o Envelope de Resposta

O `TransformInterceptor` (definido em `rest-api-patterns.md`) envolve todas as respostas em `{ data, meta }`. Para refletir isso corretamente no Swagger, use schemas inline nos decorators de resposta quando necessário.

```typescript
// Para endpoints que retornam envelope com meta (paginação)
@ApiOkResponse({
  description: 'Lista paginada de refeições.',
  schema: {
    type: 'object',
    required: ['data'],
    properties: {
      data: {
        type: 'array',
        items: { $ref: '#/components/schemas/MealViewModel' },
      },
      meta: {
        type: 'object',
        properties: {
          nextCursor: { type: 'string', nullable: true, example: 'eyJjcmVhdGVk...' },
          hasNextPage: { type: 'boolean', example: true },
          limit: { type: 'number', example: 20 },
        },
      },
    },
  },
})

// Para endpoints de item único (o interceptor adiciona { data: ... } automaticamente)
// Documente apenas o ViewModel; o envelope é comportamento do interceptor
@ApiOkResponse({ type: MealViewModel })
```

---

## 11. Anti-Padrões Comuns

### ❌ `@ApiProperty` sem `example`

```typescript
// ❌ ERRADO — sem exemplo, o Swagger gera schema vazio
@ApiProperty({ description: 'ID do usuário' })
userId: string;

// ✅ CORRETO — com exemplo realista
@ApiProperty({ description: 'ID único do usuário (UUID v4)', example: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890' })
userId: string;
```

### ❌ `@ApiTags` duplicada entre controller e método

```typescript
// ❌ ERRADO — tags duplicadas geram seção duplicada no Swagger UI
@ApiTags('meals')
@Controller('meals')
export class MealsController {
  @Get()
  @ApiTags('meals') // ← redundante
  findAll() {}
}

// ✅ CORRETO — @ApiTags apenas no controller
@ApiTags('meals')
@Controller('meals')
export class MealsController {
  @Get()
  findAll() {}
}
```

### ❌ `PartialType` de `@nestjs/mapped-types` em vez de `@nestjs/swagger`

```typescript
// ❌ ERRADO — perde os @ApiProperty no schema do Swagger
import { PartialType } from '@nestjs/mapped-types';
export class UpdateMealDto extends PartialType(CreateMealDto) {}

// ✅ CORRETO — preserva @ApiProperty
import { PartialType } from '@nestjs/swagger';
export class UpdateMealDto extends PartialType(CreateMealDto) {}
```

### ❌ Swagger habilitado incondicionalmente em produção

```typescript
// ❌ ERRADO — expõe contratos internos em produção
setupSwagger(app); // sem verificação de ambiente

// ✅ CORRETO — condicional por ambiente
if (env.NODE_ENV !== 'production') {
  setupSwagger(app);
}
```

### ❌ Documentar o ViewModel com `@ApiProperty` no Controller (inline)

```typescript
// ❌ ERRADO — schema inline no decorator do controller
@ApiOkResponse({ schema: { properties: { id: { type: 'string' }, blocks: { type: 'number' } } } })

// ✅ CORRETO — referenciar a classe ViewModel decorada
@ApiOkResponse({ type: MealViewModel })
```

---

## 12. Checklist de Documentação

### Controller
- [ ] `@ApiTags('dominio')` no nível do controller (uma por controller)?
- [ ] `@ApiBearerAuth('access-token')` em controllers protegidos por JWT?
- [ ] `@ApiOperation({ summary, description })` em cada endpoint?
- [ ] Pelo menos a resposta de sucesso documentada com `@ApiXxxResponse`?
- [ ] Respostas de erro prováveis documentadas (`401`, `404`, `400`)?

### DTOs
- [ ] Todo campo obrigatório tem `@ApiProperty({ description, example })`?
- [ ] Todo campo opcional tem `@ApiPropertyOptional({ description, example })`?
- [ ] Enums documentados com `enum: MinhaEnum, enumName: 'NomeDaEnum'`?
- [ ] Arrays de objetos documentados com `type: [MinhaClasse]`?
- [ ] `UpdateXxxDto` usa `PartialType` de `@nestjs/swagger` (não de `@nestjs/mapped-types`)?

### ViewModels
- [ ] Cada campo do ViewModel tem `@ApiProperty`?
- [ ] `type: MealViewModel` nos `@ApiXxxResponse` dos controllers?

### Configuração
- [ ] `setupSwagger()` chamado no `main.ts` condicionalmente (não em produção)?
- [ ] Security scheme `'access-token'` declarado no `DocumentBuilder`?
- [ ] Tags de domínio declaradas no `DocumentBuilder` correspondem aos `@ApiTags`?
- [ ] URL do Swagger (`/api/docs`) definida e não conflita com rotas da API?

---

## Referências

- [rest-api-patterns.md](.agent/skills/backend/api-and-contracts/rest-api-patterns.md) — Versionamento URI (`/v1/`), envelope `{ data, meta, error }`, configuração do `main.ts`
- [dto-validation.md](.agent/skills/backend/api-and-contracts/dto-validation.md) — DTOs com `class-validator`; `@ApiProperty` coexiste com validações
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — ViewModels como shape de saída dos Controllers
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — `swagger.config.ts` em `src/config/`, path aliases
- [@nestjs/swagger — Documentação Oficial](https://docs.nestjs.com/openapi/introduction)
- [openapi-typescript-codegen — GitHub](https://github.com/ferdikoomen/openapi-typescript-codegen)
