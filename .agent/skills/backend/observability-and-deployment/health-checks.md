---
name: health-checks
description: Health checks com @nestjs/terminus no NestJS — custom indicators para Prisma e Redis via ioredis, endpoints /health/live (liveness) vs /health/ready (readiness), integração com Kubernetes probes (liveness, readiness, startup), graceful shutdown, e exclusão de logs e rate limiting para rotas de saúde. Use quando configurar health checks do zero, integrar com Kubernetes ou load balancers, ou adicionar verificação de dependências externas.
---

# Health Checks — @nestjs/terminus no NestJS

Você é um Engenheiro de Software Senior especialista em observabilidade e operações de sistemas NestJS em produção. Sua responsabilidade é garantir que o projeto tenha **health checks corretos, seguros e integrados com a infraestrutura de orquestração** (Kubernetes, ECS, load balancers), distinguindo claramente entre liveness e readiness.

> **Pré-requisitos:** Esta skill assume que as skills `config-management`, `nest-project-structure`, `logging-pattern`, `security-hardening`, `prisma-patterns` (ou `typeorm-patterns`) e `caching-strategy` já foram aplicadas. O `env.config.ts`, `PrismaService`, e `AppCacheModule` com `ioredis` são as fontes de verdade para conexões com banco e Redis.

---

## Quando usar esta skill

- **Novo projeto:** Configurar health checks do zero antes do primeiro deploy.
- **Kubernetes / ECS:** Definir liveness, readiness e startup probes.
- **Load balancer:** Criar endpoint de saúde para ALB, Nginx, Traefik ou similar.
- **Dependências externas:** Adicionar check de banco, Redis ou APIs de terceiros.
- **Graceful shutdown:** Garantir que o pod não perde conexões durante rollout.

---

## 1. Conceitos: Liveness vs Readiness vs Startup

| Probe | Pergunta | Ação em falha | O que verificar |
|---|---|---|---|
| **Liveness** (`/health/live`) | O processo está vivo? | Kubernetes reinicia o container | Apenas se o processo responde — **sem dependências externas** |
| **Readiness** (`/health/ready`) | Pode receber tráfego? | Remove do load balancer (não reinicia) | Banco, Redis, dependências externas |
| **Startup** (`/health/live`) | Terminou de inicializar? | Kubernetes aguarda (failureThreshold × period) | Mesmo que liveness — usado só para apps com start lento |

> **Regra de ouro:** Liveness simples e rápido. Readiness completo e com todas as dependências. Nunca coloque checks de banco no liveness — uma indisponibilidade temporária do banco causaria reinício desnecessário do pod.

---

## 2. Instalação

```bash
npm install @nestjs/terminus
npm install @nestjs/axios axios   # necessário para HttpHealthIndicator
```

> `ioredis` já está instalado via `caching-strategy.md` — **não instale novamente**. O custom Redis indicator usa a instância existente.

---

## 3. Variáveis de Ambiente

Complementa `config-management.md` — nenhuma variável nova é necessária. O módulo usa as conexões já configuradas (`DATABASE_URL`, `REDIS_URL`). Adicione apenas se precisar verificar APIs externas:

```typescript
// src/config/env.config.ts — adicione APENAS se o projeto verificar APIs externas
const envSchema = z.object({
  // ... variáveis existentes
  EXTERNAL_API_URL: z.string().url().optional(),   // URL da API externa a verificar
});
```

---

## 4. Estrutura de Arquivos

```text
src/
└── infra/
    └── health/
        ├── health.module.ts                    ← módulo do Terminus
        ├── health.controller.ts                ← endpoints /health/live e /health/ready
        ├── indicators/
        │   ├── prisma.health.ts                ← custom indicator para Prisma
        │   └── redis.health.ts                 ← custom indicator para Redis (ioredis)
        └── __tests__/
            └── health.controller.spec.ts       ← testes de integração
```

> **Localização em `src/infra/`**: Seguindo `nest-project-structure.md`, health checks são infraestrutura — não lógica de domínio. Ficam em `src/infra/health/`, paralelo a `src/infra/database/` e `src/infra/cache/`.

---

## 5. Custom Indicator — Prisma

O `@nestjs/terminus` não inclui indicador nativo para Prisma — apenas para TypeORM, Sequelize e Mongoose. Crie um `HealthIndicator` customizado:

```typescript
// src/infra/health/indicators/prisma.health.ts
import { Injectable } from '@nestjs/common';
import {
  HealthIndicator,
  HealthIndicatorResult,
  HealthCheckError,
} from '@nestjs/terminus';
import { PrismaService } from '@infra/database/prisma.service';

/**
 * Health indicator customizado para Prisma ORM.
 *
 * Executa `SELECT 1` via `$queryRaw` para verificar conectividade
 * com o banco sem depender de tabelas específicas do domínio.
 *
 * Usado em: /health/ready (readiness probe)
 */
@Injectable()
export class PrismaHealthIndicator extends HealthIndicator {
  constructor(private readonly prisma: PrismaService) {
    super();
  }

  async pingCheck(key: string): Promise<HealthIndicatorResult> {
    try {
      // SELECT 1 — verificação mínima de conectividade
      await this.prisma.$queryRaw`SELECT 1`;

      return this.getStatus(key, true);
    } catch (error) {
      throw new HealthCheckError(
        `${key} check failed`,
        this.getStatus(key, false, {
          message: error instanceof Error ? error.message : 'Unknown error',
        }),
      );
    }
  }
}
```

---

## 6. Custom Indicator — Redis

O Terminus não inclui indicador nativo para Redis standalone (apenas para microservices Redis transport). Crie um indicador que reutiliza a instância `ioredis` já configurada no `AppCacheModule`:

```typescript
// src/infra/health/indicators/redis.health.ts
import { Injectable } from '@nestjs/common';
import {
  HealthIndicator,
  HealthIndicatorResult,
  HealthCheckError,
} from '@nestjs/terminus';
import { InjectRedis } from '@nestjs-modules/ioredis';   // ou via token direto
import type Redis from 'ioredis';

/**
 * Health indicator customizado para Redis via ioredis.
 *
 * Executa PING → espera PONG para verificar conectividade.
 * Reutiliza a instância de Redis já configurada no AppCacheModule.
 *
 * IMPORTANTE: Importe o token correto de acordo com como o projeto
 * expõe o cliente ioredis. Se o projeto usa @keyv/redis sem expor
 * a instância diretamente, use a abordagem com IORedis standalone
 * via REDIS_URL (seção 6.1).
 *
 * Usado em: /health/ready (readiness probe)
 */
@Injectable()
export class RedisHealthIndicator extends HealthIndicator {
  constructor(
    @InjectRedis() private readonly redis: Redis,
  ) {
    super();
  }

  async pingCheck(key: string): Promise<HealthIndicatorResult> {
    try {
      const result = await this.redis.ping();

      if (result !== 'PONG') {
        throw new Error(`Expected PONG, received: ${result}`);
      }

      return this.getStatus(key, true);
    } catch (error) {
      throw new HealthCheckError(
        `${key} check failed`,
        this.getStatus(key, false, {
          message: error instanceof Error ? error.message : 'Unknown error',
        }),
      );
    }
  }
}
```

### 6.1 Alternativa: Redis sem decorador @InjectRedis

Se o projeto não usa `@nestjs-modules/ioredis` e expõe o Redis apenas via `@keyv/redis`, crie uma conexão dedicada para health check usando `REDIS_URL`:

```typescript
// src/infra/health/indicators/redis.health.ts (alternativa sem @InjectRedis)
import { Injectable, OnModuleDestroy } from '@nestjs/common';
import {
  HealthIndicator,
  HealthIndicatorResult,
  HealthCheckError,
} from '@nestjs/terminus';
import { ConfigType } from '@nestjs/config';
import { Inject } from '@nestjs/common';
import Redis from 'ioredis';
import cacheConfig from '@config/cache.config';

@Injectable()
export class RedisHealthIndicator
  extends HealthIndicator
  implements OnModuleDestroy
{
  private readonly client: Redis | null;

  constructor(
    @Inject(cacheConfig.KEY)
    private readonly config: ConfigType<typeof cacheConfig>,
  ) {
    super();
    // Cria cliente apenas se REDIS_URL estiver configurada
    this.client = config.redisUrl
      ? new Redis(config.redisUrl, {
          maxRetriesPerRequest: 1,      // falha rápido em health check
          connectTimeout: 2000,         // 2s timeout
          lazyConnect: true,            // não conecta no construtor
        })
      : null;
  }

  async onModuleDestroy(): Promise<void> {
    await this.client?.quit();
  }

  async pingCheck(key: string): Promise<HealthIndicatorResult> {
    // Se Redis não está configurado, retorna up com aviso
    if (!this.client) {
      return this.getStatus(key, true, { note: 'Redis not configured' });
    }

    try {
      const result = await this.client.ping();

      if (result !== 'PONG') {
        throw new Error(`Expected PONG, received: ${result}`);
      }

      return this.getStatus(key, true);
    } catch (error) {
      throw new HealthCheckError(
        `${key} check failed`,
        this.getStatus(key, false, {
          message: error instanceof Error ? error.message : 'Unknown error',
        }),
      );
    }
  }
}
```

---

## 7. Controller — `/health/live` e `/health/ready`

```typescript
// src/infra/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { VERSION_NEUTRAL } from '@nestjs/common';
import { ApiTags, ApiOperation } from '@nestjs/swagger';
import { SkipThrottle } from '@nestjs/throttler';
import {
  HealthCheck,
  HealthCheckResult,
  HealthCheckService,
  HttpHealthIndicator,
  MemoryHealthIndicator,
} from '@nestjs/terminus';
import { PrismaHealthIndicator } from './indicators/prisma.health';
import { RedisHealthIndicator } from './indicators/redis.health';

/**
 * Controller de health checks.
 *
 * - VERSION_NEUTRAL: sem prefixo de versão (cf. rest-api-patterns.md)
 * - @SkipThrottle: monitoramento externo não deve ser bloqueado (cf. security-hardening.md)
 * - Excluído do auto-logging do Pino (cf. logging-pattern.md — autoLogging.ignore)
 * - Excluído do envelope { data, meta } — resposta no formato Terminus nativo
 *
 * Rotas:
 *   GET /health/live  → liveness probe  (apenas processo em pé)
 *   GET /health/ready → readiness probe (banco + redis + dependências)
 */
@ApiTags('health')
@Controller({ path: 'health', version: VERSION_NEUTRAL })
@SkipThrottle()
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly memory: MemoryHealthIndicator,
    private readonly prismaHealth: PrismaHealthIndicator,
    private readonly redisHealth: RedisHealthIndicator,
    private readonly http: HttpHealthIndicator,
  ) {}

  /**
   * GET /health/live — Liveness Probe
   *
   * Propósito: verificar se o processo Node.js está vivo e responsivo.
   * Kubernetes reinicia o container se este endpoint falhar.
   *
   * ⚠️  NÃO inclua checks de banco ou Redis aqui.
   *     Uma indisponibilidade do banco causaria reinício desnecessário do pod.
   *
   * Checks:
   * - Heap: processo não está em OOM (>= 500MB disponível indica normalidade)
   */
  @Get('live')
  @HealthCheck()
  @ApiOperation({
    summary: 'Liveness probe',
    description: 'Verifica se o processo está vivo. Usado pelo Kubernetes para decidir reiniciar o container.',
  })
  async liveness(): Promise<HealthCheckResult> {
    return this.health.check([
      // Memória heap — detecta vazamentos que travam o processo
      () => this.memory.checkHeap('memory_heap', 500 * 1024 * 1024), // 500 MB
    ]);
  }

  /**
   * GET /health/ready — Readiness Probe
   *
   * Propósito: verificar se a aplicação está pronta para receber tráfego.
   * Kubernetes remove o pod do load balancer se este endpoint falhar (sem reiniciar).
   *
   * Checks:
   * - Banco de dados (Prisma): SELECT 1
   * - Redis: PING → PONG
   * - APIs externas: HTTP ping (opcional — descomente se o projeto usar)
   */
  @Get('ready')
  @HealthCheck()
  @ApiOperation({
    summary: 'Readiness probe',
    description: 'Verifica se a aplicação está pronta para receber tráfego (banco, Redis e dependências).',
  })
  async readiness(): Promise<HealthCheckResult> {
    return this.health.check([
      // Banco de dados via Prisma
      () => this.prismaHealth.pingCheck('database'),

      // Redis (omita se REDIS_URL não estiver configurada)
      () => this.redisHealth.pingCheck('redis'),

      // ── API externa (descomente se necessário) ─────────────────────────
      // () => this.http.pingCheck('payments-api', env.EXTERNAL_API_URL, {
      //   timeout: 3000,
      // }),
    ]);
  }
}
```

