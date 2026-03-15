---
name: logging-pattern
description: Logging estruturado no NestJS com Pino (nestjs-pino): correlation ID automático por request via AsyncLocalStorage, log levels por ambiente, redação de dados sensíveis (PII/LGPD), o que logar e o que NUNCA logar, integração com Datadog e AWS CloudWatch. Use quando configurar logging do zero, adicionar correlation ID, integrar com ferramentas de observabilidade, ou revisar conformidade com LGPD/GDPR em logs.
---

# Logging Estruturado — NestJS com Pino

Você é um Engenheiro de Software Senior especialista em observabilidade de sistemas backend. Sua responsabilidade é garantir que o projeto tenha **logs estruturados, rastreáveis e seguros em produção**, usando as ferramentas corretas, seguindo as melhores práticas do mercado e mantendo conformidade com **LGPD/GDPR**.

> **Pré-requisitos:** Esta skill assume que as skills `config-management`, `nest-project-structure` e `rest-api-patterns` já foram aplicadas. O objeto `env` de `@config/env.config` é a fonte da verdade para variáveis de ambiente. O `HttpExceptionFilter` global de `rest-api-patterns` é o ponto onde erros HTTP são logados centralizadamente.

---

## Quando usar esta skill

- **Novo projeto:** Configurar o sistema de logging do zero.
- **Correlation ID:** Rastrear requisições ponta-a-ponta através de múltiplos serviços.
- **Auditoria de segurança:** Verificar se dados sensíveis estão vazando em logs.
- **Integração com Datadog / CloudWatch:** Conectar logs à plataforma de observabilidade.
- **Code review:** Validar se os padrões de logging estão sendo seguidos.
- **LGPD / GDPR:** Garantir conformidade com regulações de proteção de dados.

---

## 1. Por que Pino e não Winston

| Critério | Pino | Winston |
|---|---|---|
| Performance | ~5x mais rápido (async JSON, worker thread) | Síncrono por padrão |
| JSON nativo | Sim — sem configuração extra | Requer formatters |
| Integração NestJS | `nestjs-pino` (oficial da comunidade) | `nest-winston` |
| Correlation ID automático | Via `AsyncLocalStorage` (built-in) | Requer middleware manual |
| Redação nativa | `redact` no construtor | Requer plugin externo |
| Uso em produção | Alto tráfego (Fastify, etc.) | Projetos com muitos transports |

**Decisão:** Use **Pino** via `nestjs-pino` como padrão. O `nestjs-pino` usa `AsyncLocalStorage` internamente para propagar o contexto da requisição (incluindo `correlationId`) para qualquer service em qualquer camada, sem passar o logger manualmente.

> **Quando usar Winston:** Apenas se o projeto já tiver Winston estabelecido, ou se precisar de múltiplos transports simultâneos complexos (e.g., arquivo + Slack + BigQuery) sem uma camada de agente externo (Fluent Bit, Logstash).

---

## 2. Instalação

```bash
npm install nestjs-pino pino-http
npm install --save-dev pino-pretty
```

> `pino-pretty` é apenas para desenvolvimento — nunca use em produção, pois desativa o formato JSON.

---

## 3. Variáveis de Ambiente

Adicione ao `env.config.ts` (cf. `config-management.md`):

```typescript
// src/config/env.config.ts — adicionar ao schema Zod existente
const envSchema = z.object({
  // ... variáveis existentes ...

  // Logging
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),
  LOG_PRETTY: z.coerce.boolean().default(false), // true apenas em desenvolvimento local
});
```

```bash
# .env.example — adicionar
LOG_LEVEL=info
LOG_PRETTY=false
```

```bash
# .env.development — desenvolvimento local
LOG_LEVEL=debug
LOG_PRETTY=true
```

---

## 4. Configuração do `LoggerModule` no `AppModule`

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { LoggerModule } from 'nestjs-pino';
import { ConfigModule } from '@nestjs/config';
import { env } from '@config/env.config';
import { v4 as uuidv4 } from 'uuid';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    LoggerModule.forRoot({
      pinoHttp: {
        // ── Nível de log por ambiente ──────────────────────────────────────
        level: env.LOG_LEVEL,

        // ── Correlation ID: lê do header ou gera um novo ──────────────────
        // O header X-Correlation-ID permite que o cliente (ou API Gateway)
        // propague o mesmo ID entre serviços. Se ausente, geramos um UUID v4.
        genReqId: (req) =>
          (req.headers['x-correlation-id'] as string) ?? uuidv4(),

        // ── Redação de campos sensíveis ────────────────────────────────────
        // Caminhos no objeto de log onde dados sensíveis podem aparecer.
        // Pino substitui o valor por "[Redacted]" antes de serializar.
        redact: {
          paths: [
            'req.headers.authorization',      // Bearer token
            'req.headers.cookie',             // Cookies de sessão
            'req.body.password',              // Senha em texto plano
            'req.body.passwordConfirmation',
            'req.body.currentPassword',
            'req.body.newPassword',
            'req.body.token',                 // Tokens de reset
            'req.body.refreshToken',
            'req.body.cpf',                   // PII — LGPD
            'req.body.cnpj',
            'req.body.birthDate',
            'req.body.phone',
            'req.body.creditCard',
            'req.body.cardNumber',
            'req.body.cvv',
            'res.headers["set-cookie"]',
          ],
          censor: '[Redacted]',
        },

        // ── Serializers: controla o que entra no log por request ──────────
        serializers: {
          req(req) {
            return {
              id:     req.id,           // correlationId
              method: req.method,
              url:    req.url,
              // ❌ Não logar: req.body completo — pode conter PII
              // ❌ Não logar: req.headers completos — contêm tokens
              userAgent: req.headers['user-agent'],
            };
          },
          res(res) {
            return {
              statusCode: res.statusCode,
              // ❌ Não logar: res.headers — podem conter cookies/tokens
            };
          },
        },

        // ── Customização de campos padrão ─────────────────────────────────
        // Adiciona campos extras a TODOS os logs da requisição
        customProps: (req) => ({
          correlationId: req.id,
          // userId virá do token JWT — adicionado no AuthGuard via logger.assign()
        }),

        // ── Formato do timestamp ──────────────────────────────────────────
        timestamp: () => `,"time":"${new Date().toISOString()}"`,

        // ── Modo pretty: apenas desenvolvimento ───────────────────────────
        transport: env.LOG_PRETTY
          ? { target: 'pino-pretty', options: { colorize: true, singleLine: false } }
          : undefined,

        // ── Desabilitar log automático de health checks ───────────────────
        autoLogging: {
          ignore: (req) => ['/health', '/metrics', '/favicon.ico'].includes(req.url ?? ''),
        },
      },
    }),

    // ... demais módulos
  ],
})
export class AppModule {}
```

---

## 5. Uso nos Services — `PinoLogger`

O `nestjs-pino` expõe dois loggers injetáveis:

- **`Logger`** (do `@nestjs/common`) — API NestJS padrão, contexto automático
- **`PinoLogger`** (do `nestjs-pino`) — API Pino nativa, `assign()` para adicionar campos

**Use `PinoLogger` quando precisar do `assign()`** (e.g., injetar `userId` após autenticação). Para uso padrão, `Logger` é suficiente.

```typescript
// src/modules/users/use-cases/create-user/create-user.use-case.ts
import { Injectable } from '@nestjs/common';
import { PinoLogger, InjectPinoLogger } from 'nestjs-pino';
import { CreateUserDto } from '../../dtos/create-user.dto';

@Injectable()
export class CreateUserUseCase {
  constructor(
    @InjectPinoLogger(CreateUserUseCase.name)
    private readonly logger: PinoLogger,
    // ... outras dependências
  ) {}

  async execute(dto: CreateUserDto): Promise<UserViewModel> {
    // ✅ Log de evento de negócio — ID do recurso, não dados pessoais
    this.logger.info({ userId: dto.id }, 'Iniciando criação de usuário');

    try {
      const user = await this.usersRepository.create(dto);

      // ✅ Log de sucesso com ID gerado
      this.logger.info({ userId: user.id }, 'Usuário criado com sucesso');

      return UserViewModel.toHttp(user);
    } catch (error) {
      // ✅ Log de erro com contexto de operação
      this.logger.error({ err: error, email: '[Redacted]' }, 'Falha ao criar usuário');
      throw error;
    }
  }
}
```

```typescript
// ── Logger simples (sem assign) ────────────────────────────────────────────
import { Logger } from '@nestjs/common';

@Injectable()
export class SomeService {
  private readonly logger = new Logger(SomeService.name);

  doSomething(): void {
    this.logger.log('Operação executada');          // info
    this.logger.warn('Aviso sobre comportamento');  // warn
    this.logger.error('Erro inesperado', stack);    // error
    this.logger.debug('Detalhe de debug');          // debug
    this.logger.verbose('Trace detalhado');         // trace
  }
}
```

---

## 6. Injetando `userId` no Contexto do Log

Após o `JwtAuthGuard` autenticar o usuário, injete o `userId` no contexto Pino para que apareça em **todos** os logs subsequentes da requisição — sem passar manualmente.

```typescript
// src/modules/auth/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { PinoLogger, InjectPinoLogger } from 'nestjs-pino';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(
    @InjectPinoLogger(JwtAuthGuard.name)
    private readonly logger: PinoLogger,
  ) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const result = super.canActivate(context);
    return result;
  }

  handleRequest<TUser>(err: unknown, user: TUser): TUser {
    if (err || !user) throw err;

    // ✅ Injeta userId no contexto do log desta requisição
    // A partir daqui, TODO log desta request terá { userId: "..." }
    this.logger.assign({ userId: (user as { id: string }).id });

    return user;
  }
}
```

---

## 7. Configuração do `main.ts`

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { Logger } from 'nestjs-pino';
import { AppModule } from './app.module';
import { env } from '@config/env.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    // Desativa o logger padrão do NestJS — Pino assume o controle
    bufferLogs: true,
  });

  // Usa o PinoLogger como logger do NestJS (bootstrap logs incluídos)
  app.useLogger(app.get(Logger));

  // ... configurações globais (pipes, interceptors, filters)

  await app.listen(env.PORT);
}
bootstrap();
```

---

## 8. Log Levels por Ambiente

| Ambiente | `LOG_LEVEL` | O que é logado |
|---|---|---|
| `development` | `debug` | Tudo: queries, fluxos internos, detalhes |
| `staging` | `info` | Eventos de negócio, erros, avisos |
| `production` | `info` (padrão) ou `warn` | Eventos críticos para o negócio |
| Investigação de bug | `debug` (temporário) | Aumentar pontualmente via env var |

**Regra:** O nível padrão de produção é `info`. Nunca suba `debug` ou `trace` em produção de forma permanente — o volume pode ser proibitivo e aumenta o risco de vazar dados sensíveis.

### Hierarquia de Níveis

```
fatal > error > warn > info > debug > trace
```

| Nível | Quando usar |
|---|---|
| `fatal` | Falha irrecuperável — app será encerrada |
| `error` | Exceção não esperada, operação falhou completamente |
| `warn` | Situação anormal mas recuperável (retry, degradação) |
| `info` | Eventos de negócio significativos (usuário criado, pagamento aprovado) |
| `debug` | Fluxo de execução, detalhes de implementação — apenas development |
| `trace` | Dados detalhados de entrada/saída — apenas investigação pontual |

---

## 9. O que Logar e o que NUNCA Logar

### ✅ O que DEVE ser logado

```typescript
// ✅ IDs de recursos — rastreáveis sem expor dados pessoais
this.logger.info({ orderId, userId }, 'Pedido criado');

// ✅ Eventos de negócio com estado
this.logger.info({ paymentId, status: 'approved' }, 'Pagamento aprovado');

// ✅ Erros com contexto de operação (sem dados do usuário)
this.logger.error({ err: error, operation: 'send-email', attempt: 3 }, 'Falha no envio');

// ✅ Métricas de performance relevantes
this.logger.info({ duration: elapsed, endpoint: '/checkout' }, 'Checkout concluído');

// ✅ Eventos de segurança (auditoria) — com dados mínimos
this.logger.warn({ userId, ip: maskedIp }, 'Tentativa de login falhou');

// ✅ Slow queries (cf. query-optimization.md) — sem valores dos parâmetros
this.logger.warn({ duration: 1200, query: 'SELECT users WHERE ...' }, 'Slow query detectada');
```

### ❌ O que NUNCA deve ser logado

```typescript
// ❌ Senhas — mesmo em hash
this.logger.debug({ passwordHash: user.passwordHash }, 'Usuário autenticado'); // ERRADO

// ❌ Tokens JWT / refresh tokens
this.logger.info({ token: jwtToken }, 'Token gerado'); // ERRADO

// ❌ Dados de cartão de crédito (PCI DSS)
this.logger.info({ cardNumber: '4111...' }, 'Pagamento processado'); // ERRADO

// ❌ CPF, RG, dados de saúde (LGPD/GDPR)
this.logger.info({ cpf: user.cpf }, 'Usuário encontrado'); // ERRADO

// ❌ Credenciais de terceiros (API keys, webhooks secrets)
this.logger.debug({ apiKey: externalKey }, 'Chamando API externa'); // ERRADO

// ❌ Dados bancários (IBAN, conta corrente)
this.logger.info({ bankAccount: payment.iban }, 'Transferência iniciada'); // ERRADO

// ❌ Corpo completo de requisições sem sanitização
this.logger.debug({ body: req.body }, 'Request recebida'); // ERRADO — pode ter PII

// ❌ Stack traces em produção no nível info/debug (nível error/fatal only)
this.logger.info({ stack: error.stack }, 'Algo deu errado'); // ERRADO — use error()
```

### Lista de campos NUNCA permitidos em logs (adicionar ao `redact`)

```typescript
// Campos que devem ser incluídos no array redact do pinoHttp
const SENSITIVE_FIELDS = [
  // Autenticação
  '*.password', '*.passwordHash', '*.token', '*.accessToken',
  '*.refreshToken', '*.apiKey', '*.secret', '*.privateKey',

  // PII — LGPD
  '*.cpf', '*.cnpj', '*.rg', '*.birthDate', '*.dataNascimento',
  '*.phone', '*.telefone', '*.celular',
  '*.address', '*.endereco', '*.cep',
  '*.motherName', '*.nomeMae',

  // Financeiro — PCI DSS
  '*.cardNumber', '*.numeroCartao', '*.cvv', '*.cvc',
  '*.bankAccount', '*.contaBancaria', '*.iban',

  // Saúde — HIPAA / LGPD sensível
  '*.healthData', '*.dadosSaude', '*.prescription',

  // Headers HTTP
  'req.headers.authorization',
  'req.headers.cookie',
  'res.headers["set-cookie"]',
];
```

---

## 10. Formato JSON de Log em Produção

Todo log produzido segue esta estrutura JSON (exemplo real):

```json
{
  "level": "info",
  "time": "2025-03-15T14:32:10.123Z",
  "pid": 1234,
  "hostname": "app-prod-1",
  "req": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "method": "POST",
    "url": "/api/v1/orders",
    "userAgent": "Mozilla/5.0..."
  },
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "usr_abc123",
  "orderId": "ord_xyz789",
  "msg": "Pedido criado com sucesso"
}
```

Campos padrão presentes em **todos** os logs:
- `level` — nível de severidade
- `time` — ISO 8601 timestamp
- `correlationId` — ID único da request (rastreabilidade)
- `userId` — injetado pelo `JwtAuthGuard` após autenticação
- `msg` — mensagem descritiva do evento

---

## 11. Integração com Datadog

### 11.1 Variável de ambiente

```bash
# .env — produção
DD_API_KEY=your-api-key
DD_SERVICE=my-nestjs-api
DD_ENV=production
DD_VERSION=1.0.0
```

### 11.2 Via Datadog Agent (recomendado)

O Datadog Agent coleta os logs JSON do `stdout` do container automaticamente. Nenhuma configuração extra no código NestJS é necessária além de logar em JSON para `stdout`.

```yaml
# docker-compose.yml ou Kubernetes DaemonSet
datadog-agent:
  image: gcr.io/datadoghq/agent:latest
  environment:
    - DD_API_KEY=${DD_API_KEY}
    - DD_LOGS_ENABLED=true
    - DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true
```

### 11.3 Mapeamento de campos para Datadog

O Datadog espera campos específicos. Configure o pipeline de log no painel do Datadog ou use o `pino-datadog-transport`:

```typescript
// Mapeamento esperado pelo Datadog
// Pino → Datadog
// "msg"   → "message"
// "level" → "status" (trace=7, debug=20, info=30, warn=40, error=50, fatal=60)
// "time"  → "@timestamp"

// Para remapear automaticamente, configure o pinoHttp com formatters:
formatters: {
  level: (label) => ({ level: label }),          // mantém o label textual
  log: (obj) => ({ ...obj, service: env.APP_NAME }),
},
messageKey: 'message',   // Datadog usa "message" como campo padrão
```

### 11.4 Correlation ID com Datadog APM (opcional)

Se você usa o `dd-trace` para APM, injete o trace ID automaticamente nos logs:

```typescript
// src/config/datadog.config.ts — inicializar ANTES de qualquer import do NestJS
import tracer from 'dd-trace';

if (env.NODE_ENV === 'production' || env.NODE_ENV === 'staging') {
  tracer.init({
    service: env.APP_NAME,
    env:     env.NODE_ENV,
    version: env.APP_VERSION,
    logInjection: true, // ← injeta dd.trace_id e dd.span_id em todo log Pino
  });
}
```

```typescript
// src/main.ts — importar ANTES de qualquer outro import
import './config/datadog.config'; // ← deve ser a primeira linha
import { NestFactory } from '@nestjs/core';
// ...
```

---

## 12. Integração com AWS CloudWatch

### 12.1 Via CloudWatch Agent ou Fluent Bit (recomendado)

Assim como no Datadog, o padrão recomendado é logar em JSON para `stdout` e deixar o agente externo coletar.

```yaml
# task-definition.json (ECS) — configuração do log driver
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-group":  "/ecs/my-nestjs-api",
    "awslogs-region": "us-east-1",
    "awslogs-stream-prefix": "ecs"
  }
}
```

### 12.2 Filtros CloudWatch (Log Insights)

Com logs em JSON, você pode usar consultas poderosas no CloudWatch Insights:

```sql
-- Buscar todos os logs de uma requisição específica
fields @timestamp, msg, userId, level
| filter correlationId = "550e8400-e29b-41d4-a716-446655440000"
| sort @timestamp asc

-- Erros nas últimas 24h
fields @timestamp, msg, err.message
| filter level = "error"
| sort @timestamp desc
| limit 100

-- Latência média por endpoint
fields req.url, responseTime
| stats avg(responseTime) as avgLatency by req.url
| sort avgLatency desc
```

---

## 13. Estrutura de Arquivos

```text
src/
├── config/
│   ├── env.config.ts          ← LOG_LEVEL e LOG_PRETTY adicionados aqui
│   └── datadog.config.ts      ← inicialização do dd-trace (opcional)
├── app.module.ts              ← LoggerModule.forRoot() configurado aqui
├── main.ts                    ← app.useLogger(app.get(Logger))
└── common/
    └── interceptors/
        └── logging.interceptor.ts   ← (opcional) log de req/res detalhado
```

---

## 14. Anti-Padrões — O que Nunca Fazer

```typescript
// ❌ console.log — bloqueante, não estruturado, sem correlationId
console.log('Usuário criado:', user);

// ❌ Instanciar Logger fora do ciclo de DI
const logger = new Logger(); // Perde o contexto do módulo

// ❌ Logar objetos completos de domínio — podem conter PII
this.logger.info({ user }, 'Usuário encontrado'); // user pode ter CPF, senha, etc.

// ❌ Usar o mesmo Logger em múltiplos módulos com contexto genérico
private readonly logger = new Logger('App'); // contexto inútil para debugging

// ❌ Log em loop de alta frequência no nível info/debug em produção
for (const item of largeArray) {
  this.logger.debug({ item }, 'Processando item'); // volume gigante
}

// ❌ Logar tokens de terceiros ou variáveis de ambiente
this.logger.debug({ config: process.env }, 'Configuração carregada'); // vaza secrets

// ❌ Usar pino-pretty em produção
transport: env.NODE_ENV === 'production'
  ? { target: 'pino-pretty' } // ERRADO — desabilita JSON, impacta performance
  : undefined,
```

---

## 15. Checklist de Revisão

### Configuração
- [ ] `nestjs-pino` configurado via `LoggerModule.forRoot()` no `AppModule`?
- [ ] `bufferLogs: true` e `app.useLogger(app.get(Logger))` no `main.ts`?
- [ ] `LOG_LEVEL` e `LOG_PRETTY` em `env.config.ts` com validação Zod?
- [ ] `LOG_PRETTY=true` apenas em `.env.development` (nunca produção)?
- [ ] `pino-pretty` no `devDependencies` (nunca em `dependencies`)?

### Correlation ID
- [ ] `genReqId` configurado para ler `x-correlation-id` do header ou gerar UUID v4?
- [ ] `customProps` propagando `correlationId` para todos os logs?
- [ ] `JwtAuthGuard` chamando `logger.assign({ userId })` após autenticação?

### Segurança e LGPD
- [ ] Array `redact` inclui todos os campos sensíveis (password, token, cpf, etc.)?
- [ ] Serializers de `req` e `res` não expõem headers completos nem body completo?
- [ ] Nenhum log usa `console.log` diretamente (regra ESLint `no-console`)?
- [ ] Objetos de domínio completos nunca são passados diretamente ao logger?
- [ ] Stack traces logados apenas no nível `error` ou `fatal`?

### Integração
- [ ] Datadog Agent ou CloudWatch Agent configurado para coletar `stdout`?
- [ ] Campos `correlationId` e `userId` aparecendo nos logs da plataforma?
- [ ] Health check (`/health`) excluído do auto-logging?

---

## Referências

- [config-management.md](.agent/skills/backend/configuration-and-environment/config-management.md) — `env.config.ts`, variáveis de ambiente, `@config/env.config`
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — `src/common/interceptors/`, estrutura de módulos
- [rest-api-patterns.md](.agent/skills/backend/api-and-contracts/rest-api-patterns.md) — `HttpExceptionFilter` global (ponto centralizado de log de erros HTTP)
- [query-optimization.md](.agent/skills/backend/database/query-optimization.md) — Log de slow queries com `logger.warn`
- [queue-workers.md](.agent/skills/backend/performance-and-infrastructure/queue-workers.md) — Padrão de log em workers BullMQ
- [nestjs-pino — GitHub](https://github.com/iamolegga/nestjs-pino)
- [Pino — Redaction](https://getpino.io/#/docs/redaction)
- [NestJS Docs — Logger](https://docs.nestjs.com/techniques/logger)
- [Datadog — Node.js Log Injection](https://docs.datadoghq.com/tracing/other_telemetry/connect_logs_and_traces/nodejs/)
- [AWS CloudWatch — Log Insights Query Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [LGPD — Lei Geral de Proteção de Dados (Lei 13.709/2018)](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm)