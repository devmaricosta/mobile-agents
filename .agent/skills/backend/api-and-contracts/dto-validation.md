---
name: dto-validation
description: Validação de entrada com class-validator + class-transformer — onde ficam os DTOs, padrão de nomenclatura (CreateUserDto, UpdateUserDto), validação aninhada com @ValidateNested() + @Type(), custom validators com @ValidatorConstraint, e como o ValidationPipe global deve ser configurado no main.ts. Complementa clean-architecture.md (DTOs na camada de apresentação, pasta dtos/), rest-api-patterns.md (configuração do ValidationPipe global e formato dos erros 400) e nest-project-structure.md (convenção de nomenclatura e path alias @modules/).
---

# Validação de Entrada — class-validator + class-transformer

Você é um Arquiteto de Software Senior. Esta skill define **como validar dados de entrada** em toda a aplicação NestJS. Ela complementa `clean-architecture.md` (DTOs pertencem à camada de apresentação, ficam em `dtos/`), `rest-api-patterns.md` (o `ValidationPipe` global está configurado no `main.ts`, e erros de validação retornam `400 BAD_REQUEST` pelo `HttpExceptionFilter`) e `nest-project-structure.md` (convenção de nomenclatura e aliases).

> **Regra fundamental:** DTOs são o contrato de entrada do Controller. Toda validação de formato e tipo deve acontecer no DTO — nunca no UseCase ou no Repository. A validação de regras de negócio é responsabilidade do UseCase.

---

## 1. Instalação

```bash
npm install class-validator class-transformer
```

> **Atenção:** `reflect-metadata` já é instalado pelo NestJS. Verifique que `emitDecoratorMetadata: true` e `experimentalDecorators: true` estão no `tsconfig.json` — conforme definido em `nest-project-structure.md`.

---

## 2. Onde ficam os DTOs

Conforme `clean-architecture.md` e `nest-project-structure.md`:

```
src/modules/<domínio>/
└── dtos/
    ├── create-<domínio>.dto.ts    ← payload de criação (POST)
    ├── update-<domínio>.dto.ts    ← payload de atualização parcial (PATCH)
    └── filter-<domínio>.dto.ts    ← query params de listagem/filtro (GET)
```

DTOs compartilhados entre módulos (ex: paginação) ficam em:

```
src/common/dtos/
├── cursor-pagination.dto.ts       ← já definido em rest-api-patterns.md
└── offset-pagination.dto.ts       ← já definido em rest-api-patterns.md
```

---

## 3. Convenção de Nomenclatura

Consistente com `nest-project-structure.md`:

| Operação | Nome da Classe | Arquivo |
|----------|---------------|---------|
| Criação (POST body) | `CreateUserDto` | `create-user.dto.ts` |
| Atualização parcial (PATCH body) | `UpdateUserDto` | `update-user.dto.ts` |
| Listagem/filtro (GET query) | `FilterUsersDto` | `filter-users.dto.ts` |
| Paginação por cursor | `CursorPaginationDto` | `cursor-pagination.dto.ts` |
| Paginação por offset | `OffsetPaginationDto` | `offset-pagination.dto.ts` |

```typescript
// ✅ CORRETO
export class CreateUserDto {}      // PostBody
export class UpdateUserDto {}      // PatchBody
export class FilterUsersDto {}     // QueryParams

// ❌ ERRADO
export class UserDTO {}            // sem sufixo Dto, sem operação no nome
export class createUser {}         // camelCase, sem sufixo
export class UserInputData {}      // nome genérico sem convenção
```

---

## 4. Configuração do ValidationPipe Global

