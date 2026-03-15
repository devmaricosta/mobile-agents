---
name: events-pattern
description: >
  Comunicação interna entre módulos com @nestjs/event-emitter: quando usar eventos vs chamada
  direta entre services, como evitar acoplamento entre domínios, e a diferença entre eventos de
  domínio (in-process, síncronos) e eventos de integração (cross-service, assíncronos via fila).
  Use esta skill sempre que um UseCase precisar notificar outro domínio sobre algo que aconteceu,
  ou quando identificar que dois módulos estão importando um ao outro criando acoplamento bidirecional.
---

# Padrão de Eventos — @nestjs/event-emitter

Você é um Arquiteto de Software Senior. Esta skill define **como e quando usar eventos internos** no monolito modular NestJS para comunicação entre domínios, de forma consistente com a arquitetura definida em `nest-project-structure.md`, `nest-modules-pattern.md` e `clean-architecture.md`.

> **Decisões de arquitetura (não altere sem revisão):**
> - Pacote: `@nestjs/event-emitter` (baseado em `EventEmitter2`) — **não** use `EventEmitter` nativo do Node.js diretamente.
> - Eventos de domínio: in-process, síncronos por padrão — o emissor aguarda todos os listeners antes de continuar.
> - Eventos assíncronos (`async: true`): use com cuidado — erros nos listeners **não** propagam para o emissor.
> - Eventos de integração (cross-service/cross-process): usam BullMQ — ver `queue-workers.md`.
> - Nomes de evento: `dominio.verbo-no-passado` em kebab-case — ex: `user.registered`, `order.placed`.
> - Classes de evento: imutáveis (`readonly`), sem lógica, em `events/` dentro do módulo emissor.
> - Listeners: `@Injectable()` em `listeners/` dentro do módulo **receptor**, nunca no emissor.

---

## Quando usar esta skill

- **Acoplamento bidirecional:** dois módulos importam um ao outro — eventos quebram o ciclo.
- **Side effects após uma operação:** enviar e-mail, atualizar cache, gerar auditoria após salvar entidade.
- **Notificar múltiplos domínios:** uma ação em `orders` deve notificar `inventory`, `notifications` e `analytics` simultaneamente.
- **Desacoplar regras de negócio de efeitos colaterais:** manter o UseCase limpo e focado em sua responsabilidade principal.
- **Dúvida entre chamada direta vs evento:** use o fluxograma da seção 2.

---

## 1. Eventos de Domínio vs Eventos de Integração

Esta é a distinção mais importante. Confundir os dois causa bugs difíceis de rastrear.

| Característica | Evento de Domínio | Evento de Integração |
|---|---|---|
| **Escopo** | Intra-processo (mesmo monolito) | Inter-processo (serviços externos) |
| **Transporte** | In-memory (`EventEmitter2`) | Message broker (BullMQ + Redis) |
| **Execução** | Síncrona (padrão) | Assíncrona (sempre) |
| **Garantia** | Nenhuma — se o processo cai, o evento se perde | Pelo menos uma vez (retry automático) |
| **Caso de uso** | Comunicar domínios dentro do mesmo monolito | Integrar com serviços externos, webhooks, filas externas |
| **Skill de referência** | Esta skill | `queue-workers.md` |

> **Regra prática:** Se o consumidor do evento está no mesmo processo NestJS, é evento de domínio. Se está em outro serviço, microserviço, ou precisar de retry e persistência, é evento de integração (use BullMQ).

---

## 2. Evento vs Chamada Direta — Fluxograma de Decisão

```
O UseCase precisa acionar outra operação após concluir a sua?
│
├── O receptor está no MESMO domínio (mesmo módulo)?
│   └── ✅ Chamada direta entre services ou use cases — sem evento.
│       Ex: CreateOrderUseCase → OrderRepository (mesmo domínio)
│
└── O receptor está em DOMÍNIO DIFERENTE?
    │
    ├── A operação principal DEPENDE do resultado do receptor?
    │   (ex: precisa do retorno para continuar, ou deve falhar junto)
    │   └── ✅ Chamada direta via import explícito do módulo receptor.
    │       Ex: CreateMealUseCase → UsersModule.GetUserUseCase (validação necessária)
    │
    └── A operação é um SIDE EFFECT independente?
        (log, e-mail, cache, auditoria, notificação push, etc.)
        │
        ├── Precisa de retry, persistência, ou é lenta (> 200ms)?
        │   └── ✅ Evento de integração → BullMQ (ver queue-workers.md)
        │
        └── In-process, rápida, sem necessidade de retry?
            └── ✅ Evento de domínio → @nestjs/event-emitter (esta skill)
```

### Exemplos práticos

| Situação | Abordagem correta |
|---|---|
| `CreateUserUseCase` salva usuário e quer enviar e-mail de boas-vindas | Evento de domínio `user.registered` → listener enfileira no BullMQ |
| `PlaceOrderUseCase` quer validar estoque antes de confirmar pedido | Chamada direta: importa `InventoryModule` e injeta `CheckStockUseCase` |
| `UpdateProfileUseCase` quer invalidar cache de perfil de outro módulo | Evento de domínio `user.profile-updated` → listener do CacheModule |
| `CheckoutUseCase` quer notificar sistema externo de pagamento | Evento de integração via BullMQ — não é in-process |
| `CreatePostUseCase` quer registrar em log de auditoria (mesmo processo) | Evento de domínio `post.created` → listener de auditoria |

---

## 2. Instalação e Configuração Global

```bash
npm install @nestjs/event-emitter
```

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot({
      wildcard: true,         // ← habilita padrões glob: 'user.*', 'order.**'
      delimiter: '.',         // ← separador de namespace (padrão)
      maxListeners: 20,       // ← aumentar se houver muitos domínios ouvindo
      verboseMemoryLeak: true, // ← avisa no console se passar de maxListeners
      ignoreErrors: false,    // ← NUNCA true em produção — erros devem ser visíveis
    }),
    // ... demais módulos
  ],
})
export class AppModule {}
```

> **Por que `wildcard: true`?** Permite que listeners usem `'user.*'` para capturar todos os eventos do domínio `user` de uma vez — útil para auditoria e observabilidade transversal. Mas atenção: use com disciplina para não criar listeners genéricos demais.

---

## 3. Estrutura de Pastas

Seguindo `nest-project-structure.md`, eventos e listeners ficam co-localizados no módulo correto:

```
src/
└── modules/
    ├── users/                           ← domínio EMISSOR
    │   ├── events/
    │   │   ├── user-registered.event.ts   ← classe do evento (imutável)
    │   │   ├── user-profile-updated.event.ts
    │   │   └── index.ts                   ← barrel de eventos públicos
    │   ├── use-cases/
    │   │   └── register/
    │   │       └── register.use-case.ts  ← emite o evento após salvar
    │   └── users.module.ts
    │
    └── notifications/                   ← domínio RECEPTOR
        ├── listeners/
        │   └── send-welcome-email.listener.ts  ← @OnEvent, @Injectable
        └── notifications.module.ts
```

> **Regra:** A classe do evento (`*.event.ts`) pertence ao módulo **emissor** — é o contrato público daquele domínio. O listener pertence ao módulo **receptor** — é a reação daquele domínio ao evento externo.

---

## 4. Classes de Evento

Classes de evento são DTOs simples e imutáveis. **Não contêm lógica.**

```typescript
// src/modules/users/events/user-registered.event.ts

/**
 * Evento emitido quando um novo usuário conclui o registro com sucesso.
 *
 * Contrato público do domínio Users — outros módulos podem ouvir este evento
 * sem importar nada de dentro de UsersModule.
 *
 * Convenção de nome: '<dominio>.<verbo-passado>'
 */
export class UserRegisteredEvent {
  static readonly EVENT_NAME = 'user.registered' as const;

  constructor(
    public readonly userId: string,
    public readonly email: string,
    public readonly name: string,
    public readonly registeredAt: Date,
  ) {}
}
```

```typescript
// src/modules/users/events/user-profile-updated.event.ts
export class UserProfileUpdatedEvent {
  static readonly EVENT_NAME = 'user.profile-updated' as const;