---

## 8. Módulo — `HealthModule`

```typescript
// src/infra/health/health.module.ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';
import { PrismaHealthIndicator } from './indicators/prisma.health';
import { RedisHealthIndicator } from './indicators/redis.health';

/**
 * HealthModule — registra o Terminus e os indicadores customizados.
 *
 * Não é @Global() — importado diretamente no AppModule.
 * DatabaseModule e AppCacheModule devem estar registrados ANTES no AppModule.
 */
@Module({
  imports: [
    TerminusModule,
    HttpModule,   // necessário para HttpHealthIndicator
  ],
  controllers: [HealthController],
  providers: [
    PrismaHealthIndicator,
    RedisHealthIndicator,
  ],
})
export class HealthModule {}
```

### Registro no `AppModule`

```typescript
// src/app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [...] }),
    DatabaseModule,      // ← deve estar ANTES do HealthModule (PrismaService)
    AppCacheModule,      // ← deve estar ANTES do HealthModule (Redis)
    HealthModule,        // ← após as dependências
    // ... módulos de domínio
  ],
})
export class AppModule {}
```

---

## 9. Exclusão do Envelope Global

O `TransformInterceptor` (de `rest-api-patterns.md`) envolve todas as respostas em `{ data, meta }`. O Terminus já tem seu próprio formato de resposta — o envelope deve ser **excluído** para as rotas de health:

```typescript
// src/common/interceptors/transform.interceptor.ts
// Adicione verificação para excluir rotas de health do envelope:
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    // ← Exclui /health/* do envelope — Terminus tem formato próprio
    if (request.url?.startsWith('/health')) {
      return next.handle();
    }

    return next.handle().pipe(
      map((data) => ({ data })),
    );
  }
}
```

---

## 10. Integração com Kubernetes Probes

### `deployment.yaml` — configuração recomendada

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nestjs-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nestjs-api
  template:
    metadata:
      labels:
        app: nestjs-api
    spec:
      containers:
        - name: nestjs-api
          image: my-registry/nestjs-api:latest
          ports:
            - containerPort: 3000

          # ── Startup Probe ──────────────────────────────────────────────
          # Aguarda até 60s (12 × 5s) para o app terminar de inicializar.
          # Enquanto pendente, desativa liveness e readiness probes.
          # Use para apps com start lento (migrations, warm-up de cache).
          startupProbe:
            httpGet:
              path: /health/live
              port: 3000
            failureThreshold: 12      # 12 tentativas × 5s = 60s máximo
            periodSeconds: 5
            timeoutSeconds: 3

          # ── Liveness Probe ─────────────────────────────────────────────
          # Verifica se o processo está vivo — reinicia em falha.
          # Simples: apenas memória heap, sem dependências externas.
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 0    # startup probe já cobre o delay inicial
            periodSeconds: 15         # verifica a cada 15s
            timeoutSeconds: 3         # timeout da requisição HTTP
            failureThreshold: 3       # 3 falhas consecutivas → reinicia

          # ── Readiness Probe ────────────────────────────────────────────
          # Verifica banco + Redis — remove do load balancer em falha (sem reiniciar).
          # Mais frequente que liveness para remoção rápida em degradação.
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10         # verifica a cada 10s
            timeoutSeconds: 5         # timeout maior — inclui checks de banco
            failureThreshold: 3       # 3 falhas → remove do load balancer
            successThreshold: 1       # 1 sucesso → volta ao load balancer

          # ── Resources ──────────────────────────────────────────────────
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"   # alinhado com checkHeap de 500MB no liveness

          # ── Graceful shutdown ──────────────────────────────────────────
          lifecycle:
            preStop:
              exec:
                # Aguarda 15s antes do SIGTERM para o ingress remover o pod
                # do load balancer antes de começar o shutdown.
                # Alinhado com terminationGracePeriodSeconds abaixo.
                command: ["/bin/sh", "-c", "sleep 15"]

      # Tempo total para shutdown gracioso após SIGTERM
      # Deve ser: preStop delay + tempo de drain de conexões
      terminationGracePeriodSeconds: 30
```

---

## 11. Graceful Shutdown no NestJS

O Terminus integra com o sistema de shutdown do NestJS para garantir que o processo não termine enquanto há requisições em andamento:

```typescript
// src/main.ts — adicione enableShutdownHooks
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    bufferLogs: true,   // cf. logging-pattern.md
  });

  // ... configurações existentes (helmet, cors, pipes, etc.)

  // ── Graceful Shutdown ────────────────────────────────────────────────────
  // Habilita os hooks de shutdown do NestJS (OnModuleDestroy, BeforeApplicationShutdown).
  // OBRIGATÓRIO para o Terminus funcionar corretamente em Kubernetes.
  // O Terminus usa esses hooks para aguardar requisições em voo antes de fechar.
  app.enableShutdownHooks();

  const logger = app.get(Logger);   // cf. logging-pattern.md
  app.useLogger(logger);

  await app.listen(env.PORT);
}
bootstrap();
```

---

## 12. Formato de Resposta do Terminus

O Terminus retorna `200 OK` quando tudo está saudável e `503 Service Unavailable` quando qualquer indicador falha.

### Resposta saudável (`200 OK`)

```json
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "redis":    { "status": "up" }
  },
  "error": {},
  "details": {
    "database": { "status": "up" },
    "redis":    { "status": "up" }
  }
}
```

### Resposta degradada (`503 Service Unavailable`)

```json
{
  "status": "error",
  "info": {
    "database": { "status": "up" }
  },
  "error": {
    "redis": {
      "status": "down",
      "message": "Expected PONG, received: Connection refused"
    }
  },
  "details": {
    "database": { "status": "up" },
    "redis": {
      "status": "down",
      "message": "Expected PONG, received: Connection refused"
    }
  }
}
```

> **Nota de segurança:** Em produção, considere não expor detalhes de erro do `error` field para clientes externos. O Kubernetes e load balancers se baseiam apenas no status HTTP (200 vs 503) — os detalhes são para observabilidade interna.

---

## 13. Integração com Logging (logging-pattern.md)

O `logging-pattern.md` já define a exclusão de `/health` do auto-logging do Pino. Confirme que a configuração está presente:

```typescript
// src/app.module.ts — dentro do LoggerModule.forRoot()
autoLogging: {
  // ← /health/* excluído — evita flood de logs de probe do Kubernetes
  ignore: (req) => ['/health/live', '/health/ready', '/metrics', '/favicon.ico']
    .some(path => req.url?.startsWith(path) ?? false),
},
```

---

## 14. Testes

```typescript
// src/infra/health/__tests__/health.controller.spec.ts
import { INestApplication } from '@nestjs/common';
import { Test, TestingModule } from '@nestjs/testing';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import * as request from 'supertest';
import { HealthController } from '../health.controller';
import { PrismaHealthIndicator } from '../indicators/prisma.health';
import { RedisHealthIndicator } from '../indicators/redis.health';

describe('HealthController', () => {
  let app: INestApplication;

  // ── Mocks ──────────────────────────────────────────────────────────────────
  const mockPrismaHealth = {
    pingCheck: jest.fn(),
  };

  const mockRedisHealth = {
    pingCheck: jest.fn(),
  };

  beforeAll(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [TerminusModule, HttpModule],
      controllers: [HealthController],
      providers: [
        { provide: PrismaHealthIndicator, useValue: mockPrismaHealth },
        { provide: RedisHealthIndicator,  useValue: mockRedisHealth  },
      ],
    }).compile();

    app = module.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  // ── Liveness ───────────────────────────────────────────────────────────────

  describe('GET /health/live', () => {
    it('deve retornar 200 quando o processo está saudável', async () => {
      const response = await request(app.getHttpServer())
        .get('/health/live')
        .expect(200);

      expect(response.body.status).toBe('ok');
    });
  });

  // ── Readiness ──────────────────────────────────────────────────────────────

  describe('GET /health/ready', () => {
    it('deve retornar 200 quando banco e Redis estão saudáveis', async () => {
      mockPrismaHealth.pingCheck.mockResolvedValueOnce({ database: { status: 'up' } });
      mockRedisHealth.pingCheck.mockResolvedValueOnce({ redis: { status: 'up' } });

      const response = await request(app.getHttpServer())
        .get('/health/ready')
        .expect(200);

      expect(response.body.status).toBe('ok');
      expect(response.body.info.database.status).toBe('up');
      expect(response.body.info.redis.status).toBe('up');
    });

    it('deve retornar 503 quando o banco está indisponível', async () => {
      const { HealthCheckError } = await import('@nestjs/terminus');

      mockPrismaHealth.pingCheck.mockRejectedValueOnce(
        new HealthCheckError('database check failed', {
          database: { status: 'down', message: 'Connection refused' },
        }),
      );
      mockRedisHealth.pingCheck.mockResolvedValueOnce({ redis: { status: 'up' } });

      const response = await request(app.getHttpServer())
        .get('/health/ready')
        .expect(503);

      expect(response.body.status).toBe('error');
      expect(response.body.error.database.status).toBe('down');
    });

    it('deve retornar 503 quando o Redis está indisponível', async () => {
      const { HealthCheckError } = await import('@nestjs/terminus');

      mockPrismaHealth.pingCheck.mockResolvedValueOnce({ database: { status: 'up' } });
      mockRedisHealth.pingCheck.mockRejectedValueOnce(
        new HealthCheckError('redis check failed', {
          redis: { status: 'down', message: 'Expected PONG' },
        }),
      );

      const response = await request(app.getHttpServer())
        .get('/health/ready')
        .expect(503);

      expect(response.body.status).toBe('error');
      expect(response.body.error.redis.status).toBe('down');
    });
  });
});
```

---

## 15. Anti-Padrões — O que Nunca Fazer

```typescript
// ❌ Check de banco no liveness — causa restart desnecessário durante indisponibilidade do DB
@Get('live')
liveness() {
  return this.health.check([
    () => this.prismaHealth.pingCheck('database'), // ← ERRADO: está no live
  ]);
}

// ❌ Retornar 200 sempre — inútil para o Kubernetes
@Get('ready')
readiness() {
  return { status: 'ok' }; // ← ERRADO: não usa Terminus, não detecta falhas reais
}

// ❌ Endpoint de health com rate limiting — bloqueia o Kubernetes
@Controller({ path: 'health', version: VERSION_NEUTRAL })
// sem @SkipThrottle() ← ERRADO: probes do K8s serão bloqueadas após limit

// ❌ Envelope { data, meta } nas respostas de health — quebra o parse do Kubernetes
// TransformInterceptor deve excluir /health/* (cf. seção 9)

// ❌ Expor health checks com autenticação — probes do K8s não têm JWT
@UseGuards(JwtAuthGuard) // ← ERRADO: o Kubernetes não tem token
@Get('ready')
readiness() { ... }

// ❌ Timeout longo no health check de banco — bloqueia o probe
await this.prisma.$queryRaw`SELECT * FROM users LIMIT 100`; // ← use SELECT 1
```

---

## 16. Checklist de Revisão

### Estrutura
- [ ] `HealthModule` em `src/infra/health/` (não em `src/modules/`)?
- [ ] `HealthModule` registrado no `AppModule` após `DatabaseModule` e `AppCacheModule`?
- [ ] `PrismaHealthIndicator` e `RedisHealthIndicator` como custom `HealthIndicator`?

### Controller
- [ ] `VERSION_NEUTRAL` no controller (sem prefixo `/api/v1/`)?
- [ ] `@SkipThrottle()` no controller (cf. `security-hardening.md`)?
- [ ] `/health/live` contém APENAS checks de processo (memória) — sem banco ou Redis?
- [ ] `/health/ready` contém banco, Redis e APIs externas?
- [ ] `TransformInterceptor` excluindo `/health/*` do envelope `{ data, meta }`?

### Liveness
- [ ] `checkHeap` configurado com limite alinhado ao `resources.limits.memory` do Kubernetes?
- [ ] Nenhuma dependência externa no endpoint de liveness?

### Readiness
- [ ] `PrismaHealthIndicator.pingCheck` executa `SELECT 1` (não queries de domínio)?
- [ ] `RedisHealthIndicator.pingCheck` executa `PING → PONG`?
- [ ] Timeout do `pingCheck` de banco ≤ `timeoutSeconds` do readinessProbe no K8s?

### Kubernetes
- [ ] `startupProbe` configurado para apps com start lento (migrations, etc.)?
- [ ] `livenessProbe` apontando para `/health/live`?
- [ ] `readinessProbe` apontando para `/health/ready`?
- [ ] `preStop` sleep configurado (15s recomendado) para drain do ingress?
- [ ] `terminationGracePeriodSeconds` > `preStop sleep + drain time`?

### Graceful Shutdown
- [ ] `app.enableShutdownHooks()` no `main.ts`?
- [ ] `OnModuleDestroy` implementado no `RedisHealthIndicator` (fecha conexão dedicada)?

### Logging
- [ ] `/health/live` e `/health/ready` excluídos do `autoLogging` do Pino?
- [ ] Rotas de health não geram logs de request/response em produção?

### Testes
- [ ] Testes de integração com mock de `PrismaHealthIndicator` e `RedisHealthIndicator`?
- [ ] Cenário 200 (tudo up) coberto?
- [ ] Cenário 503 (banco down) coberto?
- [ ] Cenário 503 (Redis down) coberto?

---

## Referências

- [nest-project-structure.md](../foundation-and-architecture/nest-project-structure.md) — `src/infra/`, separação de responsabilidades
- [config-management.md](../configuration-and-environment/config-management.md) — `cacheConfig.KEY`, acesso a `REDIS_URL`
- [rest-api-patterns.md](../api-and-contracts/rest-api-patterns.md) — `VERSION_NEUTRAL`, `TransformInterceptor`, `main.ts`
- [security-hardening.md](../authentication-and-security/security-hardening.md) — `@SkipThrottle()` em endpoints de infra
- [logging-pattern.md](./logging-pattern.md) — `autoLogging.ignore` para `/health/*`
- [prisma-patterns.md](../database/prisma-patterns.md) — `PrismaService`, `$queryRaw`
- [caching-strategy.md](../performance-and-infrastructure/caching-strategy.md) — `ioredis` e `REDIS_URL`
- [NestJS Docs — Health checks (Terminus)](https://docs.nestjs.com/recipes/terminus)
- [GitHub — nestjs/terminus](https://github.com/nestjs/terminus)
- [Better Stack — Kubernetes Health Checks](https://betterstack.com/community/guides/monitoring/kubernetes-health-checks/)
- [Kubernetes Docs — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [GoDaddy Terminus — Graceful Shutdown](https://github.com/godaddy/terminus#graceful-shutdown)