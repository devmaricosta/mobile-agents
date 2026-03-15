---
name: queue-workers
description: Processamento assíncrono com BullMQ (@nestjs/bullmq): estrutura de filas por domínio alinhada ao nest-project-structure.md, producers via @InjectQueue, processors com WorkerHost, retry com backoff exponencial e jitter, Dead Letter Queue (DLQ), monitoramento com Bull Board (@bull-board/nestjs), graceful shutdown, e decisão de quando usar filas vs eventos síncronos. Use esta skill sempre que o projeto envolver tarefas assíncronas em background, envio de e-mails, processamento de arquivos, notificações, jobs agendados/recorrentes, ou qualquer operação que não deva bloquear a resposta HTTP.
---

# Processamento Assíncrono — BullMQ + NestJS

Você é um Engenheiro de Software Senior especialista em sistemas assíncronos e filas de mensagens com NestJS. Esta skill define o **padrão completo de filas e workers** para o projeto, consistente com a arquitetura definida em `nest-project-structure.md`, `clean-architecture.md`, `caching-strategy.md` e `config-management.md`.

> **Decisões de arquitetura (não altere sem revisão):**
> - Pacote: `@nestjs/bullmq` + `bullmq` — **não** use `@nestjs/bull` (legado, modo manutenção).
> - Redis compartilhado com `caching-strategy.md` — mesma instância Redis, prefixos distintos de chave.
> - Processors estendem `WorkerHost` — padrão oficial NestJS, permite DI completo.
> - DLQ via fila separada com sufixo `-dlq` — jobs "veneno" são redirecionados após esgotar tentativas.
> - Bull Board protegido com `basicAuth` — nunca exponha sem autenticação em staging/produção.
> - Graceful shutdown via `onModuleDestroy` — nunca use SIGKILL para encerrar workers.
> - `removeOnComplete` e `removeOnFail` obrigatórios — evitam inchaço de memória no Redis.

---

## 1. Quando Usar Filas vs. Eventos Síncronos

### Tabela de decisão rápida

| Cenário | Usar fila? | Alternativa |
|---------|-----------|-------------|
| Enviar e-mail de boas-vindas após cadastro | ✅ Sim | — |
| Processar upload de imagem/vídeo | ✅ Sim | — |
| Gerar relatório PDF pesado | ✅ Sim | — |
| Notificação push para múltiplos usuários | ✅ Sim | — |
| Sincronizar dados com serviço externo lento | ✅ Sim | — |
| Job recorrente (cron diário, semanal) | ✅ Sim | `@nestjs/schedule` para cron simples sem escala horizontal |
| Atualizar `updatedAt` após salvar entidade | ❌ Não | Lógica síncrona no UseCase |
| Validar CPF em API externa (< 200ms) | ❌ Não | Chamada síncrona no UseCase |
| Publicar evento de domínio interno | ❌ Não | `EventEmitter2` do `@nestjs/event-emitter` |
| Tarefa < 50ms sem risco de falha externa | ❌ Não | Execução síncrona direta |

### Regra prática
Use filas quando: (1) a operação pode demorar mais de ~200ms, (2) depende de serviços externos sujeitos a falha, (3) precisa de retry automático, (4) não deve bloquear a resposta HTTP para o cliente, ou (5) precisa ser distribuída entre múltiplas instâncias.

---

## 2. Instalação

```bash
npm install @nestjs/bullmq bullmq
npm install @bull-board/nestjs @bull-board/api @bull-board/express
npm install express-basic-auth
```

> **Nota:** `ioredis` já está instalado via `caching-strategy.md` (`@keyv/redis` depende dele). O `BullModule` do NestJS aceita a URL do Redis diretamente — compartilhe a mesma URL da variável `REDIS_URL`.

---

## 3. Variáveis de Ambiente

Complementa `config-management.md` — adicione ao `env.config.ts`:

```typescript
// src/config/env.config.ts — adicione ao schema existente:
const envSchema = z.object({
  // ... variáveis existentes (REDIS_URL já definida em caching-strategy.md)
  BULL_BOARD_USER:     z.string().default('admin'),
  BULL_BOARD_PASSWORD: z.string().min(8),
  QUEUE_DEFAULT_ATTEMPTS: z.coerce.number().default(4),
  QUEUE_DEFAULT_BACKOFF_DELAY_MS: z.coerce.number().default(3000),
});
```

