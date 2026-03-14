---
name: nest-project-structure
description: A skill mais importante do backend. Define a arquitetura base do monolito modular NestJS com Use Cases Pattern: estrutura de pastas por domínio, separação entre modules/, common/, config/ e infra/, use cases como camada de aplicação explícita, configuração do tsconfig com path aliases, e as regras do que pertence a cada camada.
---

# Arquitetura Base — NestJS Monolito Modular com Use Cases

Você é um Arquiteto de Software Senior especialista em NestJS e Clean Architecture. Você deve impor rigorosamente a separação de responsabilidades onde:

- **`src/modules/<domínio>/`** — cada módulo de domínio é isolado e auto-suficiente
- **`use-cases/`** — camada de aplicação explícita, cada use case tem **uma única responsabilidade** e **um único método `execute()`**
- **`src/common/`** — apenas código verdadeiramente transversal, sem lógica de negócio
- **`src/config/`** — toda configuração e variáveis de ambiente centralizadas
- **`src/infra/`** — encapsula conexões com serviços externos

> **Consistência com o Frontend:** Esta arquitetura espelha a filosofia "feature-based" do app React Native. Assim como `src/features/<domínio>/` no mobile, o backend usa `src/modules/<domínio>/` — cada domínio é isolado, auto-suficiente e expõe apenas uma interface pública bem definida via seu `*.module.ts`.

---

## Quando usar esta skill

- **Novo projeto:** Criar a estrutura de pastas do zero.
- **Novo módulo/domínio:** Decidir onde criar arquivos e como organizar o módulo.
- **Novo use case:** Criar um arquivo de use case com `execute()` para uma operação de negócio.
- **Dúvidas arquiteturais:** Onde um arquivo, lógica ou responsabilidade específica deve residir.
- **Code Review:** Validar se o código segue as regras desta arquitetura.
- **Refatoração:** Reorganizar código existente que não segue as regras.

---

## 1. Estrutura de Diretórios (Obrigatória)

```text
src/
├── modules/                        # DOMÍNIOS DE NEGÓCIO (O Coração)
│   ├── auth/
│   │   ├── controllers/
│   │   │   └── auth.controller.ts
│   │   ├── use-cases/              # ← CAMADA DE APLICAÇÃO (Use Cases Pattern)
│   │   │   ├── login/
│   │   │   │   └── login.use-case.ts
│   │   │   ├── register/
│   │   │   │   └── register.use-case.ts
│   │   │   └── refresh-token/
│   │   │       └── refresh-token.use-case.ts
│   │   ├── repositories/
│   │   │   └── auth.repository.ts
│   │   ├── entities/
│   │   │   └── user.entity.ts
│   │   ├── dtos/
│   │   │   ├── login.dto.ts
│   │   │   └── register.dto.ts
│   │   ├── types/
│   │   │   └── auth.types.ts
│   │   ├── __tests__/
│   │   │   ├── login.use-case.spec.ts
│   │   │   ├── register.use-case.spec.ts
│   │   │   └── auth.controller.spec.ts
│   │   └── auth.module.ts
│   │
│   ├── meals/                      # Exemplo: feature de refeições (CAB 2.0)
│   │   ├── controllers/
│   │   │   └── meals.controller.ts
│   │   ├── use-cases/
│   │   │   ├── create-meal/
│   │   │   │   └── create-meal.use-case.ts
│   │   │   ├── get-meal-history/
│   │   │   │   └── get-meal-history.use-case.ts
│   │   │   └── share-meal/
│   │   │       └── share-meal.use-case.ts
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── dtos/
│   │   ├── types/
│   │   ├── __tests__/
│   │   └── meals.module.ts
│   │
│   └── users/
│       └── ...
│
├── common/                         # CÓDIGO TRANSVERSAL (Sem lógica de negócio)
│   ├── decorators/
│   │   ├── current-user.decorator.ts
│   │   └── public.decorator.ts
│   ├── filters/
│   │   └── http-exception.filter.ts
│   ├── guards/
│   │   └── jwt-auth.guard.ts
│   ├── interceptors/
│   │   └── logging.interceptor.ts
│   ├── pipes/
│   │   └── parse-int.pipe.ts
│   └── utils/
│       ├── format-date.util.ts
│       └── pagination.util.ts
│
├── config/                         # CONFIGURAÇÃO CENTRALIZADA
│   ├── env.config.ts               # Leitura e validação de variáveis de ambiente
│   ├── database.config.ts
│   └── swagger.config.ts
│
├── infra/                          # INFRAESTRUTURA E SERVIÇOS EXTERNOS
│   ├── database/
│   │   ├── database.module.ts
│   │   └── migrations/
│   ├── cache/
│   │   └── cache.module.ts
│   └── mailer/
│       ├── mailer.module.ts
│       └── templates/
│
├── app.module.ts                   # Módulo raiz — importa tudo
└── main.ts                         # Bootstrap da aplicação
```

---

## 2. Use Cases Pattern — A Camada de Aplicação

### O que é um Use Case

Um **use case** representa uma única operação de negócio. É uma classe `@Injectable()` com **exatamente um método público**: `execute()`. Ele orquestra o fluxo entre o Controller e o Repository sem conter lógica de infraestrutura.

> **Regra de ouro:** Se você está tentando adicionar um segundo método público ao use case, você precisa de dois use cases.

### Convenção de nomenclatura

| Item | Convenção | Exemplo |
|---|---|---|
| Pasta | kebab-case, nome da operação | `create-meal/`, `login/` |
| Arquivo | `<operação>.use-case.ts` | `create-meal.use-case.ts` |
| Classe | PascalCase + sufixo `UseCase` | `CreateMealUseCase` |
| Teste | mesmo nome + `.spec.ts` | `create-meal.use-case.spec.ts` |

### Estrutura de um Use Case

```typescript
// src/modules/meals/use-cases/create-meal/create-meal.use-case.ts

import { Injectable } from '@nestjs/common';
import { MealsRepository } from '@modules/meals/repositories/meals.repository';
import { CreateMealDto } from '@modules/meals/dtos/create-meal.dto';
import { MealEntity } from '@modules/meals/entities/meal.entity';

// Tipagem explicita da entrada (pode reutilizar DTO ou criar um tipo dedicado)
type CreateMealInput = CreateMealDto & { userId: string };

@Injectable()
export class CreateMealUseCase {
  constructor(private readonly mealsRepository: MealsRepository) {}

  async execute(input: CreateMealInput): Promise<MealEntity> {
    // 1. Validações/regras de negócio (ex: verificar blocos, categorias)
    // 2. Montar a entidade com as regras do domínio
    // 3. Persistir via repository
    // 4. Retornar resultado
    return this.mealsRepository.create({
      ...input,
      createdAt: new Date().toISOString(),
    });
  }
}
```

### Fluxo obrigatório com Use Cases

```
HTTP Request → Controller → UseCase.execute() → Repository → Banco
```

```typescript
// ✅ CORRETO — Controller chama o use case diretamente
@Controller('meals')
@UseGuards(JwtAuthGuard)
export class MealsController {
  constructor(
    private readonly createMealUseCase: CreateMealUseCase,
    private readonly getMealHistoryUseCase: GetMealHistoryUseCase,
    private readonly shareMealUseCase: ShareMealUseCase,
  ) {}

  @Post()
  create(@Body() dto: CreateMealDto, @CurrentUser() user: UserPayload) {
    return this.createMealUseCase.execute({ ...dto, userId: user.id });
  }

  @Get('history')
  getHistory(@CurrentUser() user: UserPayload) {
    return this.getMealHistoryUseCase.execute({ userId: user.id });
  }

  @Post(':id/share')
  share(@Param('id') id: string, @CurrentUser() user: UserPayload) {
    return this.shareMealUseCase.execute({ mealId: id, userId: user.id });
  }
}
```

### Use Case vs Service — Qual usar?

| Cenário | Solução |
|---|---|
| **Operação de negócio única** (criar refeição, logar, compartilhar) | **Use Case** com `execute()` |
| **Serviço de infraestrutura compartilhado** (enviar e-mail, gerar token JWT) | **Service** sem lógica de domínio |
| **Orquestração cross-módulo** (use case que precisa de outro use case) | **Use Case** injetando outro Use Case |

> **Regra:** Services que têm vários métodos de negócio devem ser decompostos em Use Cases. Um `AuthService` com `login()`, `register()`, `refreshToken()` vira `LoginUseCase`, `RegisterUseCase`, `RefreshTokenUseCase`.

---

## 3. Configuração do TypeScript — `tsconfig.json`

O projeto usa **path aliases** para eliminar imports relativos longos (`../../../`) e tornar o código autodocumentável sobre a camada de origem.

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": false,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "forceConsistentCasingInFileNames": false,
    "noFallthroughCasesInSwitch": false,
    "paths": {
      "@modules/*": ["src/modules/*"],
      "@common/*": ["src/common/*"],
      "@config/*": ["src/config/*"],
      "@infra/*": ["src/infra/*"]
    }
  }
}
```

### Uso dos aliases no código

```typescript
// ✅ CORRETO — path alias autodocumentável
import { CreateMealUseCase } from '@modules/meals/use-cases/create-meal/create-meal.use-case';
import { JwtAuthGuard } from '@common/guards/jwt-auth.guard';
import { env } from '@config/env.config';

// ❌ ERRADO — import relativo frágil
import { CreateMealUseCase } from '../../../modules/meals/use-cases/create-meal/create-meal.use-case';
```

> **Nota:** O NestJS CLI gerencia automaticamente os path aliases com `nest start`. Para `ts-node` standalone, instale `tsconfig-paths` e use `ts-node -r tsconfig-paths/register src/main.ts`.

---

## 4. O que Pertence a Cada Camada

### `src/modules/<domínio>/use-cases/` — Camada de Aplicação

| Regra | Descrição |
|---|---|
| **1 use case = 1 arquivo = 1 operação** | `create-meal.use-case.ts` faz apenas `create meal` |
| **1 método público: `execute()`** | Nenhum outro método público na classe |
| **Depende de Repository, nunca de ORM** | Injeta `MealsRepository`, nunca `Repository<MealEntity>` do TypeORM |
| **Pode injetar outros use cases** | `ShareMealUseCase` pode injetar `GetMealUseCase` para buscar antes de compartilhar |
| **Pode injetar Services de infra** | `RegisterUseCase` pode injetar `MailerService` para enviar e-mail de boas-vindas |
| **Não injeta Controllers** | Jamais |

### `src/modules/<domínio>/repositories/` — Acesso ao Banco

A camada de repositório encapsula todo o acesso ao ORM. **Use cases e services nunca interagem com TypeORM/Prisma diretamente.**

```typescript
// ✅ CORRETO — Repository de domínio como abstração
@Injectable()
export class MealsRepository {
  constructor(
    @InjectRepository(MealEntity)
    private readonly repo: Repository<MealEntity>,
  ) {}

  create(data: Partial<MealEntity>): Promise<MealEntity> {
    return this.repo.save(this.repo.create(data));
  }

  findByUserId(userId: string): Promise<MealEntity[]> {
    return this.repo.find({ where: { userId }, order: { createdAt: 'DESC' } });
  }
}
```

### `src/modules/<domínio>/controllers/` — Camada de Apresentação

Controllers são **finos**. Recebem a requisição, extraem dados, delegam ao use case, retornam a resposta. Sem lógica de negócio.

```typescript
// ✅ CORRETO — thin controller
@Post()
create(@Body() dto: CreateMealDto, @CurrentUser() user: UserPayload) {
  return this.createMealUseCase.execute({ ...dto, userId: user.id });
}

// ❌ ERRADO — lógica de negócio no controller
@Post()
async create(@Body() dto: CreateMealDto, @CurrentUser() user: UserPayload) {
  if (dto.blocks <= 0) throw new BadRequestException('Blocos devem ser positivos');
  const meal = await this.createMealUseCase.execute({ ...dto, userId: user.id });
  return { ...meal, summary: this.calculateSummary(meal) }; // ← regra no controller
}
```

### `src/common/` — Código Transversal

Código reutilizável por múltiplos módulos **sem lógica de negócio de nenhum domínio**.

| Pasta | O que vai aqui | O que NÃO vai aqui |
|---|---|---|
| `decorators/` | `@CurrentUser()`, `@Public()`, `@Roles()` | Decorators específicos de um módulo |
| `filters/` | `HttpExceptionFilter` global | Filters com lógica de domínio |
| `guards/` | `JwtAuthGuard`, `RolesGuard` | Guards com regras de negócio |
| `interceptors/` | `LoggingInterceptor`, `TransformInterceptor` | Interceptors com lógica de domínio |
| `pipes/` | `ParseIntPipe`, `ValidationPipe` global | Pipes com validações de negócio |
| `utils/` | Funções puras e genéricas | Funções com regras de negócio |

### `src/config/` — Configuração Centralizada

Nunca acesse `process.env` fora desta pasta.

```typescript
// src/config/env.config.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRATION: z.string().default('7d'),
});

const _env = envSchema.safeParse(process.env);

if (!_env.success) {
  console.error('❌ Variáveis de ambiente inválidas:', _env.error.format());
  process.exit(1);
}

export const env = _env.data;
```

### `src/infra/` — Infraestrutura e Serviços Externos

`src/modules/` **nunca configura** conexões de banco ou serviços externos — apenas **consome** o que `infra/` expõe.

---

## 5. Anatomia Completa de um Módulo

```text
src/modules/auth/
├── controllers/
│   └── auth.controller.ts          # Recebe HTTP, delega ao use case
├── use-cases/
│   ├── login/
│   │   └── login.use-case.ts       # Valida credenciais, gera token
│   ├── register/
│   │   └── register.use-case.ts    # Cria usuário, envia e-mail, retorna token
│   └── refresh-token/
│       └── refresh-token.use-case.ts
├── repositories/
│   └── auth.repository.ts          # Encapsula queries de User (TypeORM/Prisma)
├── entities/
│   └── user.entity.ts              # @Entity TypeORM / Prisma model
├── dtos/
│   ├── login.dto.ts                # @IsEmail(), @IsString(), etc.
│   └── register.dto.ts
├── types/
│   └── auth.types.ts               # Interfaces e enums locais do domínio
├── __tests__/
│   ├── login.use-case.spec.ts      # Testa o use case (mock do repository)
│   ├── register.use-case.spec.ts
│   └── auth.controller.spec.ts     # Testa o controller (mock dos use cases)
└── auth.module.ts                  # Declara e exporta providers do domínio
```

### Exemplo de `auth.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AuthController } from './controllers/auth.controller';
import { LoginUseCase } from './use-cases/login/login.use-case';
import { RegisterUseCase } from './use-cases/register/register.use-case';
import { RefreshTokenUseCase } from './use-cases/refresh-token/refresh-token.use-case';
import { AuthRepository } from './repositories/auth.repository';
import { UserEntity } from './entities/user.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([UserEntity]),
  ],
  controllers: [AuthController],
  providers: [
    // Use cases — cada operação de negócio explícita
    LoginUseCase,
    RegisterUseCase,
    RefreshTokenUseCase,
    // Repository — acesso ao banco
    AuthRepository,
  ],
  exports: [LoginUseCase], // expõe apenas o necessário para outros módulos
})
export class AuthModule {}
```

---

## 6. Isolamento entre Módulos

### Módulos comunicam-se via `exports` do `*.module.ts`

```typescript
// ❌ ERRADO — meals importa um use case diretamente do arquivo
import { GetUserUseCase } from '@modules/users/use-cases/get-user/get-user.use-case';

// ✅ CORRETO — meals importa o módulo que expõe o use case
// Em users.module.ts:
@Module({
  providers: [GetUserUseCase, UsersRepository],
  exports: [GetUserUseCase], // ← declara contrato público
})
export class UsersModule {}

// Em meals.module.ts:
@Module({
  imports: [UsersModule], // ← importa o módulo inteiro
  providers: [CreateMealUseCase, MealsRepository],
})
export class MealsModule {}
```

---

## 7. Convenção de Nomenclatura

| Tipo | Convenção | Exemplo |
|---|---|---|
| Arquivos | kebab-case com sufixo do tipo | `create-meal.use-case.ts`, `login.dto.ts` |
| Classes | PascalCase com sufixo do tipo | `CreateMealUseCase`, `LoginDto` |
| Módulos NestJS | PascalCase + `Module` | `AuthModule`, `MealsModule` |
| Controllers | PascalCase + `Controller` | `AuthController` |
| Use Cases | PascalCase + `UseCase` | `CreateMealUseCase`, `LoginUseCase` |
| Repositories | PascalCase + `Repository` | `MealsRepository` |
| Entities | PascalCase + `Entity` | `UserEntity`, `MealEntity` |
| DTOs | PascalCase + `Dto` | `CreateMealDto`, `LoginDto` |
| Guards | PascalCase + `Guard` | `JwtAuthGuard` |
| Filters | PascalCase + `Filter` | `HttpExceptionFilter` |
| Interceptors | PascalCase + `Interceptor` | `LoggingInterceptor` |
| Constantes globais | SCREAMING_SNAKE_CASE | `BLOCK_VALUES`, `JWT_EXPIRATION` |
| Arquivos de teste | mesmo nome + `.spec.ts` | `create-meal.use-case.spec.ts` |

---

## 8. Teste de Use Cases

Use cases são a parte mais importante a testar. Como cada use case tem uma única responsabilidade, os testes são diretos.

```typescript
// src/modules/meals/__tests__/create-meal.use-case.spec.ts

import { Test } from '@nestjs/testing';
import { CreateMealUseCase } from '../use-cases/create-meal/create-meal.use-case';
import { MealsRepository } from '../repositories/meals.repository';

describe('CreateMealUseCase', () => {
  let useCase: CreateMealUseCase;
  let mealsRepository: jest.Mocked<MealsRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        CreateMealUseCase,
        {
          provide: MealsRepository,
          useValue: {
            create: jest.fn(),
          },
        },
      ],
    }).compile();

    useCase = module.get(CreateMealUseCase);
    mealsRepository = module.get(MealsRepository);
  });

  it('deve criar uma refeição com os dados corretos', async () => {
    const input = { userId: 'user-1', blocks: 4, mode: 'quick' as const };
    const mockMeal = { id: 'meal-1', ...input };
    mealsRepository.create.mockResolvedValue(mockMeal as any);

    const result = await useCase.execute(input);

    expect(mealsRepository.create).toHaveBeenCalledWith(
      expect.objectContaining({ userId: 'user-1', blocks: 4 }),
    );
    expect(result.id).toBe('meal-1');
  });
});
```

---

## 9. Fluxo de Implementação de um Novo Módulo

Ao criar um novo módulo (ex: "histórico de refeições"):

**1. Criar a pasta do domínio**
`src/modules/meal-history/`

**2. Criar a entidade**
`entities/meal-history.entity.ts`

**3. Criar os DTOs**
`dtos/create-meal-history.dto.ts`

**4. Criar o Repository**
`repositories/meal-history.repository.ts`

**5. Criar os Use Cases (um por operação)**
```
use-cases/
├── save-meal-history/
│   └── save-meal-history.use-case.ts
├── get-meal-history/
│   └── get-meal-history.use-case.ts
└── delete-meal-history/
    └── delete-meal-history.use-case.ts
```

**6. Criar o Controller**
`controllers/meal-history.controller.ts`

**7. Criar os testes**
`__tests__/save-meal-history.use-case.spec.ts`

**8. Montar o módulo**
`meal-history.module.ts` — declarar todos os use cases em `providers`

**9. Registrar no AppModule**
Adicionar `MealHistoryModule` ao `app.module.ts`

---

## 10. `app.module.ts` — Módulo Raiz

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { DatabaseModule } from '@infra/database/database.module';
import { AuthModule } from '@modules/auth/auth.module';
import { MealsModule } from '@modules/meals/meals.module';
import { UsersModule } from '@modules/users/users.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    DatabaseModule,             // infra sempre antes dos módulos de domínio
    AuthModule,
    UsersModule,
    MealsModule,
  ],
})
export class AppModule {}
```

---

## 11. Checklist de Validação (Autoavaliação)

Antes de entregar qualquer código, verifique cada item:

**Use Cases**
- [ ] Cada operação de negócio tem seu próprio use case (um arquivo, uma classe, um `execute()`)?
- [ ] O use case não tem outros métodos públicos além de `execute()`?
- [ ] O use case injeta apenas Repository, infraestrutura (MailerService, etc.) ou outros use cases?
- [ ] Use cases estão em `use-cases/<operação>/<operação>.use-case.ts`?
- [ ] Todos os use cases estão declarados em `providers` no `*.module.ts`?

**Estrutura e Isolamento**
- [ ] O módulo está dentro de `src/modules/<domínio>/`?
- [ ] Nenhum módulo importa arquivos internos de outro módulo (apenas via `imports: [OutroModule]`)?
- [ ] O `*.module.ts` está declarado na raiz do domínio?

**Camadas**
- [ ] Controllers são finos? Nenhuma lógica de negócio neles?
- [ ] Repositories encapsulam todo o acesso ao ORM/banco?
- [ ] Use cases nunca acessam TypeORM/Prisma diretamente?

**DTOs e Validação**
- [ ] Todo payload de entrada usa um DTO com `class-validator`?
- [ ] `ValidationPipe` está habilitado globalmente no `main.ts`?

**Configuração e Path Aliases**
- [ ] Nenhum arquivo fora de `src/config/` acessa `process.env` diretamente?
- [ ] Todos os imports usam path aliases (`@modules/`, `@common/`, `@config/`, `@infra/`)?

**Nomenclatura**
- [ ] Arquivos em kebab-case com sufixo do tipo?
- [ ] Classes em PascalCase com sufixo do tipo?

**Testes**
- [ ] Existe `__tests__/` dentro do módulo?
- [ ] Cada use case tem um `.spec.ts` co-localizado?
- [ ] Use cases testados com mock do Repository?
- [ ] Controller testado com mock dos use cases?
