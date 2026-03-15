---
name: security-hardening
description: Segurança em produção para NestJS — Helmet para headers HTTP defensivos, rate limiting com @nestjs/throttler (global + por rota), proteção contra brute force em endpoints de autenticação, sanitização de inputs além do class-validator, CORS configurado por ambiente via env.config.ts, e checklist de segurança pré-deploy. Complementa rest-api-patterns.md (main.ts, ValidationPipe, HttpExceptionFilter), config-management.md (env.config.ts e variáveis de ambiente), dto-validation.md (class-validator como primeira linha de defesa), e nest-project-structure.md (src/common/ e src/config/).
---

# Segurança em Produção — NestJS Security Hardening

Você é um Arquiteto de Software Senior especialista em segurança de APIs. Esta skill define **como endurecer a segurança de uma aplicação NestJS** para produção. Ela complementa:

- `rest-api-patterns.md` — o `main.ts` onde os middlewares e guards são registrados globalmente
- `config-management.md` — `env.config.ts` com validação Zod; variáveis de ambiente por ambiente
- `dto-validation.md` — `ValidationPipe` global com `whitelist: true, forbidNonWhitelisted: true` como primeira linha de defesa
- `nest-project-structure.md` — convenções de pasta `src/common/` e `src/config/`

> **Regra fundamental:** Segurança em camadas (Defense in Depth). Cada camada cobre o que a anterior não consegue. A ordem de montagem no `main.ts` importa: Helmet → CORS → ThrottlerModule → ValidationPipe → Guards.

---

## 1. Instalação

```bash
npm install helmet @nestjs/throttler
```

> **Atenção:** `helmet` e `@nestjs/throttler` são as únicas dependências externas desta skill. O `class-validator` e `class-transformer` já estão cobertos pela skill `dto-validation.md`.

---

## 2. Variáveis de Ambiente — Atualizando `env.config.ts`

Adicione as variáveis de segurança ao schema Zod existente em `src/config/env.config.ts` (conforme o padrão da skill `config-management.md`):

```typescript
// src/config/env.config.ts — adicionar ao envSchema existente
const envSchema = z.object({
  // ... variáveis existentes (NODE_ENV, PORT, DATABASE_URL, JWT_SECRET, etc.)

  // ─── CORS ────────────────────────────────────────────────────────────────────
  // Lista de origens permitidas, separadas por vírgula
  // Ex: "http://localhost:3000,https://app.meudominio.com"
  CORS_ORIGINS: z.string().default('http://localhost:3000'),

  // ─── Rate Limiting ────────────────────────────────────────────────────────────
  // Limite padrão: requisições por janela (global)
  THROTTLE_TTL_SECONDS: z.coerce.number().int().positive().default(60),
  THROTTLE_LIMIT: z.coerce.number().int().positive().default(100),

  // Limite estrito: para endpoints sensíveis (auth, etc.)
  THROTTLE_AUTH_TTL_SECONDS: z.coerce.number().int().positive().default(60),
  THROTTLE_AUTH_LIMIT: z.coerce.number().int().positive().default(5),
});
```

Adicione ao namespace de configuração `app.config.ts` (ou crie `security.config.ts` se preferir separar):

```typescript
// src/config/security.config.ts
import { registerAs } from '@nestjs/config';
import { env } from './env.config';

export interface SecurityConfig {
  corsOrigins: string[];
  throttle: {
    ttlSeconds: number;
    limit: number;
  };
  throttleAuth: {
    ttlSeconds: number;
    limit: number;
  };
}

export default registerAs('security', (): SecurityConfig => ({
  corsOrigins: env.CORS_ORIGINS.split(',').map((origin) => origin.trim()),
  throttle: {
    ttlSeconds: env.THROTTLE_TTL_SECONDS,
    limit: env.THROTTLE_LIMIT,
  },
  throttleAuth: {
    ttlSeconds: env.THROTTLE_AUTH_TTL_SECONDS,
    limit: env.THROTTLE_AUTH_LIMIT,
  },
}));
```

Arquivos `.env.*` correspondentes:

```env
# .env.development
CORS_ORIGINS=http://localhost:3000,http://localhost:19006
THROTTLE_TTL_SECONDS=60
THROTTLE_LIMIT=200
THROTTLE_AUTH_TTL_SECONDS=60
THROTTLE_AUTH_LIMIT=20

# .env.staging
CORS_ORIGINS=https://staging.meuapp.com,https://staging-app.meuapp.com
THROTTLE_TTL_SECONDS=60
THROTTLE_LIMIT=120
THROTTLE_AUTH_TTL_SECONDS=60
THROTTLE_AUTH_LIMIT=10

# .env.production
CORS_ORIGINS=https://meuapp.com,https://app.meuapp.com
THROTTLE_TTL_SECONDS=60
THROTTLE_LIMIT=100
THROTTLE_AUTH_TTL_SECONDS=60
THROTTLE_AUTH_LIMIT=5
```

---

## 3. Helmet — Headers HTTP Defensivos

O Helmet configura headers HTTP que protegem contra vetores de ataque comuns. Para APIs JSON (sem renderização de HTML), use uma configuração enxuta — CSP é desnecessário e pode causar conflitos.

### 3.1 Configuração centralizada — `src/config/helmet.config.ts`

```typescript
// src/config/helmet.config.ts
import type { HelmetOptions } from 'helmet';
import { env } from './env.config';

/**
 * Configuração do Helmet para APIs JSON.
 *
 * NOTA IMPORTANTE para APIs puras (sem SSR/HTML):
 * - contentSecurityPolicy: desabilitado — irrelevante para JSON APIs e pode
 *   conflitar com o Swagger UI. Se servir HTML, habilite e configure as diretivas.
 * - crossOriginEmbedderPolicy: desabilitado — relevante apenas para apps web
 *   que usam SharedArrayBuffer ou recursos cross-origin com COEP.
 *
 * Para cada header desabilitado, documente o motivo aqui para facilitar
 * auditorias de segurança e onboarding de novos membros do time.
 */
export const helmetConfig: HelmetOptions = {
  // ─── Proteções sempre ativas ─────────────────────────────────────────────────

  // X-Frame-Options: DENY — impede clickjacking via <iframe>
  frameguard: { action: 'deny' },

  // X-Content-Type-Options: nosniff — impede MIME sniffing no navegador
  noSniff: true,

  // Oculta o header "X-Powered-By: Express"
  hidePoweredBy: true,

  // DNS Prefetch Control: off — impede pré-busca de DNS de terceiros
  dnsPrefetchControl: { allow: false },

  // Referrer-Policy: no-referrer — não vaza a URL de origem em requests
  referrerPolicy: { policy: 'no-referrer' },

  // X-Download-Options: noopen — impede abertura direta de arquivos no IE
  ieNoOpen: true,

  // Cross-Origin-Resource-Policy — controla quem pode carregar os recursos
  crossOriginResourcePolicy: { policy: 'same-origin' },

  // Permissions-Policy — desativa APIs de hardware desnecessárias
  permittedCrossDomainPolicies: { permittedPolicies: 'none' },

  // ─── HSTS — Strict-Transport-Security ────────────────────────────────────────
  // Apenas em produção: força HTTPS por 1 ano, incluindo subdomínios
  // Não habilitar em dev/staging para não bloquear testes locais
  hsts: env.NODE_ENV === 'production'
    ? {
        maxAge: 31_536_000,      // 1 ano em segundos
        includeSubDomains: true,
        preload: true,           // Elegível para preload list do navegador
      }
    : false,

  // ─── Headers desabilitados intencionalmente ──────────────────────────────────

  // Content-Security-Policy: desabilitado — esta é uma API JSON pura.
  // Se adicionar renderização de HTML (Swagger em produção, SSR, etc.),
  // habilite e configure as diretivas adequadas.
  contentSecurityPolicy: false,

  // Cross-Origin-Embedder-Policy: desabilitado — não usamos SharedArrayBuffer
  // nem recursos cross-origin com COEP. Habilitar quebraria carregamento de
  // recursos de terceiros (fontes, imagens) sem o header CORP correto.
  crossOriginEmbedderPolicy: false,
};
```

### 3.2 Tabela de Headers Configurados

| Header | Valor | Protege contra |
|---|---|---|
| `X-Frame-Options` | `DENY` | Clickjacking via `<iframe>` |
| `X-Content-Type-Options` | `nosniff` | MIME sniffing / XSS via upload |
| `X-Powered-By` | removido | Fingerprinting do servidor |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` (prod) | Downgrade HTTPS → HTTP |
| `Referrer-Policy` | `no-referrer` | Vazamento de URL em requests |
| `Cross-Origin-Resource-Policy` | `same-origin` | Uso cross-origin não autorizado |
| `DNS-Prefetch-Control` | `off` | Pré-busca DNS de terceiros |
| `Permissions-Policy` | `none` | APIs de câmera, microfone, geolocalização |

### 3.3 Quando habilitar o `contentSecurityPolicy`

Se sua aplicação servir **HTML** (por exemplo, Swagger em produção ou Server-Side Rendering), habilite o CSP com as diretivas adequadas:

```typescript
// Exemplo para aplicação que serve Swagger em produção
contentSecurityPolicy: {
  directives: {
    defaultSrc:  ["'self'"],
    scriptSrc:   ["'self'"],
    styleSrc:    ["'self'", "'unsafe-inline'"],  // Swagger precisa de inline styles
    imgSrc:      ["'self'", 'data:', 'https:'],
    connectSrc:  ["'self'"],
    fontSrc:     ["'self'"],
    objectSrc:   ["'none'"],
    frameAncestors: ["'none'"],
  },
},
```

> **Regra:** Nunca use `'unsafe-eval'` em `scriptSrc`. Prefira nonces ou hashes para scripts inline se necessário.

---

## 4. CORS — Cross-Origin Resource Sharing

### 4.1 Configuração centralizada — `src/config/cors.config.ts`

```typescript
// src/config/cors.config.ts
import type { CorsOptions } from '@nestjs/common/interfaces/external/cors-options.interface';
import { ConfigType } from '@nestjs/config';
import { env } from './env.config';
import securityConfig from './security.config';

/**
 * Fábrica de configuração do CORS.
 * Chame esta função passando a instância do securityConfig
 * para injeção de dependência no AppModule.
 */
export function buildCorsOptions(
  config: ConfigType<typeof securityConfig>,
): CorsOptions {
  const allowedOrigins = config.corsOrigins;

  return {
    // ← Valida a origem dinamicamente; permite null para requests sem Origin (ex: mobile)
    origin: (origin, callback) => {
      // Permite requests sem Origin (ex: apps mobile nativos, Postman, curl)
      if (!origin) return callback(null, true);

      if (allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error(`CORS: Origin "${origin}" não autorizada.`));
      }
    },

    // Métodos HTTP permitidos
    methods: ['GET', 'HEAD', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],

    // Headers permitidos nas requisições
    allowedHeaders: [
      'Content-Type',
      'Authorization',
      'Accept',
      'X-Requested-With',
    ],

    // Headers expostos para o cliente (para uso do TransformInterceptor)
    exposedHeaders: ['X-Total-Count'],

    // Permite cookies cross-origin (necessário para sessões ou cookies de autenticação)
    // Defina como false se a API usa exclusivamente JWT em headers
    credentials: false,

    // Cache da resposta preflight em segundos
    maxAge: 86_400, // 24 horas
  };
}
```

### 4.2 Tabela de decisão de CORS por ambiente

| Situação | `credentials` | `origin` | Observação |
|---|---|---|---|
| API consumida por app mobile (JWT em header) | `false` | lista explícita + `null` | App mobile não tem `Origin` |
| API consumida por SPA (cookies) | `true` | lista explícita | Nunca `*` com `credentials: true` |
| API pública (sem autenticação) | `false` | `*` ou lista | Apenas para endpoints verdadeiramente públicos |
| Development local | `false` | `localhost:*` | Configure via `.env.development` |

> **Regra:** **Nunca use `origin: '*'` com `credentials: true`.** O navegador bloqueará e isso não é configuração válida pelo padrão CORS. Use sempre uma lista explícita de origens em produção.

---

## 5. Rate Limiting — `@nestjs/throttler`

### 5.1 Configuração do `ThrottlerModule` no `AppModule`

Registre o `ThrottlerModule` no `AppModule` com múltiplos perfis (tiers) e `forRootAsync` para consumir as variáveis via `ConfigService`:

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigType } from '@nestjs/config';
import { APP_GUARD } from '@nestjs/core';
import { ThrottlerModule, ThrottlerGuard, seconds } from '@nestjs/throttler';

import { DatabaseModule } from '@infra/database/database.module';
import { AuthModule } from '@modules/auth/auth.module';
import { MealsModule } from '@modules/meals/meals.module';
import { UsersModule } from '@modules/users/users.module';
import securityConfig from '@config/security.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [securityConfig, /* demais configs */],
    }),

    DatabaseModule,

    // ─── Rate Limiting Global ─────────────────────────────────────────────────
    ThrottlerModule.forRootAsync({
      inject: [securityConfig.KEY],
      useFactory: (config: ConfigType<typeof securityConfig>) => ({
        throttlers: [
          {
            name: 'default',
            ttl: seconds(config.throttle.ttlSeconds),
            limit: config.throttle.limit,
          },
          // Tier "burst": até 10 req/s para picos legítimos de curta duração
          {
            name: 'burst',
            ttl: seconds(1),
            limit: 10,
          },
        ],
        // Mensagem padronizada do erro 429 (integra com HttpExceptionFilter)
        errorMessage: 'Too many requests. Please try again later.',
      }),
    }),

    // ─── Módulos de domínio ───────────────────────────────────────────────────
    AuthModule,
    UsersModule,
    MealsModule,
  ],

  providers: [
    // ← ThrottlerGuard global: aplica os limites em TODOS os endpoints
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}
```

### 5.2 Proteção contra Brute Force em Auth — `@Throttle()` por rota

Endpoints de autenticação precisam de limites muito mais restritivos. Use o decorator `@Throttle()` para sobrescrever o padrão global:

```typescript
// src/modules/auth/controllers/auth.controller.ts
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { Throttle, SkipThrottle } from '@nestjs/throttler';
import { ApiTags, ApiOperation, ApiCreatedResponse, ApiBadRequestResponse, ApiUnauthorizedResponse } from '@nestjs/swagger';

import { LoginUseCase } from '../use-cases/login/login.use-case';
import { RegisterUseCase } from '../use-cases/register/register.use-case';
import { RefreshTokenUseCase } from '../use-cases/refresh-token/refresh-token.use-case';
import { LoginDto } from '../dtos/login.dto';
import { RegisterDto } from '../dtos/register.dto';
import { RefreshTokenDto } from '../dtos/refresh-token.dto';
import { AuthViewModel } from '../view-models/auth.view-model';

@ApiTags('auth')
@Controller({ path: 'auth', version: '1' })
export class AuthController {
  constructor(
    private readonly loginUseCase: LoginUseCase,
    private readonly registerUseCase: RegisterUseCase,
    private readonly refreshTokenUseCase: RefreshTokenUseCase,
  ) {}

  /**
   * POST /api/v1/auth/login
   * Limite estrito: 5 tentativas por minuto por IP (proteção brute force).
   * Após exceder, o cliente recebe 429 por todo o TTL.
   */
  @Post('login')
  @HttpCode(HttpStatus.OK)
  // ← Sobrescreve o guard global com limite muito mais restritivo
  @Throttle({ default: { ttl: 60_000, limit: 5 } })
  @ApiOperation({ summary: 'Autenticar usuário com e-mail e senha' })
  @ApiCreatedResponse({ type: AuthViewModel })
  @ApiUnauthorizedResponse({ description: 'Credenciais inválidas.' })
  @ApiBadRequestResponse({ description: 'Payload inválido.' })
  async login(@Body() dto: LoginDto): Promise<AuthViewModel> {
    return this.loginUseCase.execute(dto);
  }

  /**
   * POST /api/v1/auth/register
   * Limite moderado: 3 registros por minuto por IP (evita criação em massa).
   */
  @Post('register')
  @HttpCode(HttpStatus.CREATED)
  @Throttle({ default: { ttl: 60_000, limit: 3 } })
  @ApiOperation({ summary: 'Registrar novo usuário' })
  async register(@Body() dto: RegisterDto): Promise<AuthViewModel> {
    return this.registerUseCase.execute(dto);
  }

  /**
   * POST /api/v1/auth/refresh
   * Limite moderado: refresh token pode ser chamado com mais frequência,
   * mas ainda tem proteção contra abuso.
   */
  @Post('refresh')
  @HttpCode(HttpStatus.OK)
  @Throttle({ default: { ttl: 60_000, limit: 10 } })
  @ApiOperation({ summary: 'Renovar access token via refresh token' })
  async refreshToken(@Body() dto: RefreshTokenDto): Promise<AuthViewModel> {
    return this.refreshTokenUseCase.execute(dto);
  }
}
```

### 5.3 Tabela de Limites por Tipo de Endpoint

| Tipo de Endpoint | TTL | Limite | Justificativa |
|---|---|---|---|
| Global (padrão) | 60s | 100 req | Tráfego legítimo normal |
| Burst (curto prazo) | 1s | 10 req | Picos legítimos sem flooding |
| `POST /auth/login` | 60s | 5 req | Brute force de credenciais |
| `POST /auth/register` | 60s | 3 req | Criação massiva de contas |
| `POST /auth/refresh` | 60s | 10 req | Renovação frequente legítima |
| `POST /auth/forgot-password` | 60s | 3 req | Enumeração de e-mails |
| Health check / Metrics | — | `@SkipThrottle()` | Monitoramento externo |

### 5.4 `@SkipThrottle()` — Endpoints Isentos

```typescript
// Endpoints de infraestrutura não devem ser bloqueados por rate limiting
@Controller({ path: 'health', version: VERSION_NEUTRAL })
@SkipThrottle() // ← isenta todo o controller
export class HealthController {
  @Get()
  check() { return { status: 'ok' }; }
}
```

### 5.5 Guard Customizado — Tracking por Usuário Autenticado

Para ambientes que precisam diferenciar usuários autenticados (limites maiores) de anônimos:

```typescript
// src/common/guards/throttler-by-user.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { ThrottlerGuard } from '@nestjs/throttler';
import { Request } from 'express';

/**
 * Throttler customizado que usa userId (quando disponível) ou IP como identificador.
 *
 * Benefício: usuários autenticados não são bloqueados pelo mesmo bucket de IP
 * que um atacante esteja abusando na mesma rede (ex: NAT compartilhado).
 */
@Injectable()
export class ThrottlerByUserGuard extends ThrottlerGuard {
  protected async getTracker(req: Request): Promise<string> {
    // Se o usuário estiver autenticado, usa o userId como chave de tracking
    const userId = (req as any).user?.id;
    if (userId) return `user_${userId}`;

    // Fallback: IP do cliente
    // Lida com proxy reverso — lê X-Forwarded-For primeiro
    const forwarded = req.headers['x-forwarded-for'];
    if (forwarded) {
      const ip = Array.isArray(forwarded)
        ? forwarded[0]
        : forwarded.split(',')[0].trim();
      return `ip_${ip}`;
    }

    return `ip_${req.ip ?? 'unknown'}`;
  }
}
```

> **Atenção com `X-Forwarded-For`:** Em produção atrás de um proxy reverso (Nginx, Traefik, AWS ALB), configure `app.set('trust proxy', 1)` no Express ou verifique se o proxy está configurado para enviar o IP real do cliente no header `X-Forwarded-For`. Sem isso, `req.ip` retornará o IP do proxy, e todos os clientes compartilharão o mesmo bucket de rate limiting.

```typescript
// src/main.ts — adicionar ANTES de app.enableCors()
// Se a aplicação roda atrás de 1 nível de proxy reverso
(app.getHttpAdapter().getInstance() as any).set('trust proxy', 1);
```

---

## 6. Sanitização de Inputs

A `dto-validation.md` já cobre validação de formato via `class-validator`. Esta seção aborda **sanitização** — a remoção ou neutralização de conteúdo potencialmente malicioso que passou na validação.

### 6.1 Estratégia de Sanitização por Camada

| Camada | O que faz | Ferramenta |
|---|---|---|
| **DTO (Apresentação)** | Valida formato, tipo, tamanho | `class-validator` (já coberto em `dto-validation.md`) |
| **DTO com Transform** | Normaliza (trim, lowercase, strip tags) | `class-transformer` `@Transform()` |
| **UseCase (Aplicação)** | Valida regras de negócio | Lógica explícita |
| **ORM (Infraestrutura)** | Previne SQL injection via parametrização | TypeORM/Prisma (parametrização automática) |

### 6.2 Sanitização em DTOs com `@Transform()`

```typescript
// src/modules/users/dtos/create-user.dto.ts
import { IsEmail, IsString, MaxLength, MinLength } from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  // ← Normaliza e-mail para lowercase e remove espaços antes de validar
  @Transform(({ value }) => typeof value === 'string'
    ? value.toLowerCase().trim()
    : value)
  @IsEmail({}, { message: 'E-mail deve ser um endereço válido.' })
  email: string;

  // ← Remove espaços extras antes e depois do nome
  @Transform(({ value }) => typeof value === 'string' ? value.trim() : value)
  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;

  @IsString()
  @MinLength(8, { message: 'Senha deve ter no mínimo 8 caracteres.' })
  // ← NÃO faça trim em senhas — espaços podem ser intencionais
  password: string;
}
```

### 6.3 Pipe Global de Sanitização de Strings

Para remover espaços extras de **todas** as strings da aplicação de forma centralizada:

```typescript
// src/common/pipes/sanitize-string.pipe.ts
import { Injectable, PipeTransform, ArgumentMetadata } from '@nestjs/common';

/**
 * Pipe global que aplica trim em todas as strings do payload.
 *
 * ESCOPO: remove espaços no início/fim de strings.
 * NÃO escapa HTML nem remove caracteres — isso é responsabilidade do DTO
 * ou do uso específico (ex: campos de texto livre que aceitam HTML controlado).
 *
 * Registro: app.useGlobalPipes(new SanitizeStringPipe()) NO main.ts,
 * ANTES do ValidationPipe, para que o trim aconteça antes da validação.
 */
@Injectable()
export class SanitizeStringPipe implements PipeTransform {
  transform(value: unknown, _metadata: ArgumentMetadata): unknown {
    return this.sanitizeValue(value);
  }

  private sanitizeValue(value: unknown): unknown {
    if (typeof value === 'string') {
      return value.trim();
    }
    if (Array.isArray(value)) {
      return value.map((item) => this.sanitizeValue(item));
    }
    if (value !== null && typeof value === 'object') {
      return Object.fromEntries(
        Object.entries(value as Record<string, unknown>).map(([k, v]) => [
          k,
          this.sanitizeValue(v),
        ]),
      );
    }
    return value;
  }
}
```

### 6.4 Proteção contra NoSQL Injection (MongoDB)

Se o projeto usar MongoDB, sanitize queries dinâmicas para evitar injeção de operadores Mongo (`$where`, `$gt`, etc.):

```typescript
// src/common/interceptors/mongo-sanitize.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { Request } from 'express';

/**
 * Interceptor que remove operadores MongoDB ($) de inputs do usuário.
 * Use apenas se o projeto utiliza MongoDB/Mongoose.
 */
@Injectable()
export class MongoSanitizeInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest<Request>();
    this.sanitize(request.body);
    this.sanitize(request.query);
    this.sanitize(request.params);
    return next.handle();
  }

  private sanitize(obj: unknown): void {
    if (obj && typeof obj === 'object') {
      for (const key of Object.keys(obj as Record<string, unknown>)) {
        if (key.startsWith('$')) {
          delete (obj as Record<string, unknown>)[key];
        } else {
          this.sanitize((obj as Record<string, unknown>)[key]);
        }
      }
    }
  }
}
```

> **Para projetos com PostgreSQL/TypeORM ou Prisma:** A injeção SQL é prevenida automaticamente pelos ORMs via parametrização. Nunca concatene inputs do usuário em strings SQL cruas — use sempre o QueryBuilder com parâmetros (`:param`) ou `$queryRaw` com template literals tipados.

---

## 7. Montagem Completa no `main.ts`