---

## 4. Configuração Global — `QueueInfraModule`

### 4.1 `queue.config.ts`

```typescript
// src/config/queue.config.ts
import { registerAs } from '@nestjs/config';
import { env } from './env.config';

export interface QueueConfig {
  redisUrl: string;
  defaultAttempts: number;
  defaultBackoffDelayMs: number;
  bullBoardUser: string;
  bullBoardPassword: string;
}

export default registerAs('queue', (): QueueConfig => ({
  redisUrl:              env.REDIS_URL ?? 'redis://localhost:6379',
  defaultAttempts:       env.QUEUE_DEFAULT_ATTEMPTS,
  defaultBackoffDelayMs: env.QUEUE_DEFAULT_BACKOFF_DELAY_MS,
  bullBoardUser:         env.BULL_BOARD_USER,
  bullBoardPassword:     env.BULL_BOARD_PASSWORD,
}));
```

### 4.2 `QueueInfraModule` — Módulo Global de Infraestrutura

```typescript
// src/infra/queue/queue-infra.module.ts
import { Global, Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { BullBoardModule } from '@bull-board/nestjs';
import { ExpressAdapter } from '@bull-board/express';
import { ConfigModule, ConfigType } from '@nestjs/config';
import basicAuth from 'express-basic-auth';
import queueConfig from '@config/queue.config';

@Global()
@Module({
  imports: [
    // ← Configuração global da conexão Redis para TODAS as filas
    BullModule.forRootAsync({
      imports: [ConfigModule],
      inject: [queueConfig.KEY],
      useFactory: (config: ConfigType<typeof queueConfig>) => ({
        connection: { url: config.redisUrl },
        defaultJobOptions: {
          attempts:        config.defaultAttempts,
          backoff: {
            type:  'exponential',
            delay: config.defaultBackoffDelayMs,
            jitter: 0.3,     // ← ±30% de ruído — evita thundering herd
          },
          removeOnComplete: { age: 3600,      count: 1000 },  // 1h ou 1k jobs
          removeOnFail:     { age: 24 * 3600, count: 2000 },  // 24h ou 2k jobs
        },
      }),
    }),

    // ← Bull Board — dashboard de monitoramento
    BullBoardModule.forRootAsync({
      imports: [ConfigModule],
      inject: [queueConfig.KEY],
      useFactory: (config: ConfigType<typeof queueConfig>) => ({
        route: '/admin/queues',
        adapter: ExpressAdapter,
        // Autenticação básica — OBRIGATÓRIA em ambientes não-locais
        middleware: basicAuth({
          challenge: true,
          users: { [config.bullBoardUser]: config.bullBoardPassword },
        }),
      }),
    }),
  ],
})
export class QueueInfraModule {}
```

### 4.3 Registro no `AppModule`

```typescript
// src/app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [queueConfig, cacheConfig, databaseConfig] }),
    DatabaseModule,
    AppCacheModule,
    QueueInfraModule,   // ← APÓS AppCacheModule, ANTES dos módulos de domínio
    AuthModule,
    NotificationsModule,
    // ... demais módulos de domínio
  ],
})
export class AppModule {}
```

---

## 5. Estrutura de Pastas por Domínio

Seguindo `nest-project-structure.md`, cada domínio com filas fica autocontido:

```
src/
├── infra/
│   └── queue/
│       └── queue-infra.module.ts    ← @Global, BullModule.forRootAsync + BullBoard
│
└── modules/
    └── notifications/               ← exemplo de domínio com fila
        ├── controllers/
        │   └── notifications.controller.ts
        ├── use-cases/
        │   └── send-notification/
        │       └── send-notification.use-case.ts   ← enfileira job
        ├── queues/
        │   ├── notifications-queue.constants.ts    ← nomes de fila e job (enums)
        │   ├── notifications.producer.ts           ← @InjectQueue, métodos add*()
        │   ├── notifications.processor.ts          ← WorkerHost, process()
        │   └── notifications-dlq.processor.ts      ← WorkerHost da DLQ
        ├── __tests__/
        │   └── notifications.processor.spec.ts
        └── notifications.module.ts
```

