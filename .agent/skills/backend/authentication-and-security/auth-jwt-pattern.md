---
name: auth-jwt-pattern
description: Autenticação completa no NestJS com JWT — access token (15min) + refresh token opaco com rotação e reuse detection, guard global manual sem Passport, @Public() para rotas abertas, @CurrentUser() decorator, blacklist Redis para logout de access tokens, tabela dedicada refresh_tokens com hash SHA-256, e mapeamento completo de onde cada responsabilidade vive. Complementa clean-architecture.md (UseCase como camada de aplicação), config-management.md (jwt.config.ts), nest-project-structure.md (estrutura de pastas), rest-api-patterns.md (envelope de resposta e status HTTP), e dependency-injection.md (tokens customizados).
---

# Autenticação JWT — NestJS (Access + Refresh + Guard Global + Redis Blacklist)

Você é um Arquiteto de Software Senior especialista em segurança. Esta skill define o **padrão completo de autenticação JWT** para o projeto NestJS. Toda a lógica de auth deve seguir estas convenções.

> **Decisões de arquitetura (não altere sem revisão):**
> - Guard **manual** com `JwtService.verify()` — sem `@nestjs/passport`, sem Passport.js.
> - Refresh token **opaco** (UUID v4) armazenado como hash SHA-256 em tabela dedicada `refresh_tokens`.
> - Access token na blacklist Redis **apenas no logout** (TTL = tempo restante até expiração).
> - Rotação completa com **reuse detection**: refresh token usado uma vez → novo token emitido → antigo invalidado → se reutilizado, toda a família é revogada.

---

## 1. Estrutura de Pastas

```
src/
├── modules/
│   └── auth/
│       ├── controllers/
│       │   └── auth.controller.ts
│       ├── use-cases/
│       │   ├── login/
│       │   │   └── login.use-case.ts
│       │   ├── refresh-token/
│       │   │   └── refresh-token.use-case.ts
│       │   └── logout/
│       │       └── logout.use-case.ts
│       ├── repositories/
│       │   └── refresh-token.repository.ts
│       ├── entities/
│       │   └── refresh-token.entity.ts      ← tabela refresh_tokens
│       ├── dtos/
│       │   ├── login.dto.ts
│       │   └── refresh-token.dto.ts
│       ├── strategies/                       ← apenas tipagem/helpers, sem Passport
│       │   └── jwt.payload.type.ts
│       └── auth.module.ts
│
├── common/
│   ├── decorators/
│   │   ├── current-user.decorator.ts        ← @CurrentUser()
│   │   └── public.decorator.ts              ← @Public()
│   ├── guards/
│   │   └── jwt-auth.guard.ts                ← guard global manual
│   └── types/
│       └── user-payload.type.ts             ← tipo do JWT decoded
│
└── infra/
    └── cache/
        └── cache.module.ts                  ← Redis (já definido em query-optimization.md)
```

---

## 2. Variáveis de Ambiente

Complementa `config-management.md` — as variáveis já estão definidas em `env.config.ts`. Verifique que existem:

```typescript
// src/config/env.config.ts — campos relevantes para auth
const envSchema = z.object({
  JWT_ACCESS_SECRET:      z.string().min(32, 'JWT_ACCESS_SECRET deve ter ≥ 32 caracteres'),
  JWT_ACCESS_EXPIRATION:  z.string().default('15m'),   // 15 minutos — nunca mais que 1h
  JWT_REFRESH_SECRET:     z.string().min(32, 'JWT_REFRESH_SECRET deve ter ≥ 32 caracteres'),
  JWT_REFRESH_EXPIRATION: z.string().default('7d'),    // 7 dias
  // ... outros campos
});
```

```typescript
// src/config/jwt.config.ts
import { registerAs } from '@nestjs/config';
import { env } from './env.config';

export interface JwtConfig {
  accessSecret: string;
  accessExpiration: string;
  refreshSecret: string;
  refreshExpiration: string;
}

export default registerAs('jwt', (): JwtConfig => ({
  accessSecret:      env.JWT_ACCESS_SECRET,
  accessExpiration:  env.JWT_ACCESS_EXPIRATION,
  refreshSecret:     env.JWT_REFRESH_SECRET,
  refreshExpiration: env.JWT_REFRESH_EXPIRATION,
}));
```

> **Segurança:** `JWT_ACCESS_SECRET` e `JWT_REFRESH_SECRET` devem ser segredos **distintos**. Nunca use o mesmo secret para os dois tokens. Em produção, armazene via EAS Secrets / secrets vault — nunca no repositório.

---

## 3. Tipos do Payload JWT

```typescript
// src/common/types/user-payload.type.ts

/**
 * Payload decodificado do access token JWT.
 * Contém apenas dados imutáveis — nunca coloque e-mail ou role mutável aqui.
 * Mudanças de role só refletem no próximo login.
 */
export type UserPayload = {
  /** ID único do usuário (UUID v4) */
  sub: string;
  /** Role no momento do login — usado para RBAC */
  role: UserRole;
  /** JWT ID único — usado para blacklist no logout */
  jti: string;
  /** Issued At (timestamp Unix) */
  iat: number;
  /** Expiration (timestamp Unix) */
  exp: number;
};

export type UserRole = 'user' | 'admin' | 'nutritionist';
```

```typescript
// src/modules/auth/strategies/jwt.payload.type.ts

/**
 * Payload que é assinado dentro do JWT (sem iat/exp — adicionados pelo JwtService).
 * Mantido separado de UserPayload para clareza na camada de geração.
 */
export type JwtSignPayload = {
  sub: string;
  role: UserRole;
  jti: string;
};
```

> **Regra de payload:** Mantenha o payload **mínimo** — apenas `sub`, `role` e `jti`. Nunca inclua e-mail, nome, senha ou qualquer dado sensível. O payload JWT é apenas Base64-encoded, não criptografado.

---

## 4. Entity — `refresh_tokens`

```typescript
// src/modules/auth/entities/refresh-token.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  Index,
  ManyToOne,
  JoinColumn,
} from 'typeorm';
import { UserEntity } from '@modules/users/entities/user.entity';

@Entity('refresh_tokens')
@Index(['userId'])
@Index(['tokenHash'], { unique: true })
export class RefreshTokenEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'uuid', name: 'user_id' })
  userId: string;

  /**
   * Hash SHA-256 do refresh token opaco (UUID v4).
   * Nunca armazenamos o token bruto — apenas o hash.
   */
  @Column({ type: 'varchar', length: 64, name: 'token_hash', unique: true })
  tokenHash: string;

  /**
   * ID da família de tokens — usado para reuse detection.
   * Todos os tokens de uma sessão compartilham o mesmo familyId.
   * Se um token antigo da família for reutilizado, toda a família é revogada.
   */
  @Column({ type: 'uuid', name: 'family_id' })
  familyId: string;

  @Column({ type: 'boolean', default: false, name: 'is_revoked' })
  isRevoked: boolean;

  @Column({ type: 'boolean', default: false, name: 'is_used' })
  isUsed: boolean;

  @Column({ type: 'timestamp', name: 'expires_at' })
  expiresAt: Date;

  @Column({ type: 'varchar', length: 512, nullable: true, name: 'user_agent' })
  userAgent: string | null;

  @Column({ type: 'varchar', length: 45, nullable: true, name: 'ip_address' })
  ipAddress: string | null;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @Column({ type: 'timestamp', nullable: true, name: 'used_at' })
  usedAt: Date | null;

  @ManyToOne(() => UserEntity, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user: UserEntity;
}
```

---

## 5. Repository — `RefreshTokenRepository`

```typescript
// src/modules/auth/repositories/refresh-token.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, LessThan } from 'typeorm';
import { createHash } from 'crypto';
import { RefreshTokenEntity } from '../entities/refresh-token.entity';

type CreateRefreshTokenData = {
  userId: string;
  tokenHash: string;
  familyId: string;
  expiresAt: Date;
  userAgent?: string | null;
  ipAddress?: string | null;
};

@Injectable()
export class RefreshTokenRepository {
  constructor(
    @InjectRepository(RefreshTokenEntity)
    private readonly repo: Repository<RefreshTokenEntity>,
  ) {}

  // ─── Helpers ──────────────────────────────────────────────────────────────

  /** Gera o hash SHA-256 de um token opaco (UUID v4 bruto) */
  static hash(rawToken: string): string {
    return createHash('sha256').update(rawToken).digest('hex');
  }

  // ─── CREATE ───────────────────────────────────────────────────────────────

  async create(data: CreateRefreshTokenData): Promise<RefreshTokenEntity> {
    const entity = this.repo.create(data);
    return this.repo.save(entity);
  }

  // ─── READ ─────────────────────────────────────────────────────────────────

  async findByHash(tokenHash: string): Promise<RefreshTokenEntity | null> {
    return this.repo.findOne({ where: { tokenHash } });
  }

  async findValidByHash(tokenHash: string): Promise<RefreshTokenEntity | null> {
    return this.repo.findOne({
      where: {
        tokenHash,
        isRevoked: false,
        isUsed: false,
      },
    });
  }

  // ─── UPDATE ───────────────────────────────────────────────────────────────

  /** Marca o token como usado (após rotação) */
  async markAsUsed(id: string): Promise<void> {
    await this.repo.update(id, { isUsed: true, usedAt: new Date() });
  }

  // ─── REVOKE ───────────────────────────────────────────────────────────────

  /** Revoga um token específico (logout de uma sessão) */
  async revokeById(id: string): Promise<void> {
    await this.repo.update(id, { isRevoked: true });
  }

  /** Revoga toda a família — usado no reuse detection */
  async revokeFamily(familyId: string): Promise<void> {
    await this.repo.update({ familyId }, { isRevoked: true });
  }

  /** Revoga todos os tokens de um usuário (logout de todos os dispositivos) */
  async revokeAllByUserId(userId: string): Promise<void> {
    await this.repo.update({ userId }, { isRevoked: true });
  }

  // ─── CLEANUP ──────────────────────────────────────────────────────────────

  /** Remove tokens expirados — ideal para rodar via cron (ex: nightly) */
  async deleteExpired(): Promise<void> {
    await this.repo.delete({ expiresAt: LessThan(new Date()) });
  }
}
```

---

## 6. DTOs

```typescript
// src/modules/auth/dtos/login.dto.ts
import { IsEmail, IsString, MinLength } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class LoginDto {
  @ApiProperty({ example: 'user@exemplo.com.br' })
  @IsEmail({}, { message: 'Informe um e-mail válido.' })
  email: string;

  @ApiProperty({ example: 'Senha@123' })
  @IsString()
  @MinLength(8, { message: 'Senha deve ter no mínimo 8 caracteres.' })
  password: string;
}
```

```typescript
// src/modules/auth/dtos/refresh-token.dto.ts
import { IsString, IsUUID } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class RefreshTokenDto {
  @ApiProperty({ description: 'Refresh token opaco emitido no login ou em rotação anterior' })
  @IsString()
  @IsUUID('4')
  refreshToken: string;
}
```

---

## 7. Guard Global — `JwtAuthGuard`

O guard é registrado globalmente via `APP_GUARD` no `AppModule`. Toda rota é protegida por padrão. Para tornar uma rota pública, use `@Public()`.

```typescript
// src/common/guards/jwt-auth.guard.ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { JwtService } from '@nestjs/jwt';
import { ConfigType } from '@nestjs/config';
import { Inject } from '@nestjs/common';
import { Request } from 'express';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import jwtConfig from '@config/jwt.config';
import { IS_PUBLIC_KEY } from '@common/decorators/public.decorator';
import type { UserPayload } from '@common/types/user-payload.type';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly jwtService: JwtService,
    @Inject(jwtConfig.KEY)
    private readonly jwt: ConfigType<typeof jwtConfig>,
    @Inject(CACHE_MANAGER)
    private readonly cache: Cache,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 1. Verifica se a rota é pública (@Public())
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    // 2. Extrai o token do header Authorization: Bearer <token>
    const request = context.switchToHttp().getRequest<Request>();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException('Token de acesso ausente.');
    }

    // 3. Verifica assinatura e expiração
    let payload: UserPayload;
    try {
      payload = await this.jwtService.verifyAsync<UserPayload>(token, {
        secret: this.jwt.accessSecret,
      });
    } catch {
      throw new UnauthorizedException('Token de acesso inválido ou expirado.');
    }

    // 4. Verifica blacklist Redis (logout)
    const isBlacklisted = await this.cache.get<boolean>(`bl:${payload.jti}`);
    if (isBlacklisted) {
      throw new UnauthorizedException('Token revogado. Faça login novamente.');
    }

    // 5. Injeta o payload na request — acessível via @CurrentUser()
    request['user'] = payload;
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

---

## 8. Decorators

### 8.1 `@Public()`

```typescript
// src/common/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';

/**
 * Marca uma rota como pública — o JwtAuthGuard global não exige autenticação.
 *
 * @example
 * @Public()
 * @Post('login')
 * async login(@Body() dto: LoginDto) { ... }
 */
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

### 8.2 `@CurrentUser()`

```typescript
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import type { UserPayload } from '@common/types/user-payload.type';

/**
 * Extrai o payload do JWT da request.
 * Só funciona em rotas protegidas pelo JwtAuthGuard.
 *
 * @example
 * @Get('profile')
 * async getProfile(@CurrentUser() user: UserPayload) {
 *   return this.getProfileUseCase.execute({ userId: user.sub });
 * }
 */
export const CurrentUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): UserPayload => {
    const request = ctx.switchToHttp().getRequest();
    return request.user as UserPayload;
  },
);
```

---

## 9. Use Cases

### 9.1 `LoginUseCase`

```typescript
// src/modules/auth/use-cases/login/login.use-case.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigType } from '@nestjs/config';
import { Inject } from '@nestjs/common';
import { randomUUID } from 'crypto';
import * as bcrypt from 'bcrypt';
import jwtConfig from '@config/jwt.config';
import { UsersRepository } from '@modules/users/repositories/users.repository';
import { RefreshTokenRepository } from '../../repositories/refresh-token.repository';
import type { JwtSignPayload } from '../../strategies/jwt.payload.type';

type LoginInput = {
  email: string;
  password: string;
  userAgent?: string | null;
  ipAddress?: string | null;
};

type LoginOutput = {
  accessToken: string;
  refreshToken: string;
  user: {
    id: string;
    email: string;
    role: string;
  };
};

@Injectable()
export class LoginUseCase {
  constructor(
    private readonly usersRepository: UsersRepository,
    private readonly refreshTokenRepository: RefreshTokenRepository,
    private readonly jwtService: JwtService,
    @Inject(jwtConfig.KEY)
    private readonly jwt: ConfigType<typeof jwtConfig>,
  ) {}

  async execute(input: LoginInput): Promise<LoginOutput> {
    // 1. Busca usuário
    const user = await this.usersRepository.findByEmail(input.email);
    if (!user) {
      // Mesmo delay quando usuário não existe — evita timing attack
      await bcrypt.hash('dummy', 10);
      throw new UnauthorizedException('Credenciais inválidas.');
    }

    // 2. Verifica senha
    const passwordMatch = await bcrypt.compare(input.password, user.passwordHash);
    if (!passwordMatch) {
      throw new UnauthorizedException('Credenciais inválidas.');
    }

    // 3. Gera access token com jti único (para blacklist)
    const jti = randomUUID();
    const accessPayload: JwtSignPayload = { sub: user.id, role: user.role, jti };
    const accessToken = await this.jwtService.signAsync(accessPayload, {
      secret:    this.jwt.accessSecret,
      expiresIn: this.jwt.accessExpiration,
    });

    // 4. Gera refresh token opaco (UUID v4) + hash SHA-256
    const rawRefreshToken = randomUUID();
    const tokenHash = RefreshTokenRepository.hash(rawRefreshToken);
    const familyId = randomUUID(); // nova sessão = nova família

    const refreshExpMs = this.parseExpiration(this.jwt.refreshExpiration);
    const expiresAt = new Date(Date.now() + refreshExpMs);

    await this.refreshTokenRepository.create({
      userId:     user.id,
      tokenHash,
      familyId,
      expiresAt,
      userAgent:  input.userAgent ?? null,
      ipAddress:  input.ipAddress ?? null,
    });

    return {
      accessToken,
      refreshToken: rawRefreshToken, // token bruto — enviado ao cliente
      user: { id: user.id, email: user.email, role: user.role },
    };
  }

  /**
   * Converte string de expiração ('15m', '7d') para milissegundos.
   */
  private parseExpiration(exp: string): number {
    const unit = exp.slice(-1);
    const value = parseInt(exp.slice(0, -1), 10);
    const multipliers: Record<string, number> = {
      s: 1_000,
      m: 60_000,
      h: 3_600_000,
      d: 86_400_000,
    };
    return value * (multipliers[unit] ?? 1_000);
  }
}
```

### 9.2 `RefreshTokenUseCase`

```typescript
// src/modules/auth/use-cases/refresh-token/refresh-token.use-case.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigType } from '@nestjs/config';
import { Inject } from '@nestjs/common';
import { randomUUID } from 'crypto';
import jwtConfig from '@config/jwt.config';
import { RefreshTokenRepository } from '../../repositories/refresh-token.repository';
import type { JwtSignPayload } from '../../strategies/jwt.payload.type';

type RefreshInput = {
  rawRefreshToken: string;
  userAgent?: string | null;
  ipAddress?: string | null;
};

type RefreshOutput = {
  accessToken: string;
  refreshToken: string;
};

@Injectable()
export class RefreshTokenUseCase {
  constructor(
    private readonly refreshTokenRepository: RefreshTokenRepository,
    private readonly jwtService: JwtService,
    @Inject(jwtConfig.KEY)
    private readonly jwt: ConfigType<typeof jwtConfig>,
  ) {}

  async execute(input: RefreshInput): Promise<RefreshOutput> {
    const tokenHash = RefreshTokenRepository.hash(input.rawRefreshToken);

    // 1. Busca o token no banco pelo hash
    const storedToken = await this.refreshTokenRepository.findByHash(tokenHash);
    if (!storedToken) {
      throw new UnauthorizedException('Refresh token inválido.');
    }

    // 2. Reuse Detection — token já foi usado antes → ataque de replay
    if (storedToken.isUsed) {
      // Revoga toda a família (sessão comprometida)
      await this.refreshTokenRepository.revokeFamily(storedToken.familyId);
      throw new UnauthorizedException(
        'Replay attack detectado. Sessão encerrada por segurança. Faça login novamente.',
      );
    }

    // 3. Verifica se está revogado (logout manual) ou expirado
    if (storedToken.isRevoked) {
      throw new UnauthorizedException('Refresh token revogado. Faça login novamente.');
    }
    if (storedToken.expiresAt < new Date()) {
      throw new UnauthorizedException('Refresh token expirado. Faça login novamente.');
    }

    // 4. Marca o token atual como USADO (single-use)
    await this.refreshTokenRepository.markAsUsed(storedToken.id);

    // 5. Emite novo access token
    const jti = randomUUID();
    const accessPayload: JwtSignPayload = {
      sub:  storedToken.userId,
      role: storedToken.user?.role ?? 'user',
      jti,
    };
    const accessToken = await this.jwtService.signAsync(accessPayload, {
      secret:    this.jwt.accessSecret,
      expiresIn: this.jwt.accessExpiration,
    });

    // 6. Emite novo refresh token (rotação) — mesmo familyId
    const newRawToken = randomUUID();
    const newTokenHash = RefreshTokenRepository.hash(newRawToken);
    const refreshExpMs = this.parseExpiration(this.jwt.refreshExpiration);
    const expiresAt = new Date(Date.now() + refreshExpMs);

    await this.refreshTokenRepository.create({
      userId:    storedToken.userId,
      tokenHash: newTokenHash,
      familyId:  storedToken.familyId, // ← mesma família
      expiresAt,
      userAgent: input.userAgent ?? storedToken.userAgent,
      ipAddress: input.ipAddress ?? storedToken.ipAddress,
    });

    return { accessToken, refreshToken: newRawToken };
  }

  private parseExpiration(exp: string): number {
    const unit = exp.slice(-1);
    const value = parseInt(exp.slice(0, -1), 10);
    const multipliers: Record<string, number> = {
      s: 1_000, m: 60_000, h: 3_600_000, d: 86_400_000,
    };
    return value * (multipliers[unit] ?? 1_000);
  }
}
```

### 9.3 `LogoutUseCase`

```typescript
// src/modules/auth/use-cases/logout/logout.use-case.ts
import { Inject, Injectable } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import { RefreshTokenRepository } from '../../repositories/refresh-token.repository';
import type { UserPayload } from '@common/types/user-payload.type';

type LogoutInput = {
  /** Payload do access token atual (vem do @CurrentUser()) */
  user: UserPayload;
  /** Token bruto enviado no body (opcional — logout de sessão específica) */
  rawRefreshToken?: string | null;
  /** Se true, revoga TODOS os refresh tokens do usuário (logout de todos os dispositivos) */
  allDevices?: boolean;
};

@Injectable()
export class LogoutUseCase {
  constructor(
    private readonly refreshTokenRepository: RefreshTokenRepository,
    @Inject(CACHE_MANAGER)
    private readonly cache: Cache,
  ) {}

  async execute(input: LogoutInput): Promise<void> {
    // 1. Adiciona o access token atual na blacklist Redis
    //    TTL = tempo restante até expiração (não mantém na blacklist mais que necessário)
    const nowSeconds = Math.floor(Date.now() / 1_000);
    const ttlSeconds = Math.max(0, input.user.exp - nowSeconds);

    if (ttlSeconds > 0) {
      await this.cache.set(`bl:${input.user.jti}`, true, ttlSeconds * 1_000);
    }

    // 2. Revoga o(s) refresh token(s) no banco
    if (input.allDevices) {
      // Logout de todos os dispositivos
      await this.refreshTokenRepository.revokeAllByUserId(input.user.sub);
    } else if (input.rawRefreshToken) {
      // Logout da sessão atual pelo hash
      const hash = RefreshTokenRepository.hash(input.rawRefreshToken);
      const token = await this.refreshTokenRepository.findByHash(hash);
      if (token && token.userId === input.user.sub) {
        await this.refreshTokenRepository.revokeById(token.id);
      }
    }
  }
}
```

---

## 10. Controller

```typescript
// src/modules/auth/controllers/auth.controller.ts
import {
  Body,
  Controller,
  HttpCode,
  HttpStatus,
  Post,
  Req,
} from '@nestjs/common';
import {
  ApiTags,
  ApiOperation,
  ApiOkResponse,
  ApiCreatedResponse,
  ApiUnauthorizedResponse,
  ApiBearerAuth,
} from '@nestjs/swagger';
import { Request } from 'express';
import { Public } from '@common/decorators/public.decorator';
import { CurrentUser } from '@common/decorators/current-user.decorator';
import type { UserPayload } from '@common/types/user-payload.type';
import { LoginDto } from '../dtos/login.dto';
import { RefreshTokenDto } from '../dtos/refresh-token.dto';
import { LoginUseCase } from '../use-cases/login/login.use-case';
import { RefreshTokenUseCase } from '../use-cases/refresh-token/refresh-token.use-case';
import { LogoutUseCase } from '../use-cases/logout/logout.use-case';

@ApiTags('auth')
@Controller({ path: 'auth', version: '1' })
export class AuthController {
  constructor(
    private readonly loginUseCase: LoginUseCase,
    private readonly refreshTokenUseCase: RefreshTokenUseCase,
    private readonly logoutUseCase: LogoutUseCase,
  ) {}

  // ─── POST /api/v1/auth/login ──────────────────────────────────────────────

  @Public()
  @Post('login')
  @HttpCode(HttpStatus.OK)
  @ApiOperation({ summary: 'Autenticar usuário', description: 'Retorna access token (JWT 15min) e refresh token opaco (7 dias).' })
  @ApiOkResponse({ description: 'Login bem-sucedido.' })
  @ApiUnauthorizedResponse({ description: 'Credenciais inválidas.' })
  async login(@Body() dto: LoginDto, @Req() req: Request) {
    return this.loginUseCase.execute({
      email:     dto.email,
      password:  dto.password,
      userAgent: req.headers['user-agent'] ?? null,
      ipAddress: req.ip ?? null,
    });
  }

  // ─── POST /api/v1/auth/refresh ────────────────────────────────────────────

  @Public()
  @Post('refresh')
  @HttpCode(HttpStatus.OK)
  @ApiOperation({
    summary: 'Renovar access token',
    description: 'Troca um refresh token válido por um novo par access + refresh (rotação). O refresh token enviado é invalidado e um novo é emitido.',
  })
  @ApiOkResponse({ description: 'Tokens renovados com sucesso.' })
  @ApiUnauthorizedResponse({ description: 'Refresh token inválido, expirado, revogado ou já utilizado.' })
  async refresh(@Body() dto: RefreshTokenDto, @Req() req: Request) {
    return this.refreshTokenUseCase.execute({
      rawRefreshToken: dto.refreshToken,
      userAgent:       req.headers['user-agent'] ?? null,
      ipAddress:       req.ip ?? null,
    });
  }

  // ─── POST /api/v1/auth/logout ─────────────────────────────────────────────

  @ApiBearerAuth('access-token')
  @Post('logout')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Encerrar sessão atual' })
  async logout(
    @Body() dto: RefreshTokenDto,
    @CurrentUser() user: UserPayload,
  ) {
    await this.logoutUseCase.execute({
      user,
      rawRefreshToken: dto.refreshToken,
    });
  }

  // ─── POST /api/v1/auth/logout-all ─────────────────────────────────────────

  @ApiBearerAuth('access-token')
  @Post('logout-all')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Encerrar todas as sessões (todos os dispositivos)' })
  async logoutAll(@CurrentUser() user: UserPayload) {
    await this.logoutUseCase.execute({ user, allDevices: true });
  }
}
```