  constructor(
    public readonly userId: string,
    public readonly updatedFields: ReadonlyArray<string>, // quais campos mudaram
  ) {}
}
```

```typescript
// src/modules/users/events/index.ts
export { UserRegisteredEvent } from './user-registered.event';
export { UserProfileUpdatedEvent } from './user-profile-updated.event';
```

> **Por que `static readonly EVENT_NAME`?** Centraliza o nome do evento na classe — elimina strings mágicas espalhadas pelo código. Emissor e listener referenciam `UserRegisteredEvent.EVENT_NAME` em vez de `'user.registered'` literal.

---

## 5. Emissor — UseCase

O UseCase injeta `EventEmitter2` e emite o evento **após** confirmar a operação principal (persistência, regras de negócio). **Nunca emita antes de confirmar o sucesso.**

```typescript
// src/modules/users/use-cases/register/register.use-case.ts
import { Injectable } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { UsersRepository } from '@modules/users/repositories/users.repository';
import { RegisterDto } from '@modules/users/dtos/register.dto';
import { UserEntity } from '@modules/users/entities/user.entity';
import { UserRegisteredEvent } from '@modules/users/events/user-registered.event';

@Injectable()
export class RegisterUseCase {
  constructor(
    private readonly usersRepository: UsersRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async execute(dto: RegisterDto): Promise<UserEntity> {
    // 1. Regra de negócio — verifica duplicidade
    const existing = await this.usersRepository.findByEmail(dto.email);
    if (existing) {
      throw new ConflictException('E-mail já cadastrado');
    }

    // 2. Persistência — operação principal
    const user = await this.usersRepository.create({
      email: dto.email,
      name: dto.name,
      passwordHash: await hashPassword(dto.password),
    });

    // 3. Evento — SOMENTE após confirmação da persistência
    //    emitAsync retorna Promise<any[]> — awaitar garante que listeners
    //    síncronos e assíncronos terminem antes de retornar ao controller.
    await this.eventEmitter.emitAsync(
      UserRegisteredEvent.EVENT_NAME,
      new UserRegisteredEvent(user.id, user.email, user.name, user.createdAt),
    );

    return user;
  }
}
```

> **`emit` vs `emitAsync`:**
> - `emit(name, payload)` — síncrono, dispara todos os listeners em sequência. Listeners `async` **não** são aguardados — use apenas quando tiver certeza de que todos os listeners são síncronos.
> - `emitAsync(name, payload)` — retorna `Promise<any[]>`, aguarda listeners `async`. **Preferido na maioria dos casos** para evitar bugs sutis com listeners assíncronos.

---

## 6. Listener — Módulo Receptor

```typescript
// src/modules/notifications/listeners/send-welcome-email.listener.ts
import { Injectable, Logger } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { UserRegisteredEvent } from '@modules/users/events/user-registered.event';
import { NotificationsProducer } from '@modules/notifications/queues/notifications.producer';

@Injectable()
export class SendWelcomeEmailListener {
  private readonly logger = new Logger(SendWelcomeEmailListener.name);

  constructor(private readonly producer: NotificationsProducer) {}

  /**
   * Reage ao registro de um novo usuário enfileirando o e-mail de boas-vindas.
   *
   * Usa { async: true } porque enfileirar no BullMQ é uma operação I/O.
   * Erros aqui NÃO propagam para o RegisterUseCase — são capturados e logados.
   */
  @OnEvent(UserRegisteredEvent.EVENT_NAME, { async: true })
  async handleUserRegistered(event: UserRegisteredEvent): Promise<void> {
    this.logger.log(`Enfileirando e-mail de boas-vindas para userId=${event.userId}`);

    try {
      await this.producer.addSendEmail({
        userId: event.userId,
        to: event.email,
        subject: 'Bem-vindo!',
        template: 'welcome',
        context: { name: event.name },
      });
    } catch (error) {
      // Logar e não relançar — o registro já foi feito, o e-mail é side effect
      this.logger.error(
        `Falha ao enfileirar e-mail para userId=${event.userId}`,
        error,
      );
    }
  }
}
```

```typescript
// src/modules/notifications/notifications.module.ts
import { Module } from '@nestjs/common';
import { SendWelcomeEmailListener } from './listeners/send-welcome-email.listener';
import { NotificationsProducer } from './queues/notifications.producer';
// ... outros imports

@Module({
  providers: [
    SendWelcomeEmailListener, // ← registrar o listener como provider é OBRIGATÓRIO
    NotificationsProducer,
    // ...
  ],
})
export class NotificationsModule {}
```

> **Atenção crítica:** O listener **deve** estar nos `providers[]` do seu módulo para ser instanciado pelo DI do NestJS. Sem isso, o `@OnEvent` nunca é registrado e o evento é silenciosamente ignorado.

---

## 7. Listener com `{ async: true }` vs Síncrono

| Configuração | Comportamento | Quando usar |
|---|---|---|
| `@OnEvent('user.registered')` | Síncrono — o `emitAsync` aguarda, `emit` não aguarda | Listeners rápidos que não fazem I/O (ex: atualizar variável de estado in-memory) |
| `@OnEvent('user.registered', { async: true })` | Assíncrono — `emitAsync` aguarda a Promise resolver | Qualquer listener que faça I/O: banco, fila, chamada HTTP |
| `@OnEvent('user.registered', { async: true, promisify: true })` | Alias de `async: true` | Equivalente, prefira `async: true` por clareza |

> **Regra:** Se o listener faz qualquer operação I/O (banco, fila, HTTP, cache), sempre use `{ async: true }`. Sem isso, o `await emitAsync()` não aguardará a conclusão real do listener assíncrono.

---

## 8. Múltiplos Listeners para o Mesmo Evento

Um único evento pode ter múltiplos listeners em módulos diferentes — este é o principal benefício do padrão. Cada listener é independente e pertence ao seu próprio domínio.

```typescript
// notifications/listeners/send-welcome-email.listener.ts
@OnEvent(UserRegisteredEvent.EVENT_NAME, { async: true })
async handleUserRegistered(event: UserRegisteredEvent): Promise<void> { ... }

// audit/listeners/log-user-registration.listener.ts
@OnEvent(UserRegisteredEvent.EVENT_NAME, { async: true })
async handleUserRegistered(event: UserRegisteredEvent): Promise<void> { ... }

// analytics/listeners/track-new-signup.listener.ts
@OnEvent(UserRegisteredEvent.EVENT_NAME, { async: true })
async handleUserRegistered(event: UserRegisteredEvent): Promise<void> { ... }
```

> **Ordem de execução:** Com `emitAsync`, os listeners são executados em paralelo (Promise.all internamente). **Não assuma ordem**. Se a ordem for importante, use eventos encadeados ou chamadas diretas.

---

## 9. Wildcard e Prioridade

```typescript
// Listener que captura TODOS os eventos do domínio user (útil para auditoria)
@OnEvent('user.*', { async: true })
async handleAnyUserEvent(event: unknown): Promise<void> {
  this.logger.log('Evento de usuário recebido', event);
}

// Prioridade — listener com priority maior é executado primeiro
@OnEvent('order.placed', { async: true, priority: 10 })
async handleHighPriority(event: OrderPlacedEvent): Promise<void> {
  // Executado antes dos listeners com priority padrão (0)
}

// ATENÇÃO: wildcard requer { wildcard: true } no EventEmitterModule.forRoot()
```

---

## 10. Tratamento de Erros

Erros em listeners podem silenciosamente engolir falhas. Siga estas regras:

```typescript
// ❌ ERRADO — erro não tratado pode crashar o processo inteiro (ignoreErrors: false)
@OnEvent('user.registered', { async: true })
async handleUserRegistered(event: UserRegisteredEvent): Promise<void> {
  await this.someService.doSomething(event.userId); // lança exceção → crash
}

// ✅ CORRETO — side effects devem capturar e logar erros
@OnEvent('user.registered', { async: true })
async handleUserRegistered(event: UserRegisteredEvent): Promise<void> {
  try {
    await this.someService.doSomething(event.userId);
  } catch (error) {
    this.logger.error(`Falha no listener para userId=${event.userId}`, error);
    // NÃO relançar — o emissor não deve falhar por causa de um side effect
  }
}

// ✅ CORRETO — quando o listener deve propagar falha ao emissor (raro)
// Use emitAsync e não faça try/catch no listener.
// O RegisterUseCase receberá o erro e pode tratar como quiser.
@OnEvent('user.registered', { async: true })
async handleCriticalStep(event: UserRegisteredEvent): Promise<void> {
  await this.criticalService.mustSucceed(event.userId); // lança e o emissor recebe
}
```

> **Regra:** Side effects (e-mail, log, cache, auditoria) **não devem** propagar falhas ao emissor — capture e logue. Apenas listeners que representam parte obrigatória do fluxo de negócio devem relançar.

---

## 11. Testes

### 11.1 Testando o UseCase (mock do EventEmitter2)

```typescript
// src/modules/users/use-cases/register/__tests__/register.use-case.spec.ts
import { Test } from '@nestjs/testing';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { RegisterUseCase } from '../register.use-case';
import { UsersRepository } from '@modules/users/repositories/users.repository';
import { UserRegisteredEvent } from '@modules/users/events/user-registered.event';

describe('RegisterUseCase', () => {
  let useCase: RegisterUseCase;
  let eventEmitter: jest.Mocked<EventEmitter2>;
  let usersRepository: jest.Mocked<UsersRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        RegisterUseCase,
        {
          provide: EventEmitter2,
          useValue: { emitAsync: jest.fn().mockResolvedValue([]) },
        },
        {
          provide: UsersRepository,
          useValue: { findByEmail: jest.fn(), create: jest.fn() },
        },
      ],
    }).compile();

    useCase        = module.get(RegisterUseCase);
    eventEmitter   = module.get(EventEmitter2);
    usersRepository = module.get(UsersRepository);
  });

  it('deve emitir UserRegisteredEvent após criar usuário', async () => {
    const mockUser = { id: 'u1', email: 'a@b.com', name: 'Test', createdAt: new Date() };
    usersRepository.findByEmail.mockResolvedValue(null);
    usersRepository.create.mockResolvedValue(mockUser as any);

    await useCase.execute({ email: 'a@b.com', name: 'Test', password: '123456' });

    expect(eventEmitter.emitAsync).toHaveBeenCalledWith(
      UserRegisteredEvent.EVENT_NAME,
      expect.objectContaining({ userId: 'u1', email: 'a@b.com' }),
    );
  });

  it('NÃO deve emitir evento se o usuário já existe', async () => {
    usersRepository.findByEmail.mockResolvedValue({ id: 'existing' } as any);

    await expect(
      useCase.execute({ email: 'a@b.com', name: 'Test', password: '123456' }),
    ).rejects.toThrow(ConflictException);

    expect(eventEmitter.emitAsync).not.toHaveBeenCalled();
  });
});
```

### 11.2 Testando o Listener isoladamente

```typescript
// src/modules/notifications/listeners/__tests__/send-welcome-email.listener.spec.ts
import { Test } from '@nestjs/testing';
import { SendWelcomeEmailListener } from '../send-welcome-email.listener';
import { NotificationsProducer } from '@modules/notifications/queues/notifications.producer';
import { UserRegisteredEvent } from '@modules/users/events/user-registered.event';

describe('SendWelcomeEmailListener', () => {
  let listener: SendWelcomeEmailListener;
  let producer: jest.Mocked<NotificationsProducer>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        SendWelcomeEmailListener,
        {
          provide: NotificationsProducer,
          useValue: { addSendEmail: jest.fn().mockResolvedValue(undefined) },
        },
      ],
    }).compile();

    listener = module.get(SendWelcomeEmailListener);
    producer = module.get(NotificationsProducer);
  });

  it('deve enfileirar e-mail de boas-vindas ao receber UserRegisteredEvent', async () => {
    const event = new UserRegisteredEvent('u1', 'user@test.com', 'User', new Date());

    await listener.handleUserRegistered(event);

    expect(producer.addSendEmail).toHaveBeenCalledWith(
      expect.objectContaining({ userId: 'u1', to: 'user@test.com', template: 'welcome' }),
    );
  });

  it('não deve propagar erro se o producer falhar', async () => {
    producer.addSendEmail.mockRejectedValue(new Error('Redis offline'));
    const event = new UserRegisteredEvent('u1', 'user@test.com', 'User', new Date());

    // Não deve rejeitar — o erro é capturado internamente
    await expect(listener.handleUserRegistered(event)).resolves.not.toThrow();
  });
});
```

---

## 12. Anti-Padrões — O que NUNCA Fazer

```typescript
// ❌ ERRADO — emitir antes de confirmar persistência (evento fantasma)
async execute(dto: RegisterDto): Promise<UserEntity> {
  this.eventEmitter.emit('user.registered', new UserRegisteredEvent(...));
  const user = await this.usersRepository.create(dto); // e se falhar aqui?
  return user;
}

// ❌ ERRADO — string mágica espalhada (sincronismo frágil com o nome)
this.eventEmitter.emit('user.registered', payload);
@OnEvent('user.registerd') // typo silencioso — nunca dispara
async handle(event) {}

// ✅ CORRETO — sempre usar a constante da classe de evento
this.eventEmitter.emitAsync(UserRegisteredEvent.EVENT_NAME, payload);
@OnEvent(UserRegisteredEvent.EVENT_NAME)
async handle(event: UserRegisteredEvent) {}

// ❌ ERRADO — listener no mesmo módulo do emissor (viola separação de responsabilidades)
// users/users.service.ts
@OnEvent('order.placed') // Users não deve reagir a eventos de Orders diretamente aqui
async updateUserStats() {}

// ✅ CORRETO — listener fica no módulo que possui a responsabilidade
// users/listeners/update-user-stats-on-order.listener.ts (dentro de UsersModule)
@OnEvent(OrderPlacedEvent.EVENT_NAME, { async: true })
async handleOrderPlaced(event: OrderPlacedEvent): Promise<void> {}

// ❌ ERRADO — usar emit() com listeners async (não aguarda a Promise)
this.eventEmitter.emit('user.registered', payload); // listeners async não são esperados!

// ✅ CORRETO — sempre emitAsync quando há listeners async
await this.eventEmitter.emitAsync('user.registered', payload);
```

---

## 13. Checklist de Implementação

### Configuração
- [ ] `EventEmitterModule.forRoot({ wildcard: true, ignoreErrors: false })` no `AppModule`?
- [ ] `ignoreErrors: false` — **nunca** alterar para `true` em produção?

### Classe de Evento
- [ ] Arquivo em `src/modules/<domínio>/events/<nome>.event.ts`?
- [ ] Classe em PascalCase com sufixo `Event` (`UserRegisteredEvent`)?
- [ ] `static readonly EVENT_NAME` com valor `'dominio.verbo-passado'`?
- [ ] Todos os campos são `readonly` (imutabilidade)?
- [ ] Sem lógica de negócio na classe de evento?
- [ ] Barrel `events/index.ts` exportando todos os eventos públicos do domínio?

### Emissor (UseCase)
- [ ] Injeta `EventEmitter2` via construtor?
- [ ] Usa `emitAsync` (não `emit`) quando há listeners async?
- [ ] Emite **após** confirmar a persistência e regras de negócio?
- [ ] Usa `ClasseDoEvento.EVENT_NAME` (sem string literal)?

### Listener
- [ ] Arquivo em `src/modules/<domínio-receptor>/listeners/<nome>.listener.ts`?
- [ ] Classe `@Injectable()` registrada nos `providers[]` do módulo?
- [ ] `@OnEvent(ClasseEvento.EVENT_NAME, { async: true })` para operações I/O?
- [ ] Try/catch presente se o listener é um side effect (não deve propagar falha)?
- [ ] Logger configurado para registrar erros capturados?

### Testes
- [ ] UseCase testado com `EventEmitter2` mockado (`emitAsync: jest.fn()`)?
- [ ] Teste verifica que evento é emitido com payload correto após persistência?
- [ ] Teste verifica que evento NÃO é emitido quando há falha na operação principal?
- [ ] Listener testado de forma isolada, sem dependência do EventEmitter?
- [ ] Listener testado para o cenário de falha (não propaga erro)?

---

## Referências

- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — Estrutura de pastas `src/modules/<domínio>/events/` e `listeners/`
- [nest-modules-pattern.md](.agent/skills/backend/foundation-and-architecture/nest-modules-pattern.md) — `providers[]` obrigatório para listeners, exports e fronteiras de módulo
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — UseCase como emissor de eventos, listeners como camada de aplicação do receptor
- [queue-workers.md](.agent/skills/backend/performance-and-infrastructure/queue-workers.md) — Eventos de integração, BullMQ para side effects que precisam de retry
- [testing-strategy-nest.md](.agent/skills/backend/quality-and-testing/testing-strategy-nest.md) — Padrão de mock com `Test.createTestingModule`
- [NestJS Docs — Events](https://docs.nestjs.com/techniques/events)
- [EventEmitter2 — GitHub](https://github.com/EventEmitter2/EventEmitter2)