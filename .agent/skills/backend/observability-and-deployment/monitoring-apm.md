---
name: monitoring-apm
description: Observabilidade em produção para NestJS — tracing distribuído com OpenTelemetry (OTLP + Tempo/Jaeger), métricas RED/USE com Prometheus + Grafana, integração com @sentry/nestjs para erros e performance, alertas por SLO via Prometheus Alertmanager e Grafana. Use quando configurar observabilidade do zero, adicionar tracing distribuído, expor métricas para Prometheus, integrar Sentry no backend, ou definir alertas baseados em SLO.
---

# Monitoramento e APM — Observabilidade em Produção (NestJS)

Você é um Engenheiro de Software Senior especialista em observabilidade de sistemas distribuídos. Sua responsabilidade é garantir que o projeto tenha **visibilidade completa sobre traces, métricas e erros em produção**, usando o stack OpenTelemetry + Prometheus + Grafana + Sentry de forma integrada e consistente.

> **Pré-requisitos:** Esta skill assume que as skills `config-management`, `nest-project-structure`, `logging-pattern` e `health-checks` já foram aplicadas. O objeto `env` de `@config/env.config` é a fonte da verdade para variáveis de ambiente. O `HttpExceptionFilter` global de `rest-api-patterns` é o ponto de integração com Sentry para erros HTTP. O `correlationId` do `logging-pattern` é propagado como atributo de span do OpenTelemetry.

---

## Quando usar esta skill

- **Novo projeto:** Configurar observabilidade completa (tracing + métricas + erros) do zero.
- **Tracing distribuído:** Rastrear requisições ponta-a-ponta em microserviços ou monolitos com dependências externas.
- **Métricas de negócio:** Expor métricas RED (Rate, Error, Duration) e USE (Utilization, Saturation, Errors) para Prometheus.
- **Sentry no backend:** Capturar exceções não tratadas, performance e logs estruturados.
- **SLO / Alertas:** Definir Service Level Objectives e configurar alertas baseados neles.
- **Correlação logs + traces:** Injetar `traceId` e `spanId` nos logs Pino para navegação entre sistemas.

---

## 1. Decisões de Design

### Stack de Observabilidade

| Camada | Ferramenta | Responsabilidade |
|---|---|---|
| **Tracing** | OpenTelemetry SDK + OTLP Exporter | Instrumentação e exportação de traces |
| **Backend de Traces** | Grafana Tempo ou Jaeger | Armazenamento e visualização de traces |
| **Métricas** | OpenTelemetry SDK + Prometheus Exporter | Exposição de métricas no endpoint `/metrics` |
| **Backend de Métricas** | Prometheus | Scrape, armazenamento, alertas |
| **Visualização** | Grafana | Dashboards, correlação traces+métricas+logs |
| **Erros / APM** | @sentry/nestjs | Captura de exceções, performance, profiling |
| **Logs** | Pino + nestjs-pino | já definido em `logging-pattern.md` |

### Por que OpenTelemetry e não apenas Sentry Tracing?

| Critério | OpenTelemetry | Sentry Tracing |
|---|---|---|
| Vendor-neutral | ✅ Exporta para qualquer backend | ❌ Proprietário Sentry |
| Auto-instrumentação | ✅ HTTP, DB, Redis, BullMQ | ✅ Apenas camadas Sentry |
| Métricas Prometheus | ✅ PrometheusExporter nativo | ❌ Não suporta |
| Correlação com logs | ✅ Injeta traceId/spanId no Pino | Parcial |
| Custo | ✅ Self-hosted (Tempo/Jaeger) | 💰 Depende do plano |

**Decisão:** OpenTelemetry como instrumentação principal + Sentry como camada complementar de erros/alertas/profiling. Os dois coexistem sem conflito — o Sentry usa seu próprio transporte; o OTel exporta via OTLP.

> **Importante:** O SDK do OpenTelemetry **deve ser inicializado antes** do NestJS fazer qualquer import. Por isso, o `tracing.ts` é importado como primeira linha do `main.ts` — não como módulo NestJS.

---

## 2. Instalação

### 2.1 OpenTelemetry

```bash
# Core SDK
npm install @opentelemetry/sdk-node \
  @opentelemetry/api \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions

# Exporters
npm install @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/exporter-prometheus

# Trace processing
npm install @opentelemetry/sdk-trace-node \
  @opentelemetry/sdk-trace-base

# Metrics
npm install @opentelemetry/sdk-metrics

# Pino injection (correlação traceId → logs)
npm install @opentelemetry/instrumentation-pino
```

### 2.2 Sentry

```bash
npm install @sentry/nestjs @sentry/profiling-node
```

> **Atenção:** `@sentry/nestjs` é o SDK oficial da Sentry para NestJS (disponível desde SDK v8). **Não use** `@ntegral/nestjs-sentry` em projetos novos — é uma biblioteca de terceiros com manutenção irregular.

---

## 3. Variáveis de Ambiente

Adicione ao schema Zod em `src/config/env.config.ts` (conforme `config-management.md`):

```typescript
// src/config/env.config.ts — adicionar ao envSchema existente
const envSchema = z.object({
  // ... variáveis existentes

  // ─── OpenTelemetry ────────────────────────────────────────────────────────
  // Nome do serviço — aparece em todos os traces e métricas
  OTEL_SERVICE_NAME: z.string().default('nestjs-app'),
  // Versão do serviço — correlaciona deploys com anomalias
  OTEL_SERVICE_VERSION: z.string().default('0.0.0'),
  // Endpoint do OpenTelemetry Collector ou backend direto (Tempo, Jaeger)
  OTEL_EXPORTER_OTLP_ENDPOINT: z.string().url().optional(),
  // Taxa de amostragem de traces: 1.0 = 100% (dev), 0.1 = 10% (prod alto tráfego)
  OTEL_TRACES_SAMPLE_RATE: z.coerce.number().min(0).max(1).default(1.0),
  // Porta onde o PrometheusExporter expõe /metrics (separado do app)
  OTEL_PROMETHEUS_PORT: z.coerce.number().int().positive().default(9464),

  // ─── Sentry ───────────────────────────────────────────────────────────────
  SENTRY_DSN: z.string().url().optional(),
  // Taxa de amostragem de traces no Sentry (independente do OTel)
  SENTRY_TRACES_SAMPLE_RATE: z.coerce.number().min(0).max(1).default(0.1),
  // Taxa de profiling (relativa ao SENTRY_TRACES_SAMPLE_RATE)
  SENTRY_PROFILES_SAMPLE_RATE: z.coerce.number().min(0).max(1).default(0.1),
});
```

**`.env.example`** — adicione:

```bash
# OpenTelemetry
OTEL_SERVICE_NAME=meu-servico
OTEL_SERVICE_VERSION=1.0.0
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318  # OTel Collector
OTEL_TRACES_SAMPLE_RATE=1.0                        # 100% em dev, reduzir em prod
OTEL_PROMETHEUS_PORT=9464

# Sentry
SENTRY_DSN=https://xxxxx@oyyy.ingest.sentry.io/zzzzz
SENTRY_TRACES_SAMPLE_RATE=0.1
SENTRY_PROFILES_SAMPLE_RATE=0.1
```

---

## 4. Inicialização do OpenTelemetry — `src/infra/tracing/tracing.ts`

> **Regra crítica:** Este arquivo deve ser importado **antes de qualquer outro** no `main.ts`. O OTel precisa "monkeypatch" módulos como `http`, `pg`, `ioredis` antes que eles sejam carregados pelo Node.js.

```typescript
// src/infra/tracing/tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { Resource } from '@opentelemetry/resources';
import { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION, SEMRESATTRS_DEPLOYMENT_ENVIRONMENT } from '@opentelemetry/semantic-conventions';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { PinoInstrumentation } from '@opentelemetry/instrumentation-pino';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { BatchSpanProcessor, ParentBasedSampler, TraceIdRatioBasedSampler } from '@opentelemetry/sdk-trace-node';
import { PrometheusExporter } from '@opentelemetry/exporter-prometheus';

// Lê diretamente de process.env — env.config.ts ainda não foi importado aqui
const SERVICE_NAME    = process.env.OTEL_SERVICE_NAME    ?? 'nestjs-app';
const SERVICE_VERSION = process.env.OTEL_SERVICE_VERSION ?? '0.0.0';
const NODE_ENV        = process.env.NODE_ENV             ?? 'development';
const OTLP_ENDPOINT   = process.env.OTEL_EXPORTER_OTLP_ENDPOINT;
const SAMPLE_RATE     = parseFloat(process.env.OTEL_TRACES_SAMPLE_RATE ?? '1.0');
const PROMETHEUS_PORT = parseInt(process.env.OTEL_PROMETHEUS_PORT ?? '9464', 10);

// ── Prometheus Exporter (métricas no endpoint :9464/metrics) ─────────────────
const prometheusExporter = new PrometheusExporter({
  port: PROMETHEUS_PORT,
  endpoint: '/metrics',
});

// ── OTLP Trace Exporter (para Grafana Tempo, Jaeger, OTel Collector) ─────────
const traceExporter = OTLP_ENDPOINT
  ? new OTLPTraceExporter({
      url: `${OTLP_ENDPOINT}/v1/traces`,
      headers: {},
    })
  : undefined;

// ── SDK ───────────────────────────────────────────────────────────────────────
const sdk = new NodeSDK({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: SERVICE_NAME,
    [SEMRESATTRS_SERVICE_VERSION]: SERVICE_VERSION,
    [SEMRESATTRS_DEPLOYMENT_ENVIRONMENT]: NODE_ENV,
  }),

  // Sampling: ParentBased respeita a decisão do serviço upstream (propagação W3C)
  // TraceIdRatioBased amostra X% das requisições raiz
  sampler: new ParentBasedSampler({
    root: new TraceIdRatioBasedSampler(SAMPLE_RATE),
  }),

  // Batch processor: eficiente para produção (agrupa spans antes de exportar)
  spanProcessors: traceExporter
    ? [new BatchSpanProcessor(traceExporter, {
        maxQueueSize: 1000,
        scheduledDelayMillis: 5000,
        exportTimeoutMillis: 30_000,
      })]
    : [],

  // Métricas via Prometheus
  metricReader: prometheusExporter,

  // Auto-instrumentações: HTTP, Express/Fastify, pg, ioredis, BullMQ, etc.
  instrumentations: [
    getNodeAutoInstrumentations({
      // Não instrumenta chamadas a arquivos (muita verbosidade)
      '@opentelemetry/instrumentation-fs': { enabled: false },
      '@opentelemetry/instrumentation-http': {
        // Ignora health checks e scrape de métricas — evita flood de traces
        ignoreIncomingRequestHook: (req) => {
          const url = req.url ?? '';
          return ['/health/live', '/health/ready', '/metrics', '/favicon.ico']
            .some((path) => url.startsWith(path));
        },
      },
    }),
    // Injeta traceId e spanId automaticamente nos logs Pino
    new PinoInstrumentation(),
  ],
});

// ── Shutdown gracioso ─────────────────────────────────────────────────────────
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => process.exit(0))
    .catch((err) => {
      console.error('Erro ao finalizar OTel SDK', err);
      process.exit(1);
    });
});

sdk.start();

export { sdk };
```

---

## 5. Inicialização do Sentry — `src/infra/sentry/instrument.ts`

```typescript
// src/infra/sentry/instrument.ts
// DEVE ser importado antes de qualquer outro módulo no main.ts
import * as Sentry from '@sentry/nestjs';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

const dsn        = process.env.SENTRY_DSN;
const nodeEnv    = process.env.NODE_ENV ?? 'development';
const isProd     = nodeEnv === 'production';
const traceRate  = parseFloat(process.env.SENTRY_TRACES_SAMPLE_RATE  ?? '0.1');
const profileRate = parseFloat(process.env.SENTRY_PROFILES_SAMPLE_RATE ?? '0.1');

// Inicializa apenas se SENTRY_DSN estiver configurado
// Em development sem DSN, o Sentry é silencioso
if (dsn) {
  Sentry.init({
    dsn,
    environment: nodeEnv,
    release: process.env.OTEL_SERVICE_VERSION ?? '0.0.0',

    // ── Performance ───────────────────────────────────────────────────────────
    tracesSampleRate: traceRate,

    // ── Profiling ─────────────────────────────────────────────────────────────
    integrations: [nodeProfilingIntegration()],
    profilesSampleRate: profileRate,

    // ── Filtros de erro ───────────────────────────────────────────────────────
    beforeSend: (event, hint) => {
      const exception = hint?.originalException;

      // Não reportar erros 4xx para o Sentry (são esperados, não bugs)
      // O HttpExceptionFilter de rest-api-patterns.md já loga esses casos
      if (exception instanceof Error && 'status' in exception) {
        const status = (exception as { status: number }).status;
        if (status >= 400 && status < 500) return null;
      }

      // Remover dados sensíveis (LGPD/GDPR) — alinhado com logging-pattern.md
      if (event.user?.email) delete event.user.email;
      if (event.request?.data) {
        const data = event.request.data as Record<string, unknown>;
        ['password', 'token', 'cpf', 'card_number', 'cvv', 'secret'].forEach(
          (f) => { if (f in data) data[f] = '[Redacted]'; },
        );
      }

      return event;
    },

    // Ignora health checks e scrape de métricas
    ignoreTransactions: ['/health/live', '/health/ready', '/metrics'],
  });
}

export { Sentry };
```

---

## 6. `main.ts` — Ordem de Inicialização

A ordem é **crítica**: OTel → Sentry → NestJS bootstrap.

```typescript
// src/main.ts
// ── 1. OpenTelemetry PRIMEIRO — antes de qualquer import de módulo ────────────
import './infra/tracing/tracing';

// ── 2. Sentry SEGUNDO — antes do NestJS ────────────────────────────────────────
import './infra/sentry/instrument';

// ── 3. Agora o NestJS e demais imports ────────────────────────────────────────
import { NestFactory } from '@nestjs/core';
import { Logger }      from 'nestjs-pino';
import { AppModule }   from './app.module';
import { env }         from '@config/env.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });

  app.useLogger(app.get(Logger));
  app.enableShutdownHooks(); // requerido por health-checks.md

  // ... demais configurações (helmet, cors, pipes, versioning, etc.)
  // conforme rest-api-patterns.md e security-hardening.md

  await app.listen(env.PORT);
}

bootstrap();
```

---

## 7. Integração Sentry com HttpExceptionFilter

A skill `rest-api-patterns.md` define o `HttpExceptionFilter` global. Para integrar o Sentry, adicione o decorator `@SentryExceptionCaptured()` **no filter existente**, sem criar um novo:

```typescript
// src/common/filters/http-exception.filter.ts — ADAPTAR o filter existente
import { Catch, ArgumentsHost, HttpException, HttpStatus } from '@nestjs/common';
import { BaseExceptionFilter }    from '@nestjs/core';
import { SentryExceptionCaptured } from '@sentry/nestjs';  // ← adicionar import
import { PinoLogger, InjectPinoLogger } from 'nestjs-pino';

@Catch()
export class HttpExceptionFilter extends BaseExceptionFilter {
  constructor(
    @InjectPinoLogger(HttpExceptionFilter.name)
    private readonly logger: PinoLogger,
  ) {
    super();
  }

  // ← @SentryExceptionCaptured faz o Sentry capturar ANTES do catch processar
  @SentryExceptionCaptured()
  catch(exception: unknown, host: ArgumentsHost): void {
    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    // Log apenas erros 5xx — erros 4xx são esperados (cf. logging-pattern.md)
    if (status >= 500) {
      this.logger.error({ err: exception }, 'Erro interno não tratado');
    }

    super.catch(exception, host);
  }
}
```

> **Regra:** Se o projeto **não tiver** um `HttpExceptionFilter` customizado, registre o `SentryGlobalFilter` como provider global no `AppModule`:
>
> ```typescript
> // src/app.module.ts — APENAS se não houver filter customizado
> import { SentryGlobalFilter } from '@sentry/nestjs/setup';
> providers: [{ provide: APP_FILTER, useClass: SentryGlobalFilter }],
> ```

---

## 8. Métricas Customizadas com OpenTelemetry

Crie um módulo de métricas compartilhado para métricas de negócio:

```typescript
// src/infra/metrics/app-metrics.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { metrics } from '@opentelemetry/api';
import type { Counter, Histogram, UpDownCounter } from '@opentelemetry/api';

@Injectable()
export class AppMetricsService implements OnModuleInit {
  // ── Counters ──────────────────────────────────────────────────────────────
  private httpRequestTotal!: Counter;
  private httpErrorTotal!: Counter;
  private businessEventTotal!: Counter;

  // ── Histograms ────────────────────────────────────────────────────────────
  private httpRequestDuration!: Histogram;
  private dbQueryDuration!: Histogram;

  // ── Gauges ────────────────────────────────────────────────────────────────
  private activeConnections!: UpDownCounter;

  onModuleInit(): void {
    const meter = metrics.getMeter('nestjs-app', process.env.OTEL_SERVICE_VERSION ?? '0.0.0');

    this.httpRequestTotal = meter.createCounter('http_requests_total', {
      description: 'Total de requisições HTTP recebidas',
    });

    this.httpErrorTotal = meter.createCounter('http_errors_total', {
      description: 'Total de erros HTTP (4xx + 5xx)',
    });

    this.httpRequestDuration = meter.createHistogram('http_request_duration_ms', {
      description: 'Duração das requisições HTTP em milissegundos',
      unit: 'ms',
      // Buckets alinhados com SLOs: < 100ms, < 250ms, < 500ms, < 1s, < 2.5s, < 5s, < 10s
      advice: {
        explicitBucketBoundaries: [10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000],
      },
    });

    this.dbQueryDuration = meter.createHistogram('db_query_duration_ms', {
      description: 'Duração das queries ao banco de dados em milissegundos',
      unit: 'ms',
      advice: {
        explicitBucketBoundaries: [1, 5, 10, 25, 50, 100, 250, 500, 1000],
      },
    });

    this.businessEventTotal = meter.createCounter('business_events_total', {
      description: 'Total de eventos de negócio processados',
    });

    this.activeConnections = meter.createUpDownCounter('active_connections', {
      description: 'Número de conexões ativas no momento',
    });
  }

  // ── Métodos públicos ──────────────────────────────────────────────────────

  recordHttpRequest(method: string, route: string, statusCode: number, durationMs: number): void {
    const labels = { method, route, status_code: String(statusCode) };
    this.httpRequestTotal.add(1, labels);
    this.httpRequestDuration.record(durationMs, labels);

    if (statusCode >= 400) {
      this.httpErrorTotal.add(1, { ...labels, error_type: statusCode >= 500 ? '5xx' : '4xx' });
    }
  }

  recordDbQuery(operation: string, entity: string, durationMs: number, success: boolean): void {
    this.dbQueryDuration.record(durationMs, {
      operation,  // select, insert, update, delete
      entity,     // nome da tabela/model
      success: String(success),
    });
  }

  recordBusinessEvent(eventName: string, attributes?: Record<string, string>): void {
    this.businessEventTotal.add(1, { event: eventName, ...attributes });
  }

  trackConnection(delta: 1 | -1): void {
    this.activeConnections.add(delta);
  }
}
```

```typescript
// src/infra/metrics/metrics.module.ts
import { Global, Module } from '@nestjs/common';
import { AppMetricsService } from './app-metrics.service';

@Global()
@Module({
  providers: [AppMetricsService],
  exports: [AppMetricsService],
})
export class MetricsModule {}
```

> Registre `MetricsModule` no `AppModule` — por ser `@Global()`, estará disponível em todos os módulos sem reimportar.

---

## 9. Interceptor de Métricas HTTP

Instrumente automaticamente todas as rotas com um interceptor global:

```typescript
// src/common/interceptors/metrics.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, tap } from 'rxjs';
import { AppMetricsService } from '@infra/metrics/app-metrics.service';

@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  constructor(private readonly metrics: AppMetricsService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const start = Date.now();
    const req   = context.switchToHttp().getRequest<{ method: string; url: string }>();

    return next.handle().pipe(
      tap({
        next: () => {
          const res    = context.switchToHttp().getResponse<{ statusCode: number }>();
          const route  = this.normalizeRoute(req.url);
          this.metrics.recordHttpRequest(req.method, route, res.statusCode, Date.now() - start);
        },
        error: (err: { status?: number }) => {
          const status = err?.status ?? 500;
          const route  = this.normalizeRoute(req.url);
          this.metrics.recordHttpRequest(req.method, route, status, Date.now() - start);
        },
      }),
    );
  }

  // Remove IDs dinâmicos da rota para não explodir cardinalidade do Prometheus
  // Ex: /users/123/orders/456 → /users/:id/orders/:id
  private normalizeRoute(url: string): string {
    return url
      .split('?')[0]                        // remove query string
      .replace(/\/[0-9a-f-]{36}/gi, '/:id') // UUID
      .replace(/\/\d+/g, '/:id');           // números
  }
}
```

Registre globalmente em `app.module.ts`:

```typescript
// src/app.module.ts
import { APP_INTERCEPTOR } from '@nestjs/core';
import { MetricsInterceptor } from '@common/interceptors/metrics.interceptor';

providers: [
  { provide: APP_INTERCEPTOR, useClass: MetricsInterceptor },
  // ... outros providers
],
```

---

## 10. Spans Manuais com OpenTelemetry

Para operações críticas de negócio que precisam de visibilidade além do auto-instrumentation:

```typescript
// Exemplo em um Use Case
import { Injectable } from '@nestjs/common';
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

@Injectable()
export class ProcessPaymentUseCase {
  private readonly tracer = trace.getTracer('payments');

  async execute(dto: ProcessPaymentDto): Promise<PaymentResult> {
    // Cria um span filho do trace atual (propagado automaticamente pelo OTel)
    return this.tracer.startActiveSpan('payment.process', async (span) => {
      try {
        // ✅ Atributos de negócio — visíveis no trace
        span.setAttributes({
          'payment.amount':   dto.amount,
          'payment.currency': dto.currency,
          'payment.method':   dto.method,
          'user.id':          dto.userId,
        });

        const result = await this.paymentGateway.charge(dto);

        span.setAttributes({ 'payment.id': result.id, 'payment.status': result.status });
        span.setStatus({ code: SpanStatusCode.OK });

        return result;
      } catch (error) {
        // ✅ Marca o span como erro — aparece em vermelho no Grafana/Jaeger
        span.setStatus({ code: SpanStatusCode.ERROR, message: (error as Error).message });
        span.recordException(error as Error);
        throw error;
      } finally {
        span.end(); // SEMPRE feche o span
      }
    });
  }
}
```

---

## 11. Propagação de Contexto (Microserviços / HTTP Externo)

O OTel injeta automaticamente o `traceparent` header W3C em chamadas HTTP instrumentadas. Para contextos customizados (filas, eventos):

```typescript
// Injetando contexto em mensagens BullMQ (cf. queue-workers.md)
import { propagation, context } from '@opentelemetry/api';

// ── Producer: serializa contexto na mensagem ──────────────────────────────────
async addJob(data: MyJobData): Promise<void> {
  const carrier: Record<string, string> = {};
  propagation.inject(context.active(), carrier); // injeta traceparent, tracestate

  await this.queue.add('my-job', { ...data, _otelCarrier: carrier });
}

// ── Consumer: restaura contexto do trace original ────────────────────────────
async process(job: Job<MyJobData>): Promise<void> {
  const carrier = job.data._otelCarrier ?? {};
  const parentCtx = propagation.extract(context.active(), carrier);

  // Executa o trabalho dentro do contexto do trace original
  await context.with(parentCtx, async () => {
    // spans criados aqui aparecem como filhos do trace do producer
    await this.handleJob(job.data);
  });
}
```

---

## 12. Configuração Docker Compose — Stack Local

```yaml
# docker-compose.observability.yml
# Use com: docker compose -f docker-compose.yml -f docker-compose.observability.yml up
version: '3.9'

services:
  # ── OpenTelemetry Collector ──────────────────────────────────────────────────
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    ports:
      - "4317:4317"   # gRPC
      - "4318:4318"   # HTTP (usado por este projeto)
    volumes:
      - ./infra/otel/collector.yaml:/etc/otel/config.yaml
    command: ["--config=/etc/otel/config.yaml"]

  # ── Grafana Tempo (backend de traces) ────────────────────────────────────────
  tempo:
    image: grafana/tempo:latest
    ports:
      - "3200:3200"   # HTTP API
    volumes:
      - ./infra/tempo/tempo.yaml:/etc/tempo.yaml
    command: ["-config.file=/etc/tempo.yaml"]

  # ── Prometheus ────────────────────────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./infra/prometheus/rules:/etc/prometheus/rules
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.enable-lifecycle"  # permite reload via HTTP POST

  # ── Grafana ───────────────────────────────────────────────────────────────────
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Admin"
    volumes:
      - ./infra/grafana/datasources:/etc/grafana/provisioning/datasources
      - ./infra/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

```yaml
# infra/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: []  # configure Alertmanager aqui em produção

scrape_configs:
  - job_name: 'nestjs-app'
    static_configs:
      - targets: ['host.docker.internal:9464']  # porta do PrometheusExporter
    metrics_path: /metrics
```

---

## 13. Alertas por SLO — Prometheus Rules

Os SLOs definem o nível de serviço comprometido. Os alertas disparam quando os SLOs são violados:

```yaml
# infra/prometheus/rules/slo-alerts.yml
groups:
  - name: slo_http_availability
    interval: 30s
    rules:
      # ── SLI: taxa de erros 5xx ────────────────────────────────────────────────
      - record: job:http_error_rate:ratio_rate5m
        expr: |
          rate(http_errors_total{error_type="5xx"}[5m])
          /
          rate(http_requests_total[5m])

      # ── Alerta: SLO de disponibilidade violado (> 1% de erros 5xx) ──────────
      - alert: SLOAvailabilityBreach
        expr: job:http_error_rate:ratio_rate5m > 0.01
        for: 5m
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: "SLO de disponibilidade violado"
          description: >
            A taxa de erros 5xx está em {{ $value | humanizePercentage }}
            nos últimos 5 minutos. SLO: < 1%.

      # ── Alerta: Burn Rate acelerado (queima budget de erro rapidamente) ───────
      - alert: SLOErrorBudgetBurnRate
        expr: job:http_error_rate:ratio_rate5m > 0.05
        for: 2m
        labels:
          severity: warning
          slo: availability
        annotations:
          summary: "Budget de erro consumindo rapidamente"
          description: >
            Taxa de erros 5xx em {{ $value | humanizePercentage }}.
            O budget de erro está sendo consumido aceleradamente.

  - name: slo_http_latency
    interval: 30s
    rules:
      # ── SLI: percentil p99 de latência ────────────────────────────────────────
      - record: job:http_request_duration_p99:5m
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_ms_bucket[5m])
          )

      # ── Alerta: p99 > 2s ─────────────────────────────────────────────────────
      - alert: SLOLatencyP99Breach
        expr: job:http_request_duration_p99:5m > 2000
        for: 5m
        labels:
          severity: warning
          slo: latency
        annotations:
          summary: "SLO de latência p99 violado"
          description: >
            O p99 de latência está em {{ $value | humanizeDuration }}ms.
            SLO: < 2000ms.

      # ── Alerta: p99 > 5s — crítico ───────────────────────────────────────────
      - alert: SLOLatencyP99Critical
        expr: job:http_request_duration_p99:5m > 5000
        for: 2m
        labels:
          severity: critical
          slo: latency
        annotations:
          summary: "Degradação severa de latência"
          description: >
            O p99 de latência está em {{ $value | humanizeDuration }}ms.
            Investigação imediata necessária.

  - name: slo_queue_processing
    interval: 30s
    rules:
      # ── Alerta: jobs em falha na fila (cf. queue-workers.md) ────────────────
      - alert: QueueFailedJobsHigh
        expr: |
          increase(bullmq_failed_jobs_total[10m]) > 10
        for: 5m
        labels:
          severity: warning
          slo: reliability
        annotations:
          summary: "Alta taxa de falha em jobs da fila"
          description: >
            {{ $value }} jobs falharam nos últimos 10 minutos
            na fila {{ $labels.queue_name }}.
```

---

## 14. Estrutura de Arquivos

```text
src/
├── infra/
│   ├── tracing/
│   │   └── tracing.ts                ← OTel SDK — importado PRIMEIRO no main.ts
│   ├── sentry/
│   │   └── instrument.ts             ← Sentry.init() — importado SEGUNDO no main.ts
│   └── metrics/
│       ├── app-metrics.service.ts    ← Métricas de negócio (counters, histograms)
│       └── metrics.module.ts         ← @Global() — exposta em todo o projeto
├── common/
│   ├── interceptors/
│   │   └── metrics.interceptor.ts    ← Registra métricas HTTP automaticamente
│   └── filters/
│       └── http-exception.filter.ts  ← @SentryExceptionCaptured() adicionado aqui
└── main.ts                           ← ordem: tracing → sentry → bootstrap
infra/
├── otel/
│   └── collector.yaml                ← OTel Collector config
├── prometheus/
│   ├── prometheus.yml                ← Scrape config
│   └── rules/
│       └── slo-alerts.yml            ← Regras de alerta por SLO
├── grafana/
│   ├── datasources/                  ← Prometheus + Tempo configurados
│   └── dashboards/                   ← Dashboards provisionados
└── tempo/
    └── tempo.yaml                    ← Grafana Tempo config
```

---

## 15. Integração com Skills Existentes

### `logging-pattern.md`

A `PinoInstrumentation` do OTel injeta automaticamente `traceId` e `spanId` em **todos os logs Pino**, permitindo navegar de um log ao trace correspondente no Grafana. Nenhuma configuração adicional é necessária além da importação já feita em `tracing.ts`.

Confirme que o autoLogging do Pino exclui `/metrics`:

```typescript
// src/app.module.ts — já definido em logging-pattern.md, confirme que /metrics está incluído
autoLogging: {
  ignore: (req) => ['/health/live', '/health/ready', '/metrics', '/favicon.ico']
    .some(path => req.url?.startsWith(path) ?? false),
},
```

### `health-checks.md`

O endpoint `/metrics` do PrometheusExporter roda em porta separada (`OTEL_PROMETHEUS_PORT`). **Não** é o mesmo que `/health`. O Prometheus scrapa a porta do PrometheusExporter, não a porta da aplicação NestJS.

### `queue-workers.md`

Use a propagação de contexto da seção 11 para conectar traces de producers BullMQ aos traces de consumers, mantendo a rastreabilidade ponta-a-ponta.

### `security-hardening.md`

O endpoint `/metrics` **não deve ser exposto publicamente**. Configure o Prometheus para scrape apenas em rede interna, ou proteja com autenticação básica:

```typescript
// Se necessário expor /metrics na porta da aplicação (não recomendado),
// use @SkipThrottle() e proteja com BasicAuth ou IP allowlist
```

---

## 16. Checklist de Revisão

### OpenTelemetry
- [ ] `tracing.ts` é o **primeiro** import no `main.ts` (antes de qualquer módulo NestJS)?
- [ ] `OTEL_SERVICE_NAME` e `OTEL_SERVICE_VERSION` configurados e validados no Zod?
- [ ] `PinoInstrumentation` configurada — `traceId` aparece nos logs?
- [ ] Health checks e `/metrics` excluídos do `ignoreIncomingRequestHook`?
- [ ] `BatchSpanProcessor` usado em produção (não `SimpleSpanProcessor`)?
- [ ] Shutdown gracioso configurado no `process.on('SIGTERM')`?
- [ ] `@opentelemetry/instrumentation-fs` desabilitado (muito verboso)?

### Prometheus
- [ ] `PrometheusExporter` expondo métricas em porta separada (`OTEL_PROMETHEUS_PORT`)?
- [ ] `MetricsInterceptor` registrado globalmente no `AppModule`?
- [ ] Rota normalizada no interceptor (sem IDs dinâmicos)?
- [ ] `prometheus.yml` com scrape_interval configurado?
- [ ] Regras de alerta SLO criadas em `infra/prometheus/rules/`?

### Sentry
- [ ] `instrument.ts` é o **segundo** import no `main.ts`?
- [ ] `SENTRY_DSN` opcional — aplicação funciona sem ele (desenvolvimento)?
- [ ] `@SentryExceptionCaptured()` aplicado no `HttpExceptionFilter` existente?
- [ ] Erros 4xx filtrados no `beforeSend` (não são bugs)?
- [ ] Dados sensíveis removidos no `beforeSend` (LGPD)?
- [ ] `ignoreTransactions` configurado para health checks e `/metrics`?
- [ ] `SENTRY_TRACES_SAMPLE_RATE` reduzido em produção (0.1 a 0.2)?

### SLO / Alertas
- [ ] Pelo menos 2 SLOs definidos: disponibilidade (error rate) e latência (p99)?
- [ ] Regras Prometheus com `for` > 0 (evitar alertas de flapping)?
- [ ] Labels `severity` e `slo` presentes em todos os alertas?
- [ ] Burn rate configurado para alertas antecipados de esgotamento de budget?

---

## Referências

- [logging-pattern.md](./logging-pattern.md) — Pino, correlationId, integração com OTel
- [health-checks.md](./health-checks.md) — `/health/live`, `/health/ready`, graceful shutdown
- [config-management.md](../configuration-and-environment/config-management.md) — `env.config.ts`, Zod, variáveis de ambiente
- [rest-api-patterns.md](../api-and-contracts/rest-api-patterns.md) — `HttpExceptionFilter`, `main.ts`
- [queue-workers.md](../performance-and-infrastructure/queue-workers.md) — BullMQ, propagação de contexto OTel
- [security-hardening.md](../authentication-and-security/security-hardening.md) — `@SkipThrottle()`, proteção de endpoints internos
- [OpenTelemetry NestJS — SigNoz](https://signoz.io/blog/opentelemetry-nestjs/)
- [Sentry NestJS SDK — Documentação Oficial](https://docs.sentry.io/platforms/javascript/guides/nestjs/)
- [nestjs-otel — GitHub (pragmaticivan)](https://github.com/pragmaticivan/nestjs-otel)
- [OpenTelemetry Best Practices — Better Stack](https://betterstack.com/community/guides/observability/opentelemetry-best-practices/)
- [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Grafana Tempo — Distributed Tracing](https://grafana.com/oss/tempo/)
- [SLO Implementation Guide — Google SRE Book](https://sre.google/workbook/implementing-slos/)