---

## 11. `AuthModule`

```typescript
// src/modules/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule } from '@nestjs/config';
import jwtConfig from '@config/jwt.config';

import { AuthController } from './controllers/auth.controller';
import { LoginUseCase } from './use-cases/login/login.use-case';
import { RefreshTokenUseCase } from './use-cases/refresh-token/refresh-token.use-case';
import { LogoutUseCase } from './use-cases/logout/logout.use-case';
import { RefreshTokenRepository } from './repositories/refresh-token.repository';
import { RefreshTokenEntity } from './entities/refresh-token.entity';
import { UsersModule } from '@modules/users/users.module';

@Module({
  imports: [
    // Registra a entity da tabela refresh_tokens
    TypeOrmModule.forFeature([RefreshTokenEntity]),

    // JwtModule sem configuração estática — access secret injetado por use case
    JwtModule.register({}),

    // Disponibiliza o namespace jwt para injeção via @Inject(jwtConfig.KEY)
    ConfigModule,

    // Importa UsersModule para acesso ao UsersRepository no LoginUseCase
    UsersModule,
  ],
  controllers: [AuthController],
  providers: [
    LoginUseCase,
    RefreshTokenUseCase,
    LogoutUseCase,
    RefreshTokenRepository,
  ],
  exports: [
    // Exporta somente o que outros módulos precisam (ex: guards)
    JwtModule,
  ],
})
export class AuthModule {}
```

---

## 12. Registro do Guard Global no `AppModule`

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule } from '@nestjs/config';

import { JwtAuthGuard } from '@common/guards/jwt-auth.guard';
import { AuthModule } from '@modules/auth/auth.module';
// ... outros imports

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [jwtConfig, /* ... */] }),
    DatabaseModule,
    AppCacheModule,  // ← Redis via @nestjs/cache-manager (ver query-optimization.md)
    AuthModule,
    UsersModule,
    // ... outros módulos de domínio
  ],
  providers: [
    // ← Guard global: toda rota exige JWT por padrão
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
    },
  ],
})
export class AppModule {}
```

> **Por que `APP_GUARD` e não `app.useGlobalGuards()`?** O `APP_GUARD` usa o container de DI do NestJS — permite injetar `CacheManager`, `Reflector` e `JwtService` no guard. O `useGlobalGuards()` não tem acesso ao DI container.

---

## 13. Migration — Tabela `refresh_tokens`

```typescript
// src/infra/database/migrations/<timestamp>-CreateRefreshTokensTable.ts
import { MigrationInterface, QueryRunner, Table, TableIndex, TableForeignKey } from 'typeorm';

export class CreateRefreshTokensTable<timestamp> implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'refresh_tokens',
        columns: [
          { name: 'id',          type: 'uuid',      isPrimary: true, generationStrategy: 'uuid', default: 'uuid_generate_v4()' },
          { name: 'user_id',     type: 'uuid',      isNullable: false },
          { name: 'token_hash',  type: 'varchar',   length: '64',  isNullable: false, isUnique: true },
          { name: 'family_id',   type: 'uuid',      isNullable: false },
          { name: 'is_revoked',  type: 'boolean',   default: false },
          { name: 'is_used',     type: 'boolean',   default: false },
          { name: 'expires_at',  type: 'timestamp', isNullable: false },
          { name: 'user_agent',  type: 'varchar',   length: '512', isNullable: true },
          { name: 'ip_address',  type: 'varchar',   length: '45',  isNullable: true },
          { name: 'created_at',  type: 'timestamp', default: 'now()' },
          { name: 'used_at',     type: 'timestamp', isNullable: true },
        ],
      }),
      true,
    );

    await queryRunner.createIndices('refresh_tokens', [
      new TableIndex({ name: 'IDX_refresh_tokens_user_id',   columnNames: ['user_id'] }),
      new TableIndex({ name: 'IDX_refresh_tokens_family_id', columnNames: ['family_id'] }),
    ]);

    await queryRunner.createForeignKey(
      'refresh_tokens',
      new TableForeignKey({
        columnNames:          ['user_id'],
        referencedTableName:  'users',
        referencedColumnNames: ['id'],
        onDelete: 'CASCADE',
      }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('refresh_tokens', true);
  }
}
```

---

## 14. Cron de Limpeza — Tokens Expirados

```typescript
// src/modules/auth/tasks/cleanup-expired-tokens.task.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { RefreshTokenRepository } from '../repositories/refresh-token.repository';

@Injectable()
export class CleanupExpiredTokensTask {
  private readonly logger = new Logger(CleanupExpiredTokensTask.name);

  constructor(private readonly refreshTokenRepository: RefreshTokenRepository) {}

  // Roda todo dia às 3h — remove tokens expirados do banco
  @Cron(CronExpression.EVERY_DAY_AT_3AM)
  async handle(): Promise<void> {
    this.logger.log('Limpando refresh tokens expirados...');
    await this.refreshTokenRepository.deleteExpired();
    this.logger.log('Limpeza concluída.');
  }
}
```

> Adicione `CleanupExpiredTokensTask` ao `providers` do `AuthModule` e importe `ScheduleModule.forRoot()` no `AppModule`.

---

## 15. Uso nos Controllers de Domínio

```typescript
// Qualquer controller protegido — não precisa de nenhum decorator extra
@Controller({ path: 'meals', version: '1' })
export class MealsController {

  @Get('history')
  async getHistory(@CurrentUser() user: UserPayload) {
    return this.getMealHistoryUseCase.execute({ userId: user.sub });
  }
}

// Rota pública (não requer token)
@Controller({ path: 'auth', version: '1' })
export class AuthController {

  @Public()          // ← torna a rota acessível sem JWT
  @Post('login')
  async login(@Body() dto: LoginDto) { ... }
}
```

---

## 16. Fluxo Completo — Diagrama

```
[Cliente Mobile]          [API NestJS]                   [Banco / Redis]
     │                        │                                │
     │── POST /auth/login ───►│                                │
     │                        │── findByEmail(email) ─────────►│
     │                        │◄── UserEntity ─────────────────│
     │                        │── bcrypt.compare(pass, hash)   │
     │                        │── signAsync(payload) → AT      │
     │                        │── randomUUID() → RT bruto      │
     │                        │── SHA256(RT) → hash            │
     │                        │── INSERT refresh_tokens ───────►│
     │◄── { AT, RT } ─────────│                                │
     │                        │                                │
     │── GET /meals (Bearer AT)►│                               │
     │                        │── verify(AT, secret) ✓         │
     │                        │── GET bl:<jti> ────────────────►│ (Redis)
     │                        │◄── null (não bloqueado) ────────│
     │◄── { data: meals } ─────│                                │
     │                        │                                │
     │── POST /auth/refresh ──►│                                │
     │     { refreshToken: RT }│── SHA256(RT) → hash            │
     │                        │── findByHash(hash) ────────────►│
     │                        │◄── RefreshTokenEntity ──────────│
     │                        │── isUsed? → false ✓             │
     │                        │── isRevoked? → false ✓          │
     │                        │── expiresAt > now? ✓            │
     │                        │── UPDATE is_used=true ──────────►│
     │                        │── signAsync → novo AT           │
     │                        │── randomUUID → novo RT bruto    │
     │                        │── INSERT novo refresh_token ────►│
     │◄── { novo AT, novo RT } │                                │
     │                        │                                │
     │── POST /auth/logout ───►│                                │
     │     { refreshToken: RT }│── TTL = exp - now              │
     │                        │── SET bl:<jti> true TTL ───────►│ (Redis)
     │                        │── SHA256(RT) → revokeById ─────►│
     │◄── 204 No Content ──────│                                │
```

---

## 17. Anti-Padrões Comuns

### ❌ Usar `@nestjs/passport` quando já existe guard manual

```typescript
// ❌ ERRADO — conflito com o guard global manual
@UseGuards(AuthGuard('jwt'))  // Passport: não use
async getProfile() { ... }

// ✅ CORRETO — o APP_GUARD já protege tudo globalmente
async getProfile(@CurrentUser() user: UserPayload) { ... }
```

### ❌ Armazenar refresh token bruto no banco

```typescript
// ❌ ERRADO — token bruto exposto se banco for comprometido
await this.repo.save({ userId, token: rawRefreshToken });

// ✅ CORRETO — apenas o hash SHA-256
const hash = RefreshTokenRepository.hash(rawRefreshToken);
await this.repo.create({ userId, tokenHash: hash, ... });
```

### ❌ Não fazer reuse detection

```typescript
// ❌ ERRADO — apenas verifica revogado, ignora uso duplo
if (token.isRevoked) throw new UnauthorizedException();

// ✅ CORRETO — verifica isUsed primeiro → revoga família
if (token.isUsed) {
  await this.repo.revokeFamily(token.familyId); // revoga toda a família
  throw new UnauthorizedException('Replay attack detectado.');
}
```

### ❌ Payload JWT com dados sensíveis

```typescript
// ❌ ERRADO — e-mail e nome no payload (Base64, não criptografado)
const payload = { sub: user.id, email: user.email, name: user.name, role: user.role };

// ✅ CORRETO — mínimo necessário
const payload: JwtSignPayload = { sub: user.id, role: user.role, jti };
```

### ❌ Access token com TTL longo

```typescript
// ❌ ERRADO — 24h → janela de ataque enorme
expiresIn: '24h'

// ✅ CORRETO — 15 minutos → dano mínimo se comprometido
expiresIn: '15m'
```

### ❌ Same secret para access e refresh token

```typescript
// ❌ ERRADO — se o secret vazar, ambos ficam comprometidos
const accessToken = await this.jwtService.signAsync(payload, { secret: JWT_SECRET });
const refreshJwt  = await this.jwtService.signAsync(payload, { secret: JWT_SECRET });

// ✅ CORRETO — secrets distintos (ou refresh token opaco, sem JWT)
// O refresh token neste projeto é UUID opaco — sem assinatura JWT
const rawRefreshToken = randomUUID(); // opaco, armazenado como hash no banco
```

---

## 18. Checklist de Implementação

### Configuração
- [ ] `JWT_ACCESS_SECRET` e `JWT_REFRESH_SECRET` são strings distintas com ≥ 32 caracteres?
- [ ] Ambas as variáveis definidas em `env.config.ts` com validação Zod?
- [ ] `jwt.config.ts` com `registerAs('jwt', ...)` configurado?
- [ ] `jwt.config.ts` listado no `load: [...]` do `ConfigModule.forRoot()`?

### Entity e Migration
- [ ] `RefreshTokenEntity` com os campos `tokenHash`, `familyId`, `isUsed`, `isRevoked`, `expiresAt`?
- [ ] Índice em `userId` e `familyId` na tabela `refresh_tokens`?
- [ ] Migration criada e testada (`migration:run` + `migration:revert`)?
- [ ] `TypeOrmModule.forFeature([RefreshTokenEntity])` no `AuthModule`?

### Guard e Decorators
- [ ] `JwtAuthGuard` registrado via `APP_GUARD` no `AppModule` (não por rota)?
- [ ] `@Public()` aplicado em `login`, `refresh` e outras rotas abertas?
- [ ] `@CurrentUser()` retorna `UserPayload` tipado corretamente?
- [ ] Redis (`CacheModule`) disponível globalmente para a blacklist?

### Use Cases
- [ ] `LoginUseCase`: bcrypt + delay constante mesmo quando usuário não existe?
- [ ] `LoginUseCase`: `jti` único (UUID v4) no payload do access token?
- [ ] `RefreshTokenUseCase`: verifica `isUsed` antes de `isRevoked`?
- [ ] `RefreshTokenUseCase`: revoga a família inteira no reuse detection?
- [ ] `RefreshTokenUseCase`: emite novo token com o mesmo `familyId`?
- [ ] `LogoutUseCase`: adiciona `jti` na blacklist Redis com TTL correto?
- [ ] `LogoutUseCase`: revoga o refresh token no banco?

### Segurança
- [ ] Payload JWT contém apenas `sub`, `role` e `jti` — sem dados sensíveis?
- [ ] Refresh token bruto nunca armazenado no banco — apenas hash SHA-256?
- [ ] Access token expira em ≤ 15 minutos?
- [ ] Refresh token expira em ≤ 7-14 dias?
- [ ] Cron de limpeza de tokens expirados configurado?

---

## Referências

- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — UseCase como camada de aplicação, Controllers finos
- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — `jwt.config.ts` com `registerAs`, acesso via `@Inject(jwtConfig.KEY)`
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura de pastas, convenção de Use Cases
- [rest-api-patterns.md](.agent/skills/backend/api-and-contracts/rest-api-patterns.md) — `APP_GUARD` vs `useGlobalGuards()`, status HTTP (204 logout)
- [dependency-injection.md](.agent/skills/backend/configuration-and-environment/dependency-injection.md) — `APP_GUARD` como token de injeção
- [query-optimization.md](.agent/skills/backend/database/query-optimization.md) — `AppCacheModule` com Redis (blacklist)
- [typeorm-patterns.md](.agent/skills/backend/database/typeorm-patterns.md) — Entity, Repository customizado, Migration
- [openapi-docs.md](.agent/skills/backend/api-and-contracts/openapi-docs.md) — `@ApiBearerAuth`, `@ApiTags`, `@ApiOperation`
- [NestJS Docs — Authentication](https://docs.nestjs.com/security/authentication)
- [NestJS Docs — Guards](https://docs.nestjs.com/guards)
- [NestJS Docs — Custom Decorators](https://docs.nestjs.com/custom-decorators)
- [OWASP — Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)