> **Atenção:** Esta configuração **já está definida** em `rest-api-patterns.md` (seção 1 — `main.ts`). Esta seção é o detalhamento das opções. **Não duplique a configuração** — use exatamente este bloco no `main.ts`.

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

  // ← ValidationPipe global — valida TODOS os DTOs automaticamente
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,              // remove campos não declarados no DTO
      forbidNonWhitelisted: true,   // retorna 400 se campos desconhecidos enviados
      transform: true,              // converte payload para instância da classe DTO
      transformOptions: {
        enableImplicitConversion: true, // converte tipos primitivos automaticamente (ver seção 7)
      },
    }),
  );

  app.useGlobalInterceptors(new TransformInterceptor());
  app.useGlobalFilters(new HttpExceptionFilter());

  await app.listen(env.PORT);
}
bootstrap();
```

### O que cada opção faz

| Opção | Efeito | Por que usar |
|-------|--------|-------------|
| `whitelist: true` | Remove silenciosamente campos extras do payload | Segurança: evita over-posting |
| `forbidNonWhitelisted: true` | Retorna `400` se campos desconhecidos presentes | Controle explícito do contrato |
| `transform: true` | Usa `plainToInstance` para converter plain object → classe DTO | Necessário para `class-validator` funcionar com instâncias |
| `transformOptions.enableImplicitConversion: true` | Converte tipos primitivos (string → number, string → boolean) automaticamente pelo tipo TypeScript declarado | Evita `@Type(() => Number)` em cada campo numérico de query param |

---

## 5. DTOs Básicos — Padrão por Operação

### 5.1 DTO de Criação (POST)

```typescript
// src/modules/users/dtos/create-user.dto.ts
import {
  IsEmail,
  IsString,
  IsEnum,
  IsOptional,
  MinLength,
  MaxLength,
  IsNotEmpty,
} from 'class-validator';

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
}

export class CreateUserDto {
  @IsEmail({}, { message: 'E-mail deve ser um endereço válido' })
  email: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(100)
  name: string;

  @IsString()
  @MinLength(8, { message: 'Senha deve ter no mínimo 8 caracteres' })
  password: string;

  @IsOptional()
  @IsEnum(UserRole, { message: 'Perfil deve ser "admin" ou "user"' })
  role?: UserRole = UserRole.USER;
}
```

### 5.2 DTO de Atualização Parcial (PATCH) — com `PartialType`

Para atualização parcial, **todos os campos são opcionais**. Use o utilitário `PartialType` do NestJS:

```typescript
// src/modules/users/dtos/update-user.dto.ts
import { PartialType, OmitType } from '@nestjs/swagger'; // ← CORRETO: de @nestjs/swagger
import { CreateUserDto } from './create-user.dto';

// ✅ PartialType: herda todos os decorators de CreateUserDto (class-validator + @ApiProperty),
// mas torna tudo opcional
export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['email', 'password'] as const), // omite campos que não devem ser alterados via PATCH
) {}

// Resultado: name? e role? — ambos opcionais, com as mesmas validações de CreateUserDto
```

> **Importante — `@nestjs/swagger` vs `@nestjs/mapped-types`:** Importe sempre `PartialType` e `OmitType` de `@nestjs/swagger`. Essa versão propaga **tanto** os decorators de `class-validator` quanto os `@ApiProperty`, mantendo o schema Swagger correto nos DTOs derivados. O `@nestjs/mapped-types` só propaga validação, não `@ApiProperty`. Consulte `openapi-docs.md` para mais detalhes.

> **Regra prática:** E-mail e senha geralmente têm endpoints dedicados para alteração (ex: `PATCH /users/me/email`, `POST /users/me/change-password`). Omita-os do `UpdateUserDto` genérico.

### 5.3 DTO de Query Params (GET com filtros)

```typescript
// src/modules/users/dtos/filter-users.dto.ts
import { IsOptional, IsString, IsEnum, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';
import { UserRole } from './create-user.dto';

export class FilterUsersDto {
  @IsOptional()
  @IsString()
  search?: string;              // busca por nome ou e-mail

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  @Type(() => Number)           // necessário para query params (string → number)
  limit?: number = 20;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  page?: number = 1;
}
```

> **Nota sobre `@Type(() => Number)` em query params:** Query params chegam como strings. Mesmo com `enableImplicitConversion: true`, use `@Type(() => Number)` explicitamente para campos numéricos em `@Query()` DTOs — é mais claro e explícito sobre a intenção de conversão.

---

## 6. Validação Aninhada — `@ValidateNested()` + `@Type()`

Quando um DTO contém objetos aninhados, você precisa de **dois decorators juntos**:
1. `@ValidateNested()` — instrui o `class-validator` a validar o objeto interno
2. `@Type(() => ClasseAninhada)` — instrui o `class-transformer` a instanciar a classe correta

```typescript
// src/modules/meals/dtos/create-meal.dto.ts
import { IsString, IsNumber, IsEnum, ValidateNested, IsArray, Min } from 'class-validator';
import { Type } from 'class-transformer';

// DTO aninhado
export class MealFoodItemDto {
  @IsString()
  foodId: string;

  @IsNumber()
  @Min(1)
  quantity: number;

  @IsString()
  unit: string; // 'g', 'ml', 'unidade'
}

// DTO principal com validação aninhada
export class CreateMealDto {
  @IsEnum(['quick', 'custom'])
  mode: 'quick' | 'custom';

  // ← objeto aninhado único
  @ValidateNested()
  @Type(() => MealFoodItemDto)
  mainItem: MealFoodItemDto;

  // ← array de objetos aninhados — use { each: true }
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => MealFoodItemDto)
  additionalItems: MealFoodItemDto[];
}
```

### Por que `@Type()` é obrigatório em validação aninhada

Sem `@Type()`, o `class-transformer` não sabe qual classe instanciar — o objeto aninhado permanece como `plain object` e os decorators do `class-validator` não são aplicados.

```typescript
// ❌ ERRADO — sem @Type(), o aninhado não é validado
export class CreateMealDto {
  @ValidateNested()
  mainItem: MealFoodItemDto; // class-validator ignora as regras de MealFoodItemDto
}