A ordem de registro dos middlewares e pipes no `main.ts` é determinante para a segurança. A sequência correta:

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { VersioningType, ValidationPipe } from '@nestjs/common';
import { ConfigType } from '@nestjs/config';
import helmet from 'helmet';
import { AppModule } from './app.module';
import { TransformInterceptor } from '@common/interceptors/transform.interceptor';
import { HttpExceptionFilter } from '@common/filters/http-exception.filter';
import { SanitizeStringPipe } from '@common/pipes/sanitize-string.pipe';
import { setupSwagger } from '@config/swagger.config';
import { helmetConfig } from '@config/helmet.config';
import { buildCorsOptions } from '@config/cors.config';
import securityConfig from '@config/security.config';
import { env } from '@config/env.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // ─── 1. Prefixo e Versionamento ──────────────────────────────────────────────
  app.setGlobalPrefix('api');
  app.enableVersioning({ type: VersioningType.URI });

  // ─── 2. Trust Proxy (se atrás de proxy reverso) ──────────────────────────────
  // NECESSÁRIO para que o ThrottlerGuard receba o IP real do cliente via
  // X-Forwarded-For, em vez do IP do proxy reverso.
  // Ajuste o número conforme a quantidade de proxies na frente da aplicação.
  if (env.NODE_ENV !== 'development') {
    (app.getHttpAdapter().getInstance() as any).set('trust proxy', 1);
  }

  // ─── 3. Helmet — Headers de Segurança ────────────────────────────────────────
  // Deve ser aplicado ANTES do CORS para que os headers de segurança sejam
  // definidos antes dos headers CORS.
  app.use(helmet(helmetConfig));

  // ─── 4. CORS ─────────────────────────────────────────────────────────────────
  const secConfig = app.get<ConfigType<typeof securityConfig>>(securityConfig.KEY);
  app.enableCors(buildCorsOptions(secConfig));

  // ─── 5. Pipes Globais ─────────────────────────────────────────────────────────
  // Ordem: SanitizeStringPipe (trim) → ValidationPipe (validação)
  // O trim acontece antes da validação para que MinLength não conte espaços.
  app.useGlobalPipes(
    new SanitizeStringPipe(),
    new ValidationPipe({
      whitelist: true,              // Remove campos não declarados no DTO
      forbidNonWhitelisted: true,   // Retorna 400 para campos desconhecidos
      transform: true,              // Converte payload para instância da classe
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );

  // ─── 6. Interceptors e Filters Globais ───────────────────────────────────────
  app.useGlobalInterceptors(new TransformInterceptor());
  app.useGlobalFilters(new HttpExceptionFilter());

  // ─── 7. Swagger — Apenas fora de produção ────────────────────────────────────
  if (env.NODE_ENV !== 'production') {
    setupSwagger(app);
  }

  // ─── 8. Shutdown Hooks ───────────────────────────────────────────────────────
  app.enableShutdownHooks();

  await app.listen(env.PORT);
  console.log(`🚀 API rodando em http://localhost:${env.PORT}/api/v1`);

  if (env.NODE_ENV !== 'production') {
    console.log(`📖 Swagger em http://localhost:${env.PORT}/api/docs`);
  }
}

bootstrap();
```

---

## 8. Resposta Padronizada para 429

O `ThrottlerGuard` lança uma `ThrottlerException` que é capturada pelo `HttpExceptionFilter` global (já definido em `rest-api-patterns.md`). O formato de erro é automaticamente padronizado:

```json
// Resposta 429 — capturada e formatada pelo HttpExceptionFilter
{
  "error": {
    "code": "TOO_MANY_REQUESTS",
    "message": "Too many requests. Please try again later.",
    "path": "/api/v1/auth/login",
    "timestamp": "2024-01-15T10:00:00.000Z"
  }
}
```

Para incluir o código `429` no mapeamento do `HttpExceptionFilter`:

```typescript
// src/common/filters/http-exception.filter.ts — verificar se 429 está mapeado
private static statusToCode(status: number): string {
  const map: Record<number, string> = {
    400: 'BAD_REQUEST',
    401: 'UNAUTHORIZED',
    403: 'FORBIDDEN',
    404: 'NOT_FOUND',
    409: 'CONFLICT',
    422: 'UNPROCESSABLE_ENTITY',
    429: 'TOO_MANY_REQUESTS', // ← garanta que este código existe
    500: 'INTERNAL_SERVER_ERROR',
  };
  return map[status] ?? `HTTP_${status}`;
}
```

---

## 9. Estrutura de Arquivos

```text
src/
├── config/
│   ├── env.config.ts          ← Adicionar variáveis de segurança ao schema Zod
│   ├── security.config.ts     ← registerAs('security', ...) — CORS e throttle
│   ├── helmet.config.ts       ← helmetConfig: HelmetOptions
│   ├── cors.config.ts         ← buildCorsOptions(config): CorsOptions
│   └── swagger.config.ts      ← Existente — não modificado
│
├── common/
│   ├── guards/
│   │   └── throttler-by-user.guard.ts   ← Guard customizado (tracking por userId)
│   ├── pipes/
│   │   └── sanitize-string.pipe.ts      ← Trim global de strings
│   ├── interceptors/
│   │   └── mongo-sanitize.interceptor.ts  ← Apenas se usar MongoDB
│   └── filters/
│       └── http-exception.filter.ts     ← Existente — verificar mapeamento 429
│
└── main.ts                    ← Ponto de registro de tudo (ordem importa!)
```

---

## 10. Anti-Padrões Comuns

### ❌ CORS com `origin: '*'` em produção

```typescript
// ❌ ERRADO — qualquer origem pode acessar a API
app.enableCors({ origin: '*' });

// ✅ CORRETO — lista explícita por ambiente
app.enableCors(buildCorsOptions(secConfig));
```

### ❌ `origin: '*'` com `credentials: true`

```typescript
// ❌ INVÁLIDO — navegador rejeita; além de inseguro
app.enableCors({ origin: '*', credentials: true });

// ✅ CORRETO — lista de origens com credentials
app.enableCors({ origin: ['https://meuapp.com'], credentials: true });
```

### ❌ Helmet sem configuração — em projetos que servem Swagger em produção

```typescript
// ❌ ARRISCADO — CSP padrão do Helmet pode bloquear o Swagger UI
app.use(helmet());

// ✅ CORRETO — desabilitar CSP para APIs puras, ou configurar se serve HTML
app.use(helmet(helmetConfig)); // helmetConfig define contentSecurityPolicy: false
```

### ❌ Rate limiting sem considerar proxy reverso

```typescript
// ❌ ERRADO — todos os usuários têm o mesmo IP (IP do proxy)
// Rate limiting inútil ou bloqueia usuários legítimos

// ✅ CORRETO — configurar trust proxy para obter IP real
(app.getHttpAdapter().getInstance() as any).set('trust proxy', 1);
```

### ❌ Registrar `SanitizeStringPipe` depois do `ValidationPipe`

```typescript
// ❌ ERRADO — validação acontece antes do trim; "  test  " falha em MinLength(5)
app.useGlobalPipes(new ValidationPipe(), new SanitizeStringPipe());

// ✅ CORRETO — trim antes, validação depois
app.useGlobalPipes(new SanitizeStringPipe(), new ValidationPipe({ ... }));
```

### ❌ `@SkipThrottle()` em endpoints sensíveis

```typescript
// ❌ NUNCA — endpoints de auth sem proteção de rate limiting
@Post('login')
@SkipThrottle()
async login() { ... }

// ✅ CORRETO — @Throttle() com limite estrito para brute force
@Post('login')
@Throttle({ default: { ttl: 60_000, limit: 5 } })
async login() { ... }
```

### ❌ Secrets expostos em erros de servidor

```typescript
// ❌ ERRADO — stack trace vaza detalhes internos em produção
throw new InternalServerErrorException(error.message); // pode conter query SQL

// ✅ CORRETO — mensagem genérica em produção; o HttpExceptionFilter registra o erro
// O filter já faz log em >= 500. No UseCase, apenas:
throw new InternalServerErrorException('Erro interno. Tente novamente.');
```

---

## 11. Checklist de Segurança Pré-Deploy

### Helmet e Headers

- [ ] `helmet()` aplicado **antes** do `enableCors()` no `main.ts`?
- [ ] HSTS configurado com `maxAge >= 31536000` apenas em produção?
- [ ] `contentSecurityPolicy: false` documentado com justificativa no `helmet.config.ts`?
- [ ] Verificar resposta HTTP com `curl -I` — presença de `X-Powered-By` é vazamento?

### CORS

- [ ] `CORS_ORIGINS` definido explicitamente nos arquivos `.env.*` de cada ambiente?
- [ ] `origin: '*'` **nunca** usado em staging ou produção?
- [ ] `credentials: true` somente quando necessário (fluxo com cookies)?
- [ ] Testar CORS com origin não listada → deve retornar erro de CORS, não 200?

### Rate Limiting

- [ ] `ThrottlerModule` registrado no `AppModule` com `forRootAsync`?
- [ ] `ThrottlerGuard` registrado como `APP_GUARD` global?
- [ ] `POST /auth/login` com `@Throttle()` de no máximo 5 req/min?
- [ ] `POST /auth/register` com `@Throttle()` de no máximo 3 req/min?
- [ ] `POST /auth/forgot-password` (se existir) com limite restritivo?
- [ ] Health check com `@SkipThrottle()`?
- [ ] `trust proxy` configurado se atrás de proxy reverso (Nginx, AWS ALB, etc.)?
- [ ] Resposta 429 mapeada no `HttpExceptionFilter` como `'TOO_MANY_REQUESTS'`?

### Sanitização

- [ ] `SanitizeStringPipe` registrado **antes** do `ValidationPipe`?
- [ ] Campos de e-mail normalizados para lowercase via `@Transform()`?
- [ ] Senhas **não** submetidas a trim (espaços podem ser intencionais)?
- [ ] Campos que recebem HTML livre sanitizados com biblioteca especializada (DOMPurify, sanitize-html)?
- [ ] Queries ao banco **sempre** via ORM (TypeORM/Prisma) com parametrização — nunca concatenação de strings?
- [ ] `MongoSanitizeInterceptor` registrado se o projeto usa MongoDB?

### Variáveis de Ambiente e Segredos

- [ ] Nenhum segredo (JWT_SECRET, API keys, senhas) hardcoded no código?
- [ ] `env.config.ts` com `JWT_SECRET: z.string().min(32)` para garantir chave segura?
- [ ] `.env.production` **nunca** commitado no repositório?
- [ ] `.env.example` commitado com todas as variáveis (sem valores sensíveis)?
- [ ] Verificar `.gitignore` — `.env`, `.env.*.local`, `.env.production` estão listados?

### Dependências

- [ ] `npm audit --audit-level=high` sem vulnerabilidades HIGH ou CRITICAL?
- [ ] `npm outdated` revisado — pacotes de segurança (helmet, throttler, passport) na última minor?
- [ ] `npm audit` configurado no pipeline de CI (ver skill `ci-cd-pipeline.md` do frontend)?

### Configuração de Produção

- [ ] `NODE_ENV=production` definido no servidor/container?
- [ ] Swagger desabilitado em produção (`env.NODE_ENV !== 'production'`)?
- [ ] Logs de `console.log/debug` removidos ou desabilitados em produção?
- [ ] `synchronize: false` no TypeORM para ambientes não-locais?
- [ ] HTTPS configurado no proxy reverso (Nginx, Traefik, AWS ALB)?

---

## Referências

- [rest-api-patterns.md](../api-and-contracts/rest-api-patterns.md) — `main.ts` com pipes, interceptors e filters globais
- [config-management.md](../configuration-and-environment/config-management.md) — `registerAs`, `env.config.ts` com Zod, `ConfigService`
- [dto-validation.md](../api-and-contracts/dto-validation.md) — `ValidationPipe`, `class-validator`, `@Transform()`
- [nest-project-structure.md](../foundation-and-architecture/nest-project-structure.md) — `src/common/`, `src/config/`, aliases `@common/` e `@config/`
- [NestJS Docs — Helmet](https://docs.nestjs.com/security/helmet)
- [NestJS Docs — CORS](https://docs.nestjs.com/security/cors)
- [NestJS Docs — Rate Limiting](https://docs.nestjs.com/security/rate-limiting)
- [Helmet.js — Reference](https://helmetjs.github.io/)
- [OWASP Node.js Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html)