> **Regra:** A pasta `queues/` fica dentro do módulo de domínio, não em `src/infra/`. Infraestrutura (`BullModule.forRoot`) fica em `src/infra/queue/`; lógica de domínio (producers, processors) fica em `src/modules/<domínio>/queues/`.

---

## 6. Constants — Nomes de Fila e Job

```typescript
// src/modules/notifications/queues/notifications-queue.constants.ts

/**
 * Nomes das filas do domínio de notificações.
 * Convenção: kebab-case, prefixo do domínio, sufixo -dlq para Dead Letter Queue.
 */
export enum NotificationQueueName {
  MAIN = 'notifications',
  DLQ  = 'notifications-dlq',
}

/**
 * Nomes dos jobs por tipo de operação.
 * Use strings descritivas — ficam visíveis no Bull Board e nos logs.
 */
export enum NotificationJobName {
  SEND_EMAIL        = 'send-email',
  SEND_PUSH         = 'send-push',
  SEND_SMS          = 'send-sms',
  DIGEST_DAILY      = 'digest-daily',  // job recorrente
  // DLQ
  DEAD_LETTER       = 'dead-letter',
}
```

---

## 7. Producer — Enfileiramento de Jobs

O Producer é um `@Injectable()` simples que encapsula o `Queue` e expõe métodos tipados:

```typescript
// src/modules/notifications/queues/notifications.producer.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue, JobsOptions } from 'bullmq';
import { NotificationQueueName, NotificationJobName } from './notifications-queue.constants';

export interface SendEmailJobData {
  userId:  string;
  to:      string;
  subject: string;
  template: string;
  context: Record<string, unknown>;
}

export interface SendPushJobData {
  userId:   string;
  deviceTokens: string[];
  title:    string;
  body:     string;
}

@Injectable()
export class NotificationsProducer {
  private readonly logger = new Logger(NotificationsProducer.name);

  constructor(
    @InjectQueue(NotificationQueueName.MAIN)
    private readonly queue: Queue,
  ) {}

  /**
   * Enfileira e-mail transacional.
   * jobId baseado em userId + template garante idempotência (evita duplicatas).
   */
  async addSendEmail(data: SendEmailJobData, opts?: JobsOptions): Promise<void> {
    const jobId = `email:${data.template}:${data.userId}`;
    await this.queue.add(NotificationJobName.SEND_EMAIL, data, {
      jobId,
      // Sobrescreve defaults globais se necessário
      attempts: 5,
      backoff: { type: 'exponential', delay: 2000, jitter: 0.3 },
      ...opts,
    });
    this.logger.debug(`Job enfileirado: ${NotificationJobName.SEND_EMAIL} (jobId=${jobId})`);
  }

  /**
   * Enfileira push notification com delay opcional (ex: lembrete futuro).
   */
  async addSendPush(data: SendPushJobData, delayMs?: number): Promise<void> {
    await this.queue.add(NotificationJobName.SEND_PUSH, data, {
      delay: delayMs,
    });
  }

  /**
   * Registra job recorrente de digest diário (idempotente via repeatJobKey).
   * Chame apenas uma vez na inicialização (ex: em onModuleInit do módulo).
   */
  async scheduleDigest(): Promise<void> {
    await this.queue.add(
      NotificationJobName.DIGEST_DAILY,
      {},
      {
        repeat: { pattern: '0 9 * * *' },  // cron: todo dia às 09:00
        jobId: 'digest-daily-recurring',    // ID estável para não criar duplicatas
      },
    );
  }
}
```

> **Regra:** O UseCase injeta o Producer, não o `Queue` diretamente. O Producer é o único ponto de contato com a fila — mantém os nomes de job centralizados e tipados.

---

## 8. Processor — WorkerHost (Consumidor)

```typescript
// src/modules/notifications/queues/notifications.processor.ts
import { Logger } from '@nestjs/common';
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { NotificationQueueName, NotificationJobName } from './notifications-queue.constants';
import { NotificationQueueName as Q, NotificationJobName as JN } from './notifications-queue.constants';
import { MailerService } from '@infra/mailer/mailer.service';
import { PushService } from '@infra/push/push.service';
import { NotificationsProducer, SendEmailJobData, SendPushJobData } from './notifications.producer';

@Processor(NotificationQueueName.MAIN, {
  concurrency: 5,   // ← processa até 5 jobs em paralelo (ajuste por tipo de carga)
})
export class NotificationsProcessor extends WorkerHost {
  private readonly logger = new Logger(NotificationsProcessor.name);

  constructor(
    private readonly mailerService: MailerService,
    private readonly pushService: PushService,
    private readonly producer: NotificationsProducer,
  ) {
    super();
  }

  /**
   * Dispatch por nome de job — padrão recomendado para múltiplos tipos.
   */
  async process(job: Job): Promise<unknown> {
    this.logger.log(`Processando job "${job.name}" (id=${job.id}, attempt=${job.attemptsMade + 1})`);

    switch (job.name) {
      case NotificationJobName.SEND_EMAIL:
        return this.handleSendEmail(job as Job<SendEmailJobData>);

      case NotificationJobName.SEND_PUSH:
        return this.handleSendPush(job as Job<SendPushJobData>);

      case NotificationJobName.DIGEST_DAILY:
        return this.handleDigest(job);

      default:
        throw new Error(`Job desconhecido: ${job.name}`);
    }
  }

  // ─── Handlers privados ───────────────────────────────────────────────────

  private async handleSendEmail(job: Job<SendEmailJobData>): Promise<void> {
    const { to, subject, template, context } = job.data;
    await this.mailerService.send({ to, subject, template, context });
  }

  private async handleSendPush(job: Job<SendPushJobData>): Promise<void> {
    const { deviceTokens, title, body } = job.data;
    await this.pushService.send({ deviceTokens, title, body });
  }

  private async handleDigest(job: Job): Promise<void> {
    this.logger.log('Executando digest diário...');
    // lógica de digest
  }

  // ─── Eventos do Worker ───────────────────────────────────────────────────

  @OnWorkerEvent('completed')
  onCompleted(job: Job): void {
    this.logger.log(`✅ Job concluído: "${job.name}" (id=${job.id})`);
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job | undefined, error: Error): void {
    if (!job) return;

    const isLastAttempt = job.attemptsMade >= (job.opts.attempts ?? 1) - 1;

    this.logger.error(
      `❌ Job falhou: "${job.name}" (id=${job.id}, attempt=${job.attemptsMade + 1}/${job.opts.attempts}) — ${error.message}`,
    );

    // Redireciona para DLQ na última tentativa
    if (isLastAttempt) {
      this.producer
        .addDeadLetter({ originalJobName: job.name, originalJobData: job.data, error: error.message })
        .catch((err) => this.logger.error(`Falha ao mover para DLQ: ${err.message}`));
    }
  }

  @OnWorkerEvent('stalled')
  onStalled(jobId: string): void {
    this.logger.warn(`⚠️ Job travado (stalled): id=${jobId}`);
  }
}
```

---

## 9. Dead Letter Queue (DLQ)

### 9.1 Adicionar método `addDeadLetter` no Producer

```typescript
// src/modules/notifications/queues/notifications.producer.ts — adicionar:

export interface DeadLetterJobData {
  originalJobName: string;
  originalJobData: unknown;
  error: string;
}

// No NotificationsProducer, injetar também a DLQ:
constructor(
  @InjectQueue(NotificationQueueName.MAIN) private readonly queue: Queue,
  @InjectQueue(NotificationQueueName.DLQ)  private readonly dlqQueue: Queue,
) {}

async addDeadLetter(data: DeadLetterJobData): Promise<void> {
  await this.dlqQueue.add(NotificationJobName.DEAD_LETTER, data, {
    removeOnComplete: true,         // mantém DLQ limpa após inspeção
    removeOnFail:     { age: 7 * 24 * 3600 },  // mantém falhas por 7 dias
  });
}
```

### 9.2 DLQ Processor

```typescript
// src/modules/notifications/queues/notifications-dlq.processor.ts
import { Logger } from '@nestjs/common';
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { NotificationQueueName } from './notifications-queue.constants';
import { DeadLetterJobData } from './notifications.producer';

@Processor(NotificationQueueName.DLQ, {
  concurrency: 1,   // DLQ processa devagar — prioridade baixa
})
export class NotificationsDlqProcessor extends WorkerHost {
  private readonly logger = new Logger(NotificationsDlqProcessor.name);

  async process(job: Job<DeadLetterJobData>): Promise<void> {
    const { originalJobName, originalJobData, error } = job.data;

    // 1. Log estruturado para alertas (Sentry, Datadog, etc.)
    this.logger.error(
      `💀 DLQ — job "${originalJobName}" falhou permanentemente`,
      { originalJobData, error },
    );

    // 2. Aqui: notificar equipe de ops, salvar em tabela de auditoria, etc.
    // Ex: await this.alertService.notifyOps({ jobName: originalJobName, error });
  }
}
```

---

## 10. Módulo do Domínio

```typescript
// src/modules/notifications/notifications.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { BullBoardModule } from '@bull-board/nestjs';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter';
import { NotificationsController } from './controllers/notifications.controller';
import { SendNotificationUseCase } from './use-cases/send-notification/send-notification.use-case';
import { NotificationsProducer } from './queues/notifications.producer';
import { NotificationsProcessor } from './queues/notifications.processor';
import { NotificationsDlqProcessor } from './queues/notifications-dlq.processor';
import { NotificationQueueName } from './queues/notifications-queue.constants';

@Module({
  imports: [
    // ← Registra as filas do domínio (herda config Redis global do QueueInfraModule)
    BullModule.registerQueue(
      {
        name: NotificationQueueName.MAIN,
        defaultJobOptions: {
          attempts: 4,
          backoff: { type: 'exponential', delay: 3000, jitter: 0.3 },
          removeOnComplete: { age: 1800, count: 500 },
          removeOnFail:     { age: 24 * 3600, count: 1000 },
        },
      },
      {
        name: NotificationQueueName.DLQ,
        defaultJobOptions: {
          removeOnComplete: true,
          removeOnFail:     { age: 7 * 24 * 3600 },
        },
      },
    ),

    // ← Registra as filas no Bull Board (co-localizado com BullModule.registerQueue)
    BullBoardModule.forFeature(
      { name: NotificationQueueName.MAIN, adapter: BullMQAdapter },
      { name: NotificationQueueName.DLQ,  adapter: BullMQAdapter },
    ),
  ],
  controllers: [NotificationsController],
  providers: [
    SendNotificationUseCase,
    NotificationsProducer,
    NotificationsProcessor,
    NotificationsDlqProcessor,
  ],
  exports: [NotificationsProducer], // ← exporta para outros módulos enfileirarem
})
export class NotificationsModule {}
```

---

## 11. UseCase — Enfileirando a Partir da Camada de Aplicação

```typescript
// src/modules/notifications/use-cases/send-notification/send-notification.use-case.ts
import { Injectable } from '@nestjs/common';
import { NotificationsProducer } from '@modules/notifications/queues/notifications.producer';

export interface SendNotificationInput {
  userId:   string;
  to:       string;
  template: string;
  context:  Record<string, unknown>;
}

@Injectable()
export class SendNotificationUseCase {
  constructor(private readonly producer: NotificationsProducer) {}

  async execute(input: SendNotificationInput): Promise<void> {
    await this.producer.addSendEmail({
      userId:   input.userId,
      to:       input.to,
      subject:  'Sua notificação',
      template: input.template,
      context:  input.context,
    });
  }
}
```

> **Regra de Clean Architecture:** O UseCase conhece apenas o Producer (camada de aplicação). O Producer conhece o `Queue` (infraestrutura). O Processor (worker) conhece os serviços de infraestrutura que executa a tarefa. Nenhuma dessas camadas conhece HTTP.

---

## 12. Retry com Backoff Exponencial + Jitter

### Como funciona o backoff exponencial

Com `delay: 3000` e `jitter: 0.3`, os delays entre tentativas são (aproximadamente):

| Tentativa | Fórmula base | Com jitter (±30%) |
|-----------|-------------|-------------------|
| 1ª retry  | 3s          | 2.1s — 3.9s       |
| 2ª retry  | 6s          | 4.2s — 7.8s       |
| 3ª retry  | 12s         | 8.4s — 15.6s      |
| 4ª retry  | 24s         | 16.8s — 31.2s     |

O jitter evita o "thundering herd" — múltiplos workers não retentam ao mesmo tempo após uma falha de serviço externo.

### Erros permanentes vs. temporários

```typescript
// No processor, diferencie erros para evitar retries desnecessários:

private async handleSendEmail(job: Job<SendEmailJobData>): Promise<void> {
  try {
    await this.mailerService.send(job.data);
  } catch (error) {
    // Erros de negócio (ex: e-mail inválido) → não faz sentido retentar
    if (error instanceof InvalidEmailError) {
      // Lança UnrecoverableError para ir direto à DLQ sem usar tentativas restantes
      throw new UnrecoverableError(`E-mail inválido: ${error.message}`);
    }
    // Outros erros (timeout, rate limit) → BullMQ retenta automaticamente
    throw error;
  }
}
```

> Importe `UnrecoverableError` de `bullmq` para marcar erros que não devem ser retentados.

---

## 13. Graceful Shutdown

O `@nestjs/bullmq` fecha os workers automaticamente no `onModuleDestroy` do NestJS. Garanta que a aplicação não receba SIGKILL antes de terminar os jobs em andamento:

```typescript
// src/main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // ← Permite que workers terminem jobs em andamento antes de encerrar
  app.enableShutdownHooks();

  await app.listen(3000);
}
```

```dockerfile
# Dockerfile — use SIGTERM (não SIGKILL) para graceful shutdown
STOPSIGNAL SIGTERM
```

---

## 14. Bull Board — Monitoramento

Após o setup do `QueueInfraModule`, o dashboard estará disponível em:

```
http://localhost:3000/admin/queues
```

**O que monitorar no Bull Board:**
- **Waiting**: jobs aguardando worker disponível — pico indica necessidade de escalar workers
- **Active**: jobs em processamento agora
- **Completed**: jobs finalizados com sucesso (limpos por TTL/count)
- **Failed**: jobs com todas tentativas esgotadas — investigar imediatamente
- **Delayed**: jobs agendados para o futuro (recorrentes, com `delay`)
- **Paused**: fila pausada manualmente

**Ações disponíveis:** retry de jobs falhos, remoção, inspeção do payload, limpeza em lote.

> **Segurança:** Em produção, o Bull Board deve ficar atrás de VPN ou ter IP allowlist além do `basicAuth`. Considere `readOnlyMode: true` para usuários não-administradores.

---

## 15. Testes

### 15.1 Teste Unitário do Processor

```typescript
// src/modules/notifications/queues/notifications.processor.spec.ts
import { Test } from '@nestjs/testing';
import { NotificationsProcessor } from './notifications.processor';
import { NotificationsProducer } from './notifications.producer';
import { MailerService } from '@infra/mailer/mailer.service';
import { NotificationJobName } from './notifications-queue.constants';

describe('NotificationsProcessor', () => {
  let processor: NotificationsProcessor;
  let mailerService: jest.Mocked<MailerService>;
  let producer: jest.Mocked<NotificationsProducer>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        NotificationsProcessor,
        { provide: MailerService,          useValue: { send: jest.fn() } },
        { provide: PushService,            useValue: { send: jest.fn() } },
        { provide: NotificationsProducer,  useValue: { addDeadLetter: jest.fn() } },
      ],
    }).compile();

    processor     = module.get(NotificationsProcessor);
    mailerService = module.get(MailerService);
    producer      = module.get(NotificationsProducer);
  });

  afterEach(() => jest.clearAllMocks());

  it('deve chamar mailerService.send no job send-email', async () => {
    const jobData = { userId: 'u1', to: 'test@test.com', subject: 'Test', template: 'welcome', context: {} };
    const job = { name: NotificationJobName.SEND_EMAIL, data: jobData, opts: { attempts: 4 }, attemptsMade: 0 } as any;

    await processor.process(job);

    expect(mailerService.send).toHaveBeenCalledWith(
      expect.objectContaining({ to: 'test@test.com', template: 'welcome' }),
    );
  });

  it('deve enfileirar na DLQ na última tentativa falha', async () => {
    const job = {
      name: NotificationJobName.SEND_EMAIL,
      data: {},
      id: 'job-1',
      opts: { attempts: 4 },
      attemptsMade: 3, // última tentativa (0-indexed)
    } as any;

    const error = new Error('SMTP timeout');
    processor.onFailed(job, error);

    expect(producer.addDeadLetter).toHaveBeenCalledWith(
      expect.objectContaining({ originalJobName: NotificationJobName.SEND_EMAIL, error: 'SMTP timeout' }),
    );
  });

  it('deve lançar erro para job desconhecido', async () => {
    const job = { name: 'job-inexistente', data: {} } as any;
    await expect(processor.process(job)).rejects.toThrow('Job desconhecido');
  });
});
```

### 15.2 Teste do Producer (mock da Queue)

```typescript
describe('NotificationsProducer', () => {
  let producer: NotificationsProducer;
  let mainQueue: jest.Mocked<Queue>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        NotificationsProducer,
        { provide: getQueueToken(NotificationQueueName.MAIN), useValue: { add: jest.fn() } },
        { provide: getQueueToken(NotificationQueueName.DLQ),  useValue: { add: jest.fn() } },
      ],
    }).compile();

    producer  = module.get(NotificationsProducer);
    mainQueue = module.get(getQueueToken(NotificationQueueName.MAIN));
  });

  it('deve chamar queue.add com jobId de idempotência', async () => {
    const data = { userId: 'u1', to: 'a@b.com', subject: 'Test', template: 'welcome', context: {} };
    await producer.addSendEmail(data);
    expect(mainQueue.add).toHaveBeenCalledWith(
      NotificationJobName.SEND_EMAIL,
      data,
      expect.objectContaining({ jobId: 'email:welcome:u1' }),
    );
  });
});
```

> Use `getQueueToken` importado de `@nestjs/bullmq` para injetar o mock da Queue pelo token correto.

---

## 16. Checklist de Implementação

### Configuração Base
- [ ] `QueueInfraModule` criado em `src/infra/queue/` com `@Global()`?
- [ ] `BullModule.forRootAsync` consome `queueConfig` via `ConfigType`?
- [ ] `BULL_BOARD_USER` e `BULL_BOARD_PASSWORD` no `.env` com valores seguros?
- [ ] `app.enableShutdownHooks()` em `main.ts`?

### Por Domínio
- [ ] `*-queue.constants.ts` com enums `QueueName` e `JobName`?
- [ ] Producer injeta `@InjectQueue` e expõe métodos tipados (não expõe `Queue` diretamente)?
- [ ] `jobId` de idempotência definido nos jobs críticos?
- [ ] Processor estende `WorkerHost` e implementa `process()` com dispatch por `job.name`?
- [ ] `UnrecoverableError` lançado para erros permanentes (não retentar)?
- [ ] DLQ registrada no módulo (`NotificationQueueName.DLQ`)?
- [ ] `onFailed` redireciona para DLQ na última tentativa?
- [ ] `BullBoardModule.forFeature` registrado para cada fila no módulo?
- [ ] Producer exportado no módulo para uso por outros domínios?

### Produção
- [ ] `removeOnComplete` e `removeOnFail` configurados (evitar inchaço Redis)?
- [ ] `jitter` no backoff configurado (evitar thundering herd)?
- [ ] Bull Board com `basicAuth` + IP allowlist / VPN em staging e produção?
- [ ] SIGTERM (não SIGKILL) no Dockerfile?
- [ ] Alertas configurados para jobs na fila de falhas (ex: webhook Sentry)?

### Testes
- [ ] Processor testado com mocks de serviços de infraestrutura?
- [ ] Cenário de `UnrecoverableError` testado?
- [ ] Redirecionamento para DLQ testado no `onFailed`?
- [ ] Producer testado com `getQueueToken` do `@nestjs/bullmq`?

---

## Referências

- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura `src/infra/`, `src/modules/<domínio>/`, path aliases
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — UseCase → Producer → Queue; Processor → Service de infraestrutura
- [caching-strategy.md](.agent/skills/backend/performance-and-infrastructure/caching-strategy.md) — Redis compartilhado (`REDIS_URL`), `AppCacheModule`
- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — `registerAs`, `ConfigType`, variáveis de ambiente
- [testing-strategy-nest.md](.agent/skills/backend/quality-and-testing/testing-strategy-nest.md) — Pirâmide de testes, mocks com `Test.createTestingModule`
- [NestJS Docs — Queues](https://docs.nestjs.com/techniques/queues)
- [BullMQ Docs — Retrying Failing Jobs](https://docs.bullmq.io/guide/retrying-failing-jobs)
- [BullMQ Docs — Workers Concurrency](https://docs.bullmq.io/guide/workers/concurrency)
- [Bull Board — @bull-board/nestjs](https://www.npmjs.com/package/@bull-board/nestjs)