// ✅ CORRETO — ambos os decorators
export class CreateMealDto {
  @ValidateNested()
  @Type(() => MealFoodItemDto) // ← instancia MealFoodItemDto corretamente
  mainItem: MealFoodItemDto;
}
```

---

## 7. Transformações com `class-transformer`

Além de validação, `class-transformer` permite **transformar** os dados de entrada antes que cheguem ao Controller.

### 7.1 `@Transform()` — Transformação customizada

```typescript
import { IsString, IsEmail } from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  // ← normaliza o e-mail para lowercase antes de validar
  @Transform(({ value }) => value?.toLowerCase()?.trim())
  @IsEmail()
  email: string;

  // ← remove espaços extras do nome
  @Transform(({ value }) => value?.trim())
  @IsString()
  name: string;
}
```

### 7.2 `@Type()` — Conversão de tipos primitivos

```typescript
import { IsInt, IsBoolean, IsDate, Min } from 'class-validator';
import { Type } from 'class-transformer';

export class FilterMealsDto {
  // Query param chega como string '2024-01-15' → converte para Date
  @IsOptional()
  @IsDate()
  @Type(() => Date)
  startDate?: Date;

  // Query param chega como string '10' → converte para number
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  limit?: number;

  // Query param chega como string 'true' → converte para boolean
  @IsOptional()
  @IsBoolean()
  @Type(() => Boolean)
  includeArchived?: boolean;
}
```

### 7.3 `@Expose()` e `@Exclude()` — Controle de serialização

> **Nota:** Estes decorators são mais utilizados em **ViewModels** (saída) do que em DTOs (entrada). Consulte `clean-architecture.md` para o padrão de ViewModel.

```typescript
import { Exclude, Expose } from 'class-transformer';

// Exemplo de ViewModel que usa Exclude (não é um DTO de entrada)
export class UserViewModel {
  @Expose()
  id: string;

  @Expose()
  name: string;

  @Exclude() // ← nunca retorna a senha na resposta
  password: string;
}
```

---

## 8. Custom Validators

Quando os decorators built-in do `class-validator` não são suficientes, crie validators customizados. Existem dois padrões:

### 8.1 Custom Validator simples (sem DI)

Para validações puramente baseadas em lógica (sem banco de dados ou serviços):

```typescript
// src/common/validators/is-future-date.validator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';

// 1. Implementa a constraint
@ValidatorConstraint({ name: 'isFutureDate', async: false })
export class IsFutureDateConstraint implements ValidatorConstraintInterface {
  validate(value: any, args: ValidationArguments): boolean {
    if (!(value instanceof Date)) return false;
    return value > new Date();
  }

  defaultMessage(args: ValidationArguments): string {
    return `${args.property} deve ser uma data no futuro`;
  }
}

// 2. Cria o decorator reutilizável
export function IsFutureDate(validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsFutureDateConstraint,
    });
  };
}
```

```typescript
// Uso no DTO
import { IsDate } from 'class-validator';
import { Type } from 'class-transformer';
import { IsFutureDate } from '@common/validators/is-future-date.validator';

export class CreateEventDto {
  @IsDate()
  @Type(() => Date)
  @IsFutureDate({ message: 'A data do evento deve ser no futuro' })
  scheduledAt: Date;
}
```

### 8.2 Custom Validator com injeção de dependência (async + DI)

Para validações que precisam de acesso ao banco (ex: verificar unicidade de e-mail):

```typescript
// src/modules/users/validators/is-email-unique.validator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';
import { Injectable } from '@nestjs/common';
import { UsersRepository } from '@modules/users/repositories/users.repository';

// ← @Injectable() permite que o NestJS injete dependências
@ValidatorConstraint({ name: 'isEmailUnique', async: true })
@Injectable()
export class IsEmailUniqueConstraint implements ValidatorConstraintInterface {
  constructor(private readonly usersRepository: UsersRepository) {}

  async validate(email: string, args: ValidationArguments): Promise<boolean> {
    const user = await this.usersRepository.findByEmail(email);
    return !user; // válido se NÃO existir usuário com esse e-mail
  }

  defaultMessage(args: ValidationArguments): string {
    return 'E-mail já está em uso';
  }
}

// ← Decorator reutilizável
export function IsEmailUnique(validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsEmailUniqueConstraint,
    });
  };
}
```

#### Configuração obrigatória: usar o container do NestJS

Para que custom validators com DI funcionem, o `ValidationPipe` deve usar o container do NestJS:

```typescript
// src/main.ts — ajuste necessário para custom validators com DI
import { useContainer } from 'class-validator';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // ← Permite que class-validator resolva dependências via NestJS DI
  useContainer(app.select(AppModule), { fallbackOnErrors: true });

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  // ... restante do bootstrap
}
```

#### Registrar o Constraint como provider no módulo

```typescript
// src/modules/users/users.module.ts
import { Module } from '@nestjs/common';
import { IsEmailUniqueConstraint } from './validators/is-email-unique.validator';

@Module({
  providers: [
    // use cases e repositories...
    IsEmailUniqueConstraint, // ← registrado como provider para DI funcionar
  ],
})
export class UsersModule {}
```

#### Uso no DTO

```typescript
// src/modules/users/dtos/create-user.dto.ts
import { IsEmail } from 'class-validator';
import { IsEmailUnique } from '../validators/is-email-unique.validator';

export class CreateUserDto {
  @IsEmail()
  @IsEmailUnique({ message: 'Este e-mail já está cadastrado' })
  email: string;
}
```

> **Atenção sobre validators async com DI:** Use com moderação. Cada requisição executa uma query no banco para validar. Para verificações de unicidade críticas, considere também tratar o `ConflictException` no UseCase como camada de segurança adicional.

---

## 9. Onde fica a Validação vs Regra de Negócio

| Tipo de Verificação | Onde faz | Como faz |
|--------------------|----|--------|
| Campo obrigatório | DTO | `@IsNotEmpty()` |
| Tipo correto | DTO | `@IsString()`, `@IsInt()`, `@IsEmail()` |
| Formato correto | DTO | `@IsUUID()`, `@IsUrl()`, `@Matches(/regex/)` |
| Valor dentro de enum | DTO | `@IsEnum(MinhaEnum)` |
| Tamanho mínimo/máximo | DTO | `@MinLength()`, `@MaxLength()`, `@Min()`, `@Max()` |
| Unicidade no banco | DTO (custom async) + UseCase | `@IsEmailUnique()` + `ConflictException` |
| Limite de negócio (ex: máx. 3 refeições/dia) | **UseCase** | `throw new UnprocessableEntityException(...)` |
| Regra entre campos (ex: `endDate >= startDate`) | DTO (`@ValidateIf`) ou **UseCase** | Custom validator ou lógica no `execute()` |

> **400 vs 422:** `ValidationPipe` lança `BadRequestException` (400) para erros de formato. Use `UnprocessableEntityException` (422) no UseCase para regras de negócio que passam na validação estrutural. Conforme definido em `rest-api-patterns.md`.

---

## 10. Localização de Custom Validators

| Caso | Onde vai |
|------|---------|
| Validator **genérico** (reutilizável entre módulos) | `src/common/validators/<nome>.validator.ts` |
| Validator **específico de um domínio** (usa repositório do módulo) | `src/modules/<domínio>/validators/<nome>.validator.ts` |

```
src/
├── common/
│   └── validators/
│       ├── is-future-date.validator.ts    ← sem DI, reutilizável
│       └── is-strong-password.validator.ts
└── modules/
    └── users/
        └── validators/
            └── is-email-unique.validator.ts  ← com DI (UsersRepository)
```

---

## 11. Anti-Padrões Comuns

### ❌ Validação no UseCase em vez do DTO

```typescript
// ❌ ERRADO — validação de formato no UseCase
export class CreateUserUseCase {
  async execute(input: CreateUserInput) {
    if (!input.email.includes('@')) {  // ← deveria estar no DTO
      throw new BadRequestException('E-mail inválido');
    }
  }
}

// ✅ CORRETO — validação de formato no DTO
export class CreateUserDto {
  @IsEmail()
  email: string;
}
```

### ❌ Validação aninhada sem `@Type()`

```typescript
// ❌ ERRADO — ValidateNested sem @Type() não valida o objeto interno
export class CreateMealDto {
  @ValidateNested()
  mainItem: MealFoodItemDto; // class-validator não sabe que é MealFoodItemDto

  // ✅ CORRETO
  @ValidateNested()
  @Type(() => MealFoodItemDto)
  mainItem: MealFoodItemDto;
}
```

### ❌ DTO passado diretamente para o Repository

```typescript
// ❌ ERRADO — DTO vazando para infraestrutura
export class CreateUserUseCase {
  async execute(dto: CreateUserDto) {  // ← recebe DTO
    return this.usersRepository.create(dto); // ← DTO no Repository: PROIBIDO
  }
}

// ✅ CORRETO — Controller mapeia DTO para tipo de domínio
// Controller:
async create(@Body() dto: CreateUserDto, @CurrentUser() user: UserPayload) {
  return this.createUserUseCase.execute({
    ...dto,
    createdBy: user.id, // ← enriquece com dados do JWT
  });
}
// UseCase recebe CreateUserInput (tipo simples), não CreateUserDto
```

### ❌ Mensagem de erro genérica sem contexto

```typescript
// ❌ ERRADO — mensagem sem contexto
@IsString({ message: 'Inválido' })
name: string;

// ✅ CORRETO — mensagem específica e útil
@IsString({ message: 'O nome deve ser uma string' })
@MinLength(2, { message: 'O nome deve ter pelo menos 2 caracteres' })
@MaxLength(100, { message: 'O nome deve ter no máximo 100 caracteres' })
name: string;
```

### ❌ ValidationPipe configurado por rota em vez de globalmente

```typescript
// ❌ ERRADO — ValidationPipe em cada Controller ou rota
@Post()
@UsePipes(new ValidationPipe())  // ← não faça isso
async create(@Body() dto: CreateUserDto) {}

// ✅ CORRETO — ValidationPipe global no main.ts (já definido em rest-api-patterns.md)
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true, ... }));
```

---

## 12. Checklist de Validação

### DTO
- [ ] Arquivo em kebab-case com sufixo `.dto.ts`? (`create-user.dto.ts`)
- [ ] Classe em PascalCase com sufixo `Dto`? (`CreateUserDto`)
- [ ] Todo campo obrigatório tem pelo menos um decorator de validação?
- [ ] Campos opcionais têm `@IsOptional()` antes dos outros decorators?
- [ ] Arrays de objetos aninhados usam `@ValidateNested({ each: true })` + `@Type()`?
- [ ] Query params numéricos têm `@Type(() => Number)` explícito?
- [ ] `UpdateXxxDto` usa `PartialType(CreateXxxDto)` em vez de redefinir campos?
- [ ] `PartialType` e `OmitType` são importados de `@nestjs/swagger` (não de `@nestjs/mapped-types`)?

### Custom Validators
- [ ] Validator genérico está em `src/common/validators/`?
- [ ] Validator com DI está em `src/modules/<domínio>/validators/`?
- [ ] `IsEmailUniqueConstraint` (e similares com DI) está registrado no `providers[]` do módulo?
- [ ] `useContainer(app.select(AppModule), { fallbackOnErrors: true })` está no `main.ts` (se usar validators com DI)?

### Arquitetura
- [ ] DTOs ficam em `src/modules/<domínio>/dtos/` (ou `src/common/dtos/` se reutilizáveis)?
- [ ] DTOs não são passados diretamente para o Repository?
- [ ] Regras de negócio (não de formato) ficam no UseCase, não no DTO?

---

## Referências

- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — DTOs na camada de apresentação, pasta `dtos/`, nunca passar DTO ao Repository
- [rest-api-patterns.md](.agent/skills/backend/api-and-contracts/rest-api-patterns.md) — Configuração completa do `ValidationPipe` global no `main.ts`, formato de erro 400/422
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Convenção de nomenclatura, path aliases `@modules/` e `@common/`
- [openapi-docs.md](.agent/skills/backend/api-and-contracts/openapi-docs.md) — Por que `PartialType` deve vir de `@nestjs/swagger` (propaga `@ApiProperty`)
- [NestJS Docs — Validation](https://docs.nestjs.com/techniques/validation)
- [class-validator — GitHub](https://github.com/typestack/class-validator)
- [class-transformer — GitHub](https://github.com/typestack/class-transformer)
