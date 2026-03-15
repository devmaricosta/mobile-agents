---
name: rbac-authorization
description: Autorização com RBAC (Role-Based Access Control) no NestJS — @Roles() decorator com SetMetadata, RolesGuard com Reflector, integração com o JwtAuthGuard existente, hierarquia de roles com AccessControlService, decorator @Public() para rotas abertas, combinação RBAC + ABAC com CASL para regras de ownership, e como escalar para policies granulares. Complementa clean-architecture.md (Guards na camada de Presentation), nest-project-structure.md (localização dos arquivos em src/common/ e src/modules/auth/), rest-api-patterns.md (status 403 Forbidden), e graphql-pattern.md (GqlAuthGuard).
---

# Autorização RBAC — NestJS

Você é um Arquiteto de Software Senior. Esta skill define **como implementar autorização baseada em roles (RBAC)** no projeto NestJS, integrando com o sistema de autenticação JWT já existente. Ela complementa:

- `clean-architecture.md` — Guards ficam na camada de Presentation; UseCase não deve verificar roles diretamente
- `nest-project-structure.md` — Localização exata de decorators, guards e enums em `src/common/` e `src/modules/auth/`
- `rest-api-patterns.md` — Erros 401 (não autenticado) vs 403 (sem permissão)
- `graphql-pattern.md` — Adaptação para o `GqlAuthGuard`

> **Regra fundamental:** Autenticação (quem é o usuário) e autorização (o que ele pode fazer) são responsabilidades separadas. O `JwtAuthGuard` cuida da autenticação; o `RolesGuard` cuida da autorização. Os dois sempre operam em sequência — autenticação primeiro.

---

## 1. Visão Geral da Estratégia

```
HTTP Request
    ↓
JwtAuthGuard      ← Verifica o token JWT (autenticação)
    ↓               └── Popula request.user com { id, email, roles }
RolesGuard        ← Verifica se o usuário tem o role exigido (autorização)
    ↓
Controller.execute() → UseCase.execute() → ...
```

### Separação de responsabilidades

| Camada | Responsabilidade |
|---|---|
| **`JwtAuthGuard`** | Valida o JWT, popula `request.user` |
| **`RolesGuard`** | Compara `request.user.roles` com os roles exigidos pelo endpoint |
| **`@Roles()`** | Decorator declarativo que marca quais roles são necessários |
| **`@Public()`** | Decorator que bypassa os guards em rotas abertas |
| **UseCase** | Nunca verifica roles — delega ao Guard |

---

## 2. Estrutura de Arquivos

Consistente com `nest-project-structure.md`:

```
src/
├── common/
│   ├── decorators/
│   │   ├── roles.decorator.ts         ← @Roles(...roles)
│   │   ├── public.decorator.ts        ← @Public()
│   │   └── current-user.decorator.ts  ← @CurrentUser() (já existente)
│   ├── guards/
│   │   ├── jwt-auth.guard.ts          ← JwtAuthGuard (já existente)
│   │   ├── roles.guard.ts             ← RolesGuard (novo)
│   │   └── gql-auth.guard.ts          ← GqlAuthGuard (já existente)
│   └── types/
│       └── user-payload.type.ts       ← UserPayload com campo roles
│
└── modules/
    └── auth/
        ├── enums/
        │   └── role.enum.ts           ← enum Role
        └── services/
            └── access-control.service.ts ← Hierarquia de roles (opcional)
```

> **Regra:** Decorators genéricos reutilizáveis ficam em `src/common/decorators/`. O enum `Role` fica no módulo de `auth` porque é um detalhe do domínio de identidade.

---

## 3. Enum de Roles

Define os roles disponíveis no sistema. Mantenha o enum simples e adicione novos roles conforme o domínio evolui.

```typescript
// src/modules/auth/enums/role.enum.ts

/**
 * Roles disponíveis no sistema.
 *
 * Ordem de privilégio (mais alto → mais baixo):
 * ADMIN > NUTRITIONIST > USER
 *
 * Use strings legíveis (não números) para facilitar debug
 * de tokens JWT e logs de auditoria.
 */
export enum Role {
  ADMIN        = 'admin',
  NUTRITIONIST = 'nutritionist', // Ex. CAB 2.0: nutricionista vinculado ao usuário
  USER         = 'user',
}
```

> **Regra:** Strings descritivas (não inteiros) facilitam a leitura no payload JWT e em logs. Evite roles genéricos como `ROLE_1` — use nomes que refletem o domínio do negócio.

---

## 4. Decorator `@Roles()`

```typescript
// src/common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { Role } from '@modules/auth/enums/role.enum';

/**
 * Chave de metadado para roles — usada pelo RolesGuard via Reflector.
 */
export const ROLES_KEY = 'roles';

/**
 * Marca um endpoint (ou controller inteiro) com os roles permitidos.
 *
 * @example
 * // Apenas admins
 * @Roles(Role.ADMIN)
 * @Delete(':id')
 * remove(@Param('id') id: string) { ... }
 *
 * @example
 * // Admins OU nutricionistas
 * @Roles(Role.ADMIN, Role.NUTRITIONIST)
 * @Get('all')
 * findAll() { ... }
 */
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

---

## 5. Decorator `@Public()`

Rotas públicas (ex: login, registro, health check) precisam bypassa **ambos** os guards.

```typescript
// src/common/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

/**
 * Marca uma rota como pública — bypassa JwtAuthGuard e RolesGuard.
 *
 * @example
 * @Public()
 * @Post('login')
 * login(@Body() dto: LoginDto) { ... }
 */
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

---

## 6. Atualização do `UserPayload`

O `request.user` populado pelo `JwtAuthGuard` precisa incluir o campo `roles`:

```typescript
// src/common/types/user-payload.type.ts
import { Role } from '@modules/auth/enums/role.enum';

/**
 * Shape do usuário autenticado disponível em request.user.
 * Populado pelo JwtAuthGuard após validação do JWT.
 */
export type UserPayload = {
  id: string;
  email: string;
  roles: Role[];        // ← obrigatório para o RolesGuard funcionar
};
```

### Atualização na JwtStrategy

O payload do JWT deve incluir `roles`:

```typescript
// src/modules/auth/strategies/jwt.strategy.ts (trecho)
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  async validate(payload: { sub: string; email: string; roles: Role[] }): Promise<UserPayload> {
    // O retorno deste método é atribuído a request.user pelo Passport
    return {
      id: payload.sub,
      email: payload.email,
      roles: payload.roles,   // ← inclui roles no UserPayload
    };
  }
}
```

### Geração do JWT com roles

```typescript
// src/modules/auth/use-cases/login/login.use-case.ts (trecho)
const payload = {
  sub: user.id,
  email: user.email,
  roles: user.roles,   // ← inclui roles no token
};
const accessToken = this.jwtService.sign(payload);
```

> **Atenção:** Nunca inclua dados sensíveis (senha, documentos) no payload do JWT. `roles` é um dado de autorização legítimo para incluir no token.

---

## 7. `JwtAuthGuard` — Integração com `@Public()`

O guard de autenticação deve respeitar o decorator `@Public()`:

```typescript
// src/common/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AuthGuard } from '@nestjs/passport';
import { IS_PUBLIC_KEY } from '@common/decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // Verifica se a rota está marcada como pública antes de validar o JWT
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true; // ← bypassa autenticação em rotas públicas
    }

    return super.canActivate(context); // ← valida o JWT normalmente
  }
}
```

---

## 8. `RolesGuard` — Implementação Principal

```typescript
// src/common/guards/roles.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role } from '@modules/auth/enums/role.enum';
import { ROLES_KEY } from '@common/decorators/roles.decorator';
import { IS_PUBLIC_KEY } from '@common/decorators/public.decorator';
import type { UserPayload } from '@common/types/user-payload.type';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // 1. Rota pública → bypassa autorização
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    // 2. Lê os roles exigidos pelo endpoint (método) ou controller (classe)
    //    getAllAndOverride: método tem prioridade sobre a classe
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // 3. Sem @Roles() declarado → acesso liberado para qualquer autenticado
    if (!requiredRoles || requiredRoles.length === 0) {
      return true;
    }

    // 4. Extrai o usuário autenticado (populado pelo JwtAuthGuard)
    const request = context.switchToHttp().getRequest<{ user: UserPayload }>();
    const user = request.user;

    if (!user) {
      // Não deve ocorrer se JwtAuthGuard rodar antes, mas é defensivo
      throw new ForbiddenException('Usuário não autenticado.');
    }

    // 5. Verifica se o usuário tem PELO MENOS UM dos roles exigidos
    const hasRole = requiredRoles.some((role) => user.roles?.includes(role));

    if (!hasRole) {
      throw new ForbiddenException(
        `Acesso negado. Role(s) necessário(s): ${requiredRoles.join(', ')}.`,
      );
    }

    return true;
  }
}
```

> **Por que `ForbiddenException` em vez de `return false`?**  
> Retornar `false` faz o NestJS retornar `{ "statusCode": 403, "message": "Forbidden resource" }` — mensagem genérica. Lançar `ForbiddenException` com mensagem customizada permite que o `HttpExceptionFilter` (definido em `rest-api-patterns.md`) formate o erro com o `code: 'FORBIDDEN'` e `message` explicativa. Isso é consistente com o padrão de erros do projeto.

---

## 9. Registro Global dos Guards

Registre **ambos os guards** como globais no `AppModule`. A ordem importa: autenticação sempre antes de autorização.

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { JwtAuthGuard } from '@common/guards/jwt-auth.guard';
import { RolesGuard } from '@common/guards/roles.guard';

@Module({
  // ...imports, controllers, providers
  providers: [
    // ← ORDEM OBRIGATÓRIA: JwtAuthGuard antes do RolesGuard
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
    },
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> **Regra:** Com registro global via `APP_GUARD`, todos os endpoints passam pelos guards automaticamente. Use `@Public()` para abrir exceções. Isso é o padrão **secure-by-default** — nunca o contrário.

---

## 10. Uso nos Controllers

### 10.1 Decorator no nível do método

```typescript
// src/modules/meals/controllers/meals.controller.ts
import { Controller, Get, Post, Body, Param, Delete, HttpCode, HttpStatus } from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation } from '@nestjs/swagger';
import { Roles } from '@common/decorators/roles.decorator';
import { CurrentUser } from '@common/decorators/current-user.decorator';
import { Role } from '@modules/auth/enums/role.enum';
import type { UserPayload } from '@common/types/user-payload.type';

@ApiTags('meals')
@ApiBearerAuth('access-token')
@Controller({ path: 'meals', version: '1' })
export class MealsController {

  // ← Acessível para qualquer usuário autenticado (sem @Roles → todos os roles passam)
  @Get()
  @ApiOperation({ summary: 'Listar refeições do usuário autenticado' })
  async findAll(@CurrentUser() user: UserPayload) {
    return this.getMealHistoryUseCase.execute({ userId: user.id });
  }

  // ← Apenas admins podem ver TODAS as refeições do sistema
  @Get('admin/all')
  @Roles(Role.ADMIN)
  @ApiOperation({ summary: '[Admin] Listar todas as refeições do sistema' })
  async findAllAdmin() {
    return this.getMealHistoryUseCase.execute({});
  }

  // ← Admins OU Nutricionistas podem configurar a dieta
  @Post(':userId/diet-config')
  @Roles(Role.ADMIN, Role.NUTRITIONIST)
  @ApiOperation({ summary: '[Admin/Nutritionist] Configurar dieta de um usuário' })
  async configureDiet(
    @Param('userId') userId: string,
    @Body() dto: ConfigureDietDto,
  ) {
    return this.configureDietUseCase.execute({ userId, ...dto });
  }

  // ← Somente admins podem deletar
  @Delete(':id')
  @Roles(Role.ADMIN)
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: '[Admin] Deletar refeição' })
  async remove(@Param('id') id: string) {
    return this.deleteMealUseCase.execute({ id });
  }
}
```

### 10.2 Decorator no nível do controller (todos os endpoints exigem o role)

```typescript
// Todos os endpoints deste controller exigem role ADMIN
@Roles(Role.ADMIN)
@Controller({ path: 'admin/users', version: '1' })
export class AdminUsersController {
  // Herda @Roles(Role.ADMIN) em todos os endpoints
}
```

### 10.3 Override no método (método tem prioridade sobre classe)

```typescript
// Controller exige ADMIN, mas um método específico permite USER também
@Roles(Role.ADMIN)
@Controller({ path: 'nutritionists', version: '1' })
export class NutritionistsController {

  @Get('my-patients')
  @Roles(Role.NUTRITIONIST)   // ← Sobrescreve: apenas NUTRITIONIST neste método
  async getMyPatients(@CurrentUser() user: UserPayload) {
    return this.getMyPatientsUseCase.execute({ nutritionistId: user.id });
  }
}
```

> `reflector.getAllAndOverride` garante que o `@Roles` do **método** tem prioridade sobre o `@Roles` do **controller**. Isso é intencional e consistente com o comportamento padrão do NestJS com `getAllAndOverride`.

---

## 11. Rotas Públicas com `@Public()`

```typescript
// src/modules/auth/controllers/auth.controller.ts
import { Controller, Post, Body } from '@nestjs/common';
import { Public } from '@common/decorators/public.decorator';

@ApiTags('auth')
@Controller({ path: 'auth', version: '1' })
export class AuthController {

  @Public()
  @Post('login')
  async login(@Body() dto: LoginDto) { ... }

  @Public()
  @Post('register')
  async register(@Body() dto: RegisterDto) { ... }

  // Sem @Public() → exige JWT
  @Get('me')
  async me(@CurrentUser() user: UserPayload) { ... }
}
```

---

## 12. Hierarquia de Roles — `AccessControlService`

Para sistemas onde um role mais alto implica os privilégios dos roles abaixo (ex: `ADMIN` pode fazer tudo que `USER` faz), implemente o `AccessControlService`:

```typescript
// src/modules/auth/services/access-control.service.ts
import { Injectable } from '@nestjs/common';
import { Role } from '../enums/role.enum';

interface HierarchyParams {
  currentRole: Role;
  requiredRole: Role;
}

/**
 * Serviço de controle de acesso com hierarquia de roles.
 *
 * Hierarquias configuradas:
 *   ADMIN > NUTRITIONIST > USER
 *
 * Regra: se o usuário tem um role MAIOR ou IGUAL ao exigido, o acesso é permitido.
 */
@Injectable()
export class AccessControlService {
  private hierarchies: Array<Map<string, number>> = [];
  private priority = 1;

  constructor() {
    // Cadeia hierárquica principal: mais à esquerda = menos privilegiado
    this.buildHierarchy([Role.USER, Role.NUTRITIONIST, Role.ADMIN]);
    // Outras cadeias podem ser adicionadas aqui para grupos paralelos
    // this.buildHierarchy([Role.VIEWER, Role.EDITOR]);
  }

  /**
   * Verifica se `currentRole` tem privilégios suficientes para `requiredRole`.
   *
   * @example
   * isAuthorized({ currentRole: Role.ADMIN, requiredRole: Role.USER }) // → true
   * isAuthorized({ currentRole: Role.USER, requiredRole: Role.ADMIN }) // → false
   */
  isAuthorized({ currentRole, requiredRole }: HierarchyParams): boolean {
    for (const hierarchy of this.hierarchies) {
      const currentLevel = hierarchy.get(currentRole);
      const requiredLevel = hierarchy.get(requiredRole);

      if (currentLevel !== undefined && requiredLevel !== undefined) {
        return currentLevel >= requiredLevel; // maior número = maior privilégio
      }
    }
    return false;
  }

  /**
   * Constrói uma cadeia hierárquica de roles.
   * O primeiro elemento da array tem menor privilégio; o último, maior.
   */
  private buildHierarchy(roles: Role[]): void {
    const hierarchy = new Map<string, number>();
    roles.forEach((role, index) => {
      hierarchy.set(role, this.priority + index);
    });
    this.hierarchies.push(hierarchy);
    this.priority += roles.length;
  }
}
```

### Usando o `AccessControlService` no `RolesGuard`

```typescript
// src/common/guards/roles.guard.ts — versão com hierarquia
import { AccessControlService } from '@modules/auth/services/access-control.service';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly accessControl: AccessControlService, // ← injetado
  ) {}

  canActivate(context: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles || requiredRoles.length === 0) return true;

    const { user } = context.switchToHttp().getRequest<{ user: UserPayload }>();

    if (!user) {
      throw new ForbiddenException('Usuário não autenticado.');
    }

    // Verifica se o usuário tem ALGUM role que satisfaça a hierarquia
    const hasAccess = requiredRoles.some((requiredRole) =>
      user.roles.some((currentRole) =>
        this.accessControl.isAuthorized({
          currentRole,
          requiredRole,
        }),
      ),
    );

    if (!hasAccess) {
      throw new ForbiddenException(
        `Acesso negado. Privilégio insuficiente para esta operação.`,
      );
    }

    return true;
  }
}
```

> **Quando usar hierarquia vs verificação simples?**
> - **Simples (`user.roles.includes(role)`):** Adequado para a maioria dos casos onde os roles são independentes (ex: `NUTRITIONIST` não implica `USER`).
> - **Hierárquica (`AccessControlService`):** Use quando roles formam uma cadeia progressiva de privilégios e um role mais alto deve herdar os privilégios dos mais baixos.
>
> **Para o CAB 2.0**, a hierarquia é mais intuitiva: um ADMIN pode fazer tudo que um NUTRITIONIST faz, e um NUTRITIONIST pode fazer tudo que um USER faz.

---

## 13. RBAC + ABAC: Verificação de Ownership no UseCase

Guards verificam **roles** (quem você é). Verificações de **propriedade de recurso** (ex: "só o dono pode editar") são **regras de negócio** que ficam no UseCase, não no Guard.

```typescript
// src/modules/meals/use-cases/delete-meal/delete-meal.use-case.ts
import { Injectable, ForbiddenException, NotFoundException } from '@nestjs/common';
import { MealsRepository } from '@modules/meals/repositories/meals.repository';
import { Role } from '@modules/auth/enums/role.enum';

type DeleteMealInput = {
  mealId: string;
  requestingUser: { id: string; roles: Role[] };
};

@Injectable()
export class DeleteMealUseCase {
  constructor(private readonly mealsRepository: MealsRepository) {}

  async execute({ mealId, requestingUser }: DeleteMealInput): Promise<void> {
    const meal = await this.mealsRepository.findById(mealId);

    if (!meal) {
      throw new NotFoundException('Refeição não encontrada.');
    }

    // Regra de ownership — apenas o dono da refeição OU um admin pode deletar
    const isOwner = meal.userId === requestingUser.id;
    const isAdmin = requestingUser.roles.includes(Role.ADMIN);

    if (!isOwner && !isAdmin) {
      throw new ForbiddenException('Você não tem permissão para deletar esta refeição.');
    }

    await this.mealsRepository.softDeleteById(mealId);
  }
}
```

```typescript
// Controller — passa o usuário autenticado para o UseCase
@Delete(':id')
@HttpCode(HttpStatus.NO_CONTENT)
async remove(
  @Param('id') id: string,
  @CurrentUser() user: UserPayload,
) {
  await this.deleteMealUseCase.execute({
    mealId: id,
    requestingUser: user,     // ← Controller enriquece com o usuário autenticado
  });
}
```

> **Divisão de responsabilidades:**
> - `RolesGuard` → verifica se o usuário **pode fazer a ação** (role mínimo para acessar o endpoint)
> - `UseCase` → verifica se o usuário **pode agir sobre aquele recurso específico** (ownership, status do recurso)

---

## 14. Integração com GraphQL (`GqlAuthGuard`)

O `GqlAuthGuard` já adapta o contexto de execução do GraphQL (definido em `graphql-pattern.md`). Para RBAC em resolvers, crie um `GqlRolesGuard` análogo:

```typescript
// src/common/guards/gql-roles.guard.ts
import { Injectable, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';
import { RolesGuard } from './roles.guard';

@Injectable()
export class GqlRolesGuard extends RolesGuard {
  /**
   * Sobrescreve apenas a extração do request — a lógica de roles é herdada do RolesGuard.
   */
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req; // ← extrai req do contexto GraphQL
  }

  canActivate(context: ExecutionContext): boolean {
    // Adapta o contexto para HTTP-like antes de delegar ao RolesGuard
    const gqlContext = GqlExecutionContext.create(context);
    const request = gqlContext.getContext().req;

    // Substitui o método getRequest do ExecutionContext para usar o req do GraphQL
    // O RolesGuard usa context.switchToHttp().getRequest() — precisamos adaptar
    const adaptedContext = {
      ...context,
      switchToHttp: () => ({
        getRequest: () => request,
      }),
      getHandler: () => context.getHandler(),
      getClass: () => context.getClass(),
    } as unknown as ExecutionContext;

    return super.canActivate(adaptedContext);
  }
}
```

**Uso no Resolver:**

```typescript
@Resolver(() => MealType)
@UseGuards(GqlAuthGuard, GqlRolesGuard)
export class MealsResolver {

  @Mutation(() => MealType)
  @Roles(Role.ADMIN, Role.NUTRITIONIST)
  async configureDiet(...) { ... }
}
```

---

## 15. Integração com OpenAPI/Swagger

Documente os requisitos de roles nos endpoints:

```typescript
import { ApiSecurity, ApiForbiddenResponse } from '@nestjs/swagger';

@Delete(':id')
@Roles(Role.ADMIN)
@ApiBearerAuth('access-token')
@ApiOperation({ summary: '[Admin] Deletar refeição' })
@ApiForbiddenResponse({ description: 'Acesso negado. Role admin necessário.' })
async remove(@Param('id') id: string) { ... }
```

### Documentando roles nos `@ApiTags`

Use a descrição do tag para indicar os roles relevantes:

```typescript
// swagger.config.ts (em src/config/)
.addTag('meals', 'Refeições e histórico nutricional')
.addTag('meals-admin', '[Admin] Gerenciamento administrativo de refeições')
.addTag('nutritionist', '[Nutritionist] Configuração de dietas para pacientes')
```

---

## 16. Registro do `AccessControlService` no Módulo

```typescript
// src/modules/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { AccessControlService } from './services/access-control.service';

@Module({
  providers: [
    // ... use cases e repositórios
    AccessControlService,   // ← registrado como provider
  ],
  exports: [
    AccessControlService,   // ← exportado para o RolesGuard em src/common/
  ],
})
export class AuthModule {}
```

```typescript
// src/app.module.ts — o RolesGuard global consegue injetar o AccessControlService
// porque o AuthModule é importado no AppModule
@Module({
  imports: [
    DatabaseModule,
    AuthModule,    // ← exporta AccessControlService
    UsersModule,
    MealsModule,
  ],
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AppModule {}
```

> **Atenção:** Ao usar `APP_GUARD` globalmente, o guard é instanciado pelo contexto do `AppModule`. Para injetar dependências de outros módulos (como `AccessControlService`), o módulo que exporta o serviço **deve ser importado** no `AppModule`.

---

## 17. Escalando para ABAC — Quando e Como

### Quando RBAC puro não é suficiente

O RBAC simples começa a mostrar limitações quando surgem regras como:

- "Nutricionista pode editar apenas os pacientes que ele vinculou."
- "Usuário pode ver apenas as próprias refeições, mas admins veem tudo."
- "Refeições publicadas na Comunidade só podem ser removidas pelo autor."

Esses são casos de **ABAC (Attribute-Based Access Control)** — as permissões dependem de atributos do recurso (ex: `meal.userId === user.id`).

### Estratégia recomendada: RBAC para endpoints + ABAC no UseCase

Para a maioria dos projetos de médio porte, a combinação mais pragmática é:

```
RolesGuard (RBAC)  → Portão de entrada: "usuários deste role têm acesso ao endpoint"
UseCase (ABAC)     → Regra de negócio: "mas apenas se for o dono do recurso"
```

Isso evita a complexidade de CASL em casos simples e mantém as regras de ownership no lugar correto: a camada de Application.

### Quando introduzir CASL

Use a biblioteca `@casl/ability` quando a lógica de ownership for muito repetitiva em vários UseCases e você precisar de um **único ponto de definição de políticas**:

```bash
npm install @casl/ability
```

```typescript
// src/modules/auth/casl/casl-ability.factory.ts
import { Injectable } from '@nestjs/common';
import {
  AbilityBuilder,
  createMongoAbility,
  InferSubjects,
  MongoAbility,
} from '@casl/ability';
import { Role } from '../enums/role.enum';
import { MealEntity } from '@modules/meals/entities/meal.entity';
import { UserEntity } from '@modules/users/entities/user.entity';
import type { UserPayload } from '@common/types/user-payload.type';

export enum Action {
  Manage = 'manage', // qualquer ação
  Create = 'create',
  Read   = 'read',
  Update = 'update',
  Delete = 'delete',
}

type Subjects = InferSubjects<typeof MealEntity | typeof UserEntity> | 'all';
export type AppAbility = MongoAbility<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  /**
   * Cria as regras de habilidade (can/cannot) para um usuário autenticado.
   * Centralize TODAS as regras de ownership aqui em vez de espalhá-las nos UseCases.
   */
  createForUser(user: UserPayload): AppAbility {
    const { can, cannot, build } = new AbilityBuilder<AppAbility>(createMongoAbility);

    if (user.roles.includes(Role.ADMIN)) {
      can(Action.Manage, 'all'); // Admin pode fazer tudo
    } else if (user.roles.includes(Role.NUTRITIONIST)) {
      can(Action.Read, 'all');
      can(Action.Update, UserEntity, { linkedNutritionistId: user.id }); // só seus pacientes
    } else {
      // Role.USER
      can(Action.Read, MealEntity, { userId: user.id });      // só suas refeições
      can(Action.Create, MealEntity);
      can(Action.Update, MealEntity, { userId: user.id });
      can(Action.Delete, MealEntity, { userId: user.id });
      cannot(Action.Delete, MealEntity, { status: 'published' }); // não pode deletar publicadas
    }

    return build({
      detectSubjectType: (item) => item.constructor as any,
    });
  }
}
```

```typescript
// src/modules/auth/casl/casl.module.ts
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```

**Uso no UseCase com CASL:**

```typescript
// src/modules/meals/use-cases/delete-meal/delete-meal.use-case.ts
@Injectable()
export class DeleteMealUseCase {
  constructor(
    private readonly mealsRepository: MealsRepository,
    private readonly caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async execute({ mealId, requestingUser }: DeleteMealInput): Promise<void> {
    const meal = await this.mealsRepository.findById(mealId);
    if (!meal) throw new NotFoundException('Refeição não encontrada.');

    const ability = this.caslAbilityFactory.createForUser(requestingUser);

    // Verificação unificada de RBAC + ownership em uma linha
    if (ability.cannot(Action.Delete, meal)) {
      throw new ForbiddenException('Você não pode deletar esta refeição.');
    }

    await this.mealsRepository.softDeleteById(mealId);
  }
}
```

> **Regra:** Introduza CASL apenas quando houver **3 ou mais UseCases** repetindo a mesma lógica de ownership. Para casos isolados, a verificação manual no UseCase (`isOwner || isAdmin`) é mais legível e simples.

---

## 18. Anti-Padrões Comuns

### ❌ Verificar roles no UseCase em vez de no Guard

```typescript
// ❌ ERRADO — lógica de autorização por role no UseCase
export class CreateMealUseCase {
  async execute(input: CreateMealInput): Promise<MealEntity> {
    if (!input.user.roles.includes(Role.USER)) {  // ← não faz sentido aqui
      throw new ForbiddenException('...');
    }
    // ...
  }
}

// ✅ CORRETO — role no Guard, ownership no UseCase
// Guard: @Roles(Role.USER) no Controller
// UseCase: verifica apenas regras de negócio e ownership
```

### ❌ Guard sem ordem definida (autenticação antes de autorização)

```typescript
// ❌ ERRADO — RolesGuard antes do JwtAuthGuard
providers: [
  { provide: APP_GUARD, useClass: RolesGuard },   // ← request.user ainda não existe!
  { provide: APP_GUARD, useClass: JwtAuthGuard },
]

// ✅ CORRETO — JwtAuthGuard sempre primeiro
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard },
  { provide: APP_GUARD, useClass: RolesGuard },
]
```

### ❌ Usar `return false` em vez de lançar exceção no Guard

```typescript
// ❌ ERRADO — mensagem genérica do NestJS: "Forbidden resource"
canActivate(): boolean {
  if (!hasRole) return false; // ← mensagem pouco informativa para o cliente
}

// ✅ CORRETO — mensagem específica via ForbiddenException + HttpExceptionFilter
canActivate(): boolean {
  if (!hasRole) throw new ForbiddenException('Acesso negado. Role admin necessário.');
}
```

### ❌ Incluir lógica de ownership no Guard

```typescript
// ❌ ERRADO — Guard acessa o banco de dados para verificar ownership
canActivate(context: ExecutionContext): Promise<boolean> {
  const mealId = context.switchToHttp().getRequest().params.id;
  const meal = await this.mealsRepository.findById(mealId); // ← Guard acessando repositório
  return meal.userId === user.id;
}

// ✅ CORRETO — Guard verifica apenas roles; UseCase verifica ownership
```

### ❌ Roles no JWT como `string[]` sem validação

```typescript
// ❌ ERRADO — aceita qualquer string como role
const roles = payload.roles; // ['admin', 'superuser', 'hacker'] — sem validação!

// ✅ CORRETO — valida que os roles são valores do enum Role
const roles = payload.roles.filter((r) => Object.values(Role).includes(r)) as Role[];
```

### ❌ Esquecer o `@Public()` no JwtAuthGuard

```typescript
// ❌ ERRADO — JwtAuthGuard não respeita @Public(), bloqueia rotas abertas
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context); // ← sempre exige JWT
  }
}

// ✅ CORRETO — verifica @Public() antes de validar o JWT
canActivate(context: ExecutionContext) {
  const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [...]);
  if (isPublic) return true;
  return super.canActivate(context);
}
```

---

## 19. Testes de Guards

### Testando o `RolesGuard`

```typescript
// src/common/guards/__tests__/roles.guard.spec.ts
import { Test } from '@nestjs/testing';
import { Reflector } from '@nestjs/core';
import { ForbiddenException } from '@nestjs/common';
import { RolesGuard } from '../roles.guard';
import { Role } from '@modules/auth/enums/role.enum';

function createMockContext(user: { roles: Role[] }, handler = {}): any {
  return {
    getHandler: () => handler,
    getClass:   () => ({}),
    switchToHttp: () => ({
      getRequest: () => ({ user }),
    }),
  };
}

describe('RolesGuard', () => {
  let guard: RolesGuard;
  let reflector: jest.Mocked<Reflector>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        RolesGuard,
        {
          provide: Reflector,
          useValue: {
            getAllAndOverride: jest.fn(),
          },
        },
      ],
    }).compile();

    guard = module.get(RolesGuard);
    reflector = module.get(Reflector);
  });

  it('deve permitir acesso quando não há @Roles() declarado', () => {
    reflector.getAllAndOverride
      .mockReturnValueOnce(false)   // IS_PUBLIC_KEY
      .mockReturnValueOnce(null);   // ROLES_KEY

    const context = createMockContext({ roles: [Role.USER] });
    expect(guard.canActivate(context)).toBe(true);
  });

  it('deve permitir acesso quando o usuário tem o role exigido', () => {
    reflector.getAllAndOverride
      .mockReturnValueOnce(false)           // IS_PUBLIC_KEY
      .mockReturnValueOnce([Role.ADMIN]);   // ROLES_KEY

    const context = createMockContext({ roles: [Role.ADMIN] });
    expect(guard.canActivate(context)).toBe(true);
  });

  it('deve lançar ForbiddenException quando o usuário não tem o role exigido', () => {
    reflector.getAllAndOverride
      .mockReturnValueOnce(false)           // IS_PUBLIC_KEY
      .mockReturnValueOnce([Role.ADMIN]);   // ROLES_KEY

    const context = createMockContext({ roles: [Role.USER] });
    expect(() => guard.canActivate(context)).toThrow(ForbiddenException);
  });

  it('deve permitir acesso em rotas @Public()', () => {
    reflector.getAllAndOverride.mockReturnValueOnce(true); // IS_PUBLIC_KEY

    const context = createMockContext({ roles: [] });
    expect(guard.canActivate(context)).toBe(true);
  });

  it('deve permitir acesso quando o usuário tem UM dos roles exigidos (OR)', () => {
    reflector.getAllAndOverride
      .mockReturnValueOnce(false)
      .mockReturnValueOnce([Role.ADMIN, Role.NUTRITIONIST]);

    const context = createMockContext({ roles: [Role.NUTRITIONIST] });
    expect(guard.canActivate(context)).toBe(true);
  });
});
```

---

## 20. Checklist de Implementação

### Configuração Base
- [ ] `Role` enum criado em `src/modules/auth/enums/role.enum.ts` com strings descritivas?
- [ ] `@Roles()` decorator em `src/common/decorators/roles.decorator.ts` com `ROLES_KEY` exportada?
- [ ] `@Public()` decorator em `src/common/decorators/public.decorator.ts` com `IS_PUBLIC_KEY` exportada?
- [ ] `UserPayload` tipado com campo `roles: Role[]`?
- [ ] `JwtStrategy.validate()` inclui `roles` no retorno?
- [ ] JWT gerado pelo login inclui `roles` no payload?

### Guards
- [ ] `JwtAuthGuard` respeita `@Public()` via `Reflector`?
- [ ] `RolesGuard` usa `ForbiddenException` (não `return false`) para erros?
- [ ] `RolesGuard` usa `getAllAndOverride` (método tem prioridade sobre classe)?
- [ ] Registro global no `AppModule`: `JwtAuthGuard` antes de `RolesGuard`?
- [ ] `AuthModule` exporta `AccessControlService` se usando hierarquia?

### Controllers
- [ ] Rotas públicas usam `@Public()`?
- [ ] Endpoints restritos usam `@Roles(Role.X)`?
- [ ] Documentação Swagger com `@ApiForbiddenResponse` nos endpoints protegidos?

### UseCase (Ownership / ABAC)
- [ ] Verificações de ownership estão **no UseCase**, não no Guard?
- [ ] UseCase recebe `requestingUser: UserPayload` como parte do input?
- [ ] Erros de ownership lançam `ForbiddenException` com mensagem contextualizada?

### Testes
- [ ] `RolesGuard` testado com mocks do `Reflector` e contexto simulado?
- [ ] Cenários testados: sem roles, role correto, role errado, rota pública?

---

## Referências

- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Guards ficam na camada de Presentation; UseCase não verifica roles
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Localização de guards em `src/common/guards/`, decorators em `src/common/decorators/`
- [nest-modules-pattern.md](.agent/skills/backend/foundation-and-architecture/nest-modules-pattern.md) — Exportação do `AccessControlService` via `AuthModule`
- [rest-api-patterns.md](.agent/skills/backend/api-and-contracts/rest-api-patterns.md) — 401 vs 403, `HttpExceptionFilter`, formato de erro padronizado
- [graphql-pattern.md](.agent/skills/backend/api-and-contracts/graphql-pattern.md) — `GqlAuthGuard` como base para `GqlRolesGuard`
- [openapi-docs.md](.agent/skills/backend/api-and-contracts/openapi-docs.md) — `@ApiForbiddenResponse` e documentação de endpoints protegidos
- [dependency-injection.md](.agent/skills/backend/configuration-and-environment/dependency-injection.md) — `APP_GUARD` como provider global via token simbólico
- [NestJS Docs — Authorization](https://docs.nestjs.com/security/authorization)
- [CASL — Isomorphic Authorization](https://casl.js.org/)