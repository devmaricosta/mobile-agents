---
name: clean-architecture
description: Separação em camadas dentro de cada módulo NestJS seguindo Clean Architecture — Controller → UseCase → Repository → Entity. Regras de dependência (camadas internas não conhecem as externas), onde ficam DTOs vs Entities vs ViewModels, e como o NestJS mapeia para essa estrutura.
---

# Clean Architecture no NestJS — Separação de Camadas

Você é um Arquiteto de Software Senior. Esta skill complementa `nest-project-structure.md` (estrutura de pastas e use cases) e `nest-modules-pattern.md` (organização de módulos NestJS). Aqui o foco é **como as camadas se relacionam dentro de um módulo** e **quais tipos de objetos pertencem a cada camada**.

> **Regra fundamental:** As dependências fluem sempre para dentro — camadas internas não conhecem as externas.

---

## 1. As Quatro Camadas (da mais externa para a mais interna)

```
┌─────────────────────────────────────────────────────────────┐
│  PRESENTATION (Controllers, Guards, Interceptors)           │  ← NestJS
├─────────────────────────────────────────────────────────────┤
│  APPLICATION (Use Cases)                                    │  ← Orquestração
├─────────────────────────────────────────────────────────────┤
│  DOMAIN (Entities, Tipos, Interfaces de Repository)         │  ← Regras de negócio
├─────────────────────────────────────────────────────────────┤
│  INFRASTRUCTURE (Repositories impl., ORM, Serviços externos)│  ← Detalhes técnicos
└─────────────────────────────────────────────────────────────┘
```

### Fluxo de uma requisição HTTP

```
HTTP Request
    ↓
Controller          ← valida entrada com DTO, extrai dados da request
    ↓
UseCase.execute()   ← orquestra regras de negócio
    ↓
Repository          ← acessa banco de dados via ORM
    ↓
Entity              ← objeto persistido/retornado
    ↑
Controller          ← mapeia Entity/resultado para ViewModel/DTO de resposta
    ↑
HTTP Response
```

---

## 2. Regras de Dependência (The Dependency Rule)

A regra central da Clean Architecture: **o código de uma camada só pode depender de camadas mais internas.**

| Camada | Pode importar de | Nunca importa de |
|--------|------------------|------------------|
| **Controller** (Presentation) | UseCase, DTO de entrada, ViewModel | Repository, Entity diretamente, ORM |
| **UseCase** (Application) | Repository (interface), Entity, tipos do domínio | Controller, NestJS decorators, ORM, HTTP |
| **Repository impl.** (Infrastructure) | Entity, ORM (TypeORM/Prisma) | UseCase, Controller |
| **Entity** (Domain) | Nada externo | Absolutamente nada do projeto |

### Visualizando as dependências dentro de um módulo

```
src/modules/meals/
├── controllers/          ← PRESENTATION
│   └── meals.controller.ts
│       imports: MealsUseCase, CreateMealDto, MealViewModel
│
├── use-cases/            ← APPLICATION
│   └── create-meal/
│       └── create-meal.use-case.ts
│           imports: MealsRepository, MealEntity
│
├── repositories/         ← INFRASTRUCTURE
│   └── meals.repository.ts
│       imports: MealEntity, TypeORM Repository<MealEntity>
│
├── entities/             ← DOMAIN
│   └── meal.entity.ts
│       imports: TypeORM decorators apenas (aceitável neste projeto)
│
├── dtos/                 ← PRESENTATION boundary (entrada)
│   └── create-meal.dto.ts
│
└── view-models/          ← PRESENTATION boundary (saída)
    └── meal.view-model.ts
```

> **Nota sobre Entities e TypeORM:** Em projetos que usam TypeORM, é aceitável que a `@Entity` contenha decorators do TypeORM (`@Column`, `@PrimaryGeneratedColumn`). Isso é um pragmatismo acordado para evitar duplicação de modelos na nossa stack. Para levar ao extremo, existiria um "persistence model" separado na camada de infrastructure.

---

## 3. DTOs vs Entities vs ViewModels — Onde Cada Um Vive

### Tabela de decisão rápida

| Objeto | Onde fica | Responsabilidade | Contém validação? | Contém lógica? |
|--------|-----------|------------------|-------------------|----------------|
| **DTO de entrada** | `dtos/` | Define o contrato da requisição HTTP | ✅ `class-validator` | ❌ |
| **Entity** | `entities/` | Representa o modelo de dados persistido | ❌ | Mínima (getters) |
| **ViewModel** | `view-models/` | Define o contrato da resposta HTTP (shape para o cliente) | ❌ | ❌ (mapeamento puro) |
| **Tipo de domínio** | `types/` | Interfaces, enums e tipos internos ao módulo | ❌ | ❌ |

---

### 3.1 DTO de Entrada (Input DTO)

**Onde:** `src/modules/<domínio>/dtos/`

**Propósito:** Define e valida o payload de entrada de uma requisição HTTP. Nunca sai do Controller — o UseCase recebe os dados já validados.

```typescript
// src/modules/meals/dtos/create-meal.dto.ts
import { IsNumber, IsEnum, IsOptional, Min, Max } from 'class-validator';

export enum MealMode {
  QUICK = 'quick',
  CUSTOM = 'custom',
}

export class CreateMealDto {
  @IsNumber()
  @Min(1)
  @Max(10)
  blocks: number;

  @IsEnum(MealMode)
  mode: MealMode;

  @IsOptional()
  @IsNumber()
  targetCalories?: number;
}
```

**Regras:**
- Use `class-validator` e `class-transformer` para validação automática
- DTOs são "anemic" — sem métodos de negócio
- Nomes: `CreateMealDto`, `UpdateMealDto`, `FilterMealsDto`
- Nunca passe um DTO diretamente para o Repository — mapeie para Entity ou tipo

---

### 3.2 Entity

**Onde:** `src/modules/<domínio>/entities/`

**Propósito:** Representa o modelo de dados tal como é armazenado no banco. É o objeto que o Repository retorna para o UseCase.

```typescript
// src/modules/meals/entities/meal.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  ManyToOne,
  JoinColumn,
} from 'typeorm';

@Entity('meals')
export class MealEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  userId: string;

  @Column('int')
  blocks: number;

  @Column({ type: 'varchar', length: 20 })
  mode: string;

  @Column({ nullable: true })
  targetCalories?: number;

  @CreateDateColumn()
  createdAt: Date;
}
```

**Regras:**
- A Entity representa o schema do banco — não o shape da API
- Não adicione métodos de negócio complexos na Entity (pertencem ao UseCase)
- Nunca retorne uma Entity diretamente para o Controller sem mapear
- Nomes: `MealEntity`, `UserEntity`, `NotificationEntity`

---

### 3.3 ViewModel (Output DTO / Response DTO)

**Onde:** `src/modules/<domínio>/view-models/`

**Propósito:** Define exatamente o que o cliente (app mobile, frontend) vai receber. Pode agregrar dados de múltiplas entidades, formatar campos, e omitir campos sensíveis.

```typescript
// src/modules/meals/view-models/meal.view-model.ts
export class MealViewModel {
  id: string;
  blocks: number;
  mode: string;
  targetCalories: number | null;
  createdAt: string; // ISO string — não Date object

  static fromEntity(entity: MealEntity): MealViewModel {
    const vm = new MealViewModel();
    vm.id = entity.id;
    vm.blocks = entity.blocks;
    vm.mode = entity.mode;
    vm.targetCalories = entity.targetCalories ?? null;
    vm.createdAt = entity.createdAt.toISOString();
    return vm;
  }

  static fromEntities(entities: MealEntity[]): MealViewModel[] {
    return entities.map(MealViewModel.fromEntity);
  }
}
```

**Quando usar ViewModel vs retornar a Entity diretamente:**

| Situação | Use ViewModel? |
|----------|----------------|
| A Entity tem campos sensíveis (senha, token interno) | ✅ Obrigatório |
| A resposta agrega dados de múltiplas entidades | ✅ Obrigatório |
| A resposta omite campos ou os formata diferente | ✅ Obrigatório |
| A Entity tem exatamente o mesmo shape da resposta desejada | ⚠️ Ainda prefira — protege contra exposição acidental |

> **Regra prática neste projeto:** Sempre use ViewModel para respostas de Controller. A Entity é um detalhe de persistência, não um contrato de API.

---

### 3.4 Tipos de Domínio (Types)

**Onde:** `src/modules/<domínio>/types/`

**Propósito:** Interfaces e enums que são internos ao módulo — não são DTOs (não têm validação) e não são persistidos (não são Entities).

```typescript
// src/modules/meals/types/meal.types.ts

// Input tipado para o UseCase (pode estender o DTO)
export type CreateMealInput = {
  userId: string;  // vem do JWT, não do body
  blocks: number;
  mode: 'quick' | 'custom';
  targetCalories?: number;
};

// Resultado intermediário de uso interno
export type MealWithSummary = MealEntity & {
  summary: string;
};
```

---

## 4. Como o Controller Usa Cada Tipo

O Controller é o único ponto onde os três tipos se encontram — ele recebe DTOs de entrada, chama use cases, e mapeia a saída para ViewModels.

```typescript
// src/modules/meals/controllers/meals.controller.ts
import { Body, Controller, Get, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '@common/guards/jwt-auth.guard';
import { CurrentUser } from '@common/decorators/current-user.decorator';
import { UserPayload } from '@common/types/user-payload.type';

import { CreateMealDto } from '../dtos/create-meal.dto';
import { MealViewModel } from '../view-models/meal.view-model';
import { CreateMealUseCase } from '../use-cases/create-meal/create-meal.use-case';
import { GetMealHistoryUseCase } from '../use-cases/get-meal-history/get-meal-history.use-case';

@Controller('meals')
@UseGuards(JwtAuthGuard)
export class MealsController {
  constructor(
    private readonly createMealUseCase: CreateMealUseCase,
    private readonly getMealHistoryUseCase: GetMealHistoryUseCase,
  ) {}

  @Post()
  async create(
    @Body() dto: CreateMealDto,    // ← DTO valida o body
    @CurrentUser() user: UserPayload,
  ): Promise<MealViewModel> {      // ← retorna ViewModel
    const meal = await this.createMealUseCase.execute({
      ...dto,
      userId: user.id,             // ← enriquece com dados do JWT
    });
    return MealViewModel.fromEntity(meal); // ← mapeia Entity → ViewModel
  }

  @Get('history')
  async getHistory(
    @CurrentUser() user: UserPayload,
  ): Promise<MealViewModel[]> {
    const meals = await this.getMealHistoryUseCase.execute({ userId: user.id });
    return MealViewModel.fromEntities(meals);
  }
}
```

---

## 5. Como o UseCase Usa Cada Tipo

O UseCase nunca sabe que existe HTTP, Controllers ou ViewModels. Ele opera apenas com tipos de entrada simples e retorna Entities ou tipos de domínio.

```typescript
// src/modules/meals/use-cases/create-meal/create-meal.use-case.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { MealsRepository } from '@modules/meals/repositories/meals.repository';
import { MealEntity } from '@modules/meals/entities/meal.entity';
import { CreateMealInput } from '@modules/meals/types/meal.types';

@Injectable()
export class CreateMealUseCase {
  constructor(private readonly mealsRepository: MealsRepository) {}

  // ← recebe tipo de domínio, não DTO
  // ← retorna Entity, não ViewModel
  async execute(input: CreateMealInput): Promise<MealEntity> {
    // Regras de negócio ficam AQUI
    if (input.blocks <= 0) {
      throw new BadRequestException('Blocos devem ser positivos');
    }

    return this.mealsRepository.create({
      userId: input.userId,
      blocks: input.blocks,
      mode: input.mode,
      targetCalories: input.targetCalories,
    });
  }
}
```

---

## 6. Como o Repository Usa Cada Tipo

O Repository só conhece a Entity e o ORM. Nunca conhece DTOs, ViewModels ou Use Cases.

```typescript
// src/modules/meals/repositories/meals.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { MealEntity } from '../entities/meal.entity';

@Injectable()
export class MealsRepository {
  constructor(
    @InjectRepository(MealEntity)
    private readonly repo: Repository<MealEntity>,
  ) {}

  // ← recebe Partial<Entity> ou campos simples
  // ← retorna sempre Entity
  async create(data: Partial<MealEntity>): Promise<MealEntity> {
    return this.repo.save(this.repo.create(data));
  }

  async findByUserId(userId: string): Promise<MealEntity[]> {
    return this.repo.find({
      where: { userId },
      order: { createdAt: 'DESC' },
    });
  }

  async findById(id: string): Promise<MealEntity | null> {
    return this.repo.findOne({ where: { id } });
  }
}
```

---

## 7. Regras de Importação por Camada

### O que cada camada pode e não pode importar

```typescript
// ✅ CORRETO — Controller importa apenas DTOs, ViewModels e UseCases
import { CreateMealDto } from '../dtos/create-meal.dto';
import { MealViewModel } from '../view-models/meal.view-model';
import { CreateMealUseCase } from '../use-cases/create-meal/create-meal.use-case';

// ❌ ERRADO — Controller nunca importa Repository ou ORM
import { MealsRepository } from '../repositories/meals.repository';       // ← PROIBIDO
import { InjectRepository } from '@nestjs/typeorm';                       // ← PROIBIDO no Controller

// ✅ CORRETO — UseCase importa Repository e Entity
import { MealsRepository } from '@modules/meals/repositories/meals.repository';
import { MealEntity } from '@modules/meals/entities/meal.entity';

// ❌ ERRADO — UseCase nunca importa ViewModel ou decorators HTTP
import { MealViewModel } from '../view-models/meal.view-model';           // ← PROIBIDO
import { Controller, Get } from '@nestjs/common';                         // ← PROIBIDO no UseCase

// ✅ CORRETO — Repository importa Entity e ORM
import { MealEntity } from '../entities/meal.entity';
import { Repository } from 'typeorm';

// ❌ ERRADO — Entity não importa nada do projeto
import { MealsRepository } from '../repositories/meals.repository';       // ← PROIBIDO na Entity
```

---

## 8. Mapeamento para Estrutura NestJS

Como a Clean Architecture se traduz nos conceitos nativos do NestJS:

| Clean Architecture | NestJS / Este Projeto |
|--------------------|-----------------------|
| Interface Adapters (Presentation) | `controllers/`, Guards, Interceptors, Pipes |
| Application Layer (Use Cases) | `use-cases/<operação>/<operação>.use-case.ts` |
| Domain Layer (Entities) | `entities/<nome>.entity.ts`, `types/<domínio>.types.ts` |
| Infrastructure Layer | `repositories/`, `src/infra/` (DatabaseModule, MailerModule) |
| DTOs (entrada) | `dtos/<nome>.dto.ts` → validados com `class-validator` |
| ViewModels (saída) | `view-models/<nome>.view-model.ts` → mapeados no Controller |

### NestJS `@Module()` como fronteira de camada

O `@Module()` do NestJS reforça o encapsulamento entre módulos — é a única fronteira explícita entre domínios. As camadas **dentro** de um módulo são reforçadas por convenções de código (esta skill) e não por mecanismos do framework.

```typescript
// auth.module.ts — declara todos os providers de todas as camadas do domínio auth
@Module({
  imports: [TypeOrmModule.forFeature([UserEntity])],
  controllers: [AuthController],               // ← PRESENTATION
  providers: [
    LoginUseCase,                              // ← APPLICATION
    RegisterUseCase,                           // ← APPLICATION
    AuthRepository,                            // ← INFRASTRUCTURE
    // Entities são registradas via TypeOrmModule, não como providers
  ],
  exports: [LoginUseCase],                     // ← exporta apenas o contrato público
})
export class AuthModule {}
```

---

## 9. Anti-Padrões Comuns

### ❌ Fat Controller (lógica de negócio no Controller)

```typescript
// ❌ ERRADO
@Post()
async create(@Body() dto: CreateMealDto, @CurrentUser() user: UserPayload) {
  // Regra de negócio no controller!
  if (dto.blocks <= 0) throw new BadRequestException('Blocos inválidos');
  const meal = await this.mealsRepository.create({ ...dto, userId: user.id }); // acessa repository diretamente!
  return { ...meal, formattedDate: meal.createdAt.toLocaleDateString('pt-BR') }; // manipula no controller!
}
```

### ❌ UseCase conhecendo HTTP / NestJS framework

```typescript
// ❌ ERRADO
@Injectable()
export class CreateMealUseCase {
  // UseCase NUNCA deve dependender de req/res ou tipos HTTP
  async execute(@Body() dto: CreateMealDto): Promise<MealViewModel> { // ← decorators HTTP proibidos
    const vm = new MealViewModel();                                     // ← ViewModel no use case: PROIBIDO
    return vm;
  }
}
```

### ❌ Entity retornada diretamente pela API

```typescript
// ❌ ERRADO — Entity exposta diretamente
@Get(':id')
async findOne(@Param('id') id: string): Promise<MealEntity> { // ← Entity como retorno HTTP: RISCO
  return this.getMealUseCase.execute({ id });
  // Expõe todos os campos do banco, inclusive internos ou sensíveis
}

// ✅ CORRETO
@Get(':id')
async findOne(@Param('id') id: string): Promise<MealViewModel> {
  const meal = await this.getMealUseCase.execute({ id });
  return MealViewModel.fromEntity(meal);
}
```

### ❌ Repository com lógica de negócio

```typescript
// ❌ ERRADO
export class MealsRepository {
  async createWithValidation(data: Partial<MealEntity>) {
    if (data.blocks <= 0) throw new Error('Blocos inválidos'); // regra de negócio: pertence ao UseCase
    const totalCalories = data.blocks * 500;   // cálculo de domínio: pertence ao UseCase
    return this.repo.save({ ...data, totalCalories });
  }
}
```

---

## 10. Checklist de Validação por Camada

### Controller
- [ ] Recebe DTO de entrada (validado por `class-validator`)?
- [ ] **Não** contém nenhuma lógica de negócio?
- [ ] **Não** acessa Repository diretamente?
- [ ] Mapeia o retorno do UseCase para um ViewModel antes de retornar?
- [ ] Injeta apenas Use Cases (e decorators/guards do `@common/`)?

### UseCase
- [ ] Tem apenas um método público: `execute()`?
- [ ] Recebe um tipo de entrada simples (não um DTO do Controller)?
- [ ] Retorna uma Entity ou tipo de domínio (não um ViewModel)?
- [ ] **Não** contém `@Body()`, `@Param()`, `Res`, `Req` ou imports de `@nestjs/common` além de `@Injectable()`?
- [ ] **Não** acessa TypeORM/Prisma diretamente?

### Repository
- [ ] Apenas operações de leitura/escrita no banco?
- [ ] **Não** contém regras de negócio ou validações de domínio?
- [ ] Trabalha apenas com `Entity` e métodos do ORM?
- [ ] Retorna sempre `Entity` (nunca DTO ou ViewModel)?

### Entity
- [ ] Sem imports de outras camadas do projeto?
- [ ] Contém apenas decorators do ORM (TypeORM/Prisma) e campos?
- [ ] Sem métodos com lógica de negócio complexa?

### DTO
- [ ] Contém decorators de validação (`class-validator`)?
- [ ] Não contém lógica de negócio?
- [ ] Representa claramente o contrato do endpoint?

### ViewModel
- [ ] Tem método estático `fromEntity()` (ou equivalente para mapeamento)?
- [ ] Nunca é modificado dentro de um UseCase?
- [ ] Não expõe campos sensíveis da Entity/banco?

---

## Referências

- [Clean Architecture — Robert C. Martin (Uncle Bob)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [NestJS Docs — Providers](https://docs.nestjs.com/providers)
- [NestJS Docs — Controllers](https://docs.nestjs.com/controllers)
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura de pastas e Use Cases Pattern
- [nest-modules-pattern.md](.agent/skills/backend/foundation-and-architecture/nest-modules-pattern.md) — Módulos NestJS, exports e dependências circulares
