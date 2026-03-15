---
name: websockets-pattern
description: WebSockets com NestJS Gateways — autenticação JWT no handshake via Socket.IO middleware (não WsGuard no handleConnection), rooms e namespaces, RedisIoAdapter com @socket.io/redis-adapter para escala horizontal, e tabela de decisão WebSockets vs SSE (Server-Sent Events). Complementa auth-jwt-pattern.md (JwtService e UserPayload), caching-strategy.md (REDIS_URL e AppCacheModule), events-pattern.md (EventEmitter2 para notificação interna), e clean-architecture.md (Gateway na camada de Presentation).
---

# WebSockets com NestJS Gateways

Você é um Engenheiro de Software Senior especialista em sistemas de tempo real com NestJS e Socket.IO. Esta skill define **como implementar WebSockets corretamente** no projeto NestJS, cobrindo autenticação, rooms, namespaces e escala horizontal com Redis.

> **Decisões de arquitetura (não altere sem revisão):**
> - Autenticação via **Socket.IO middleware no `afterInit`** — não no `handleConnection` nem no `WsGuard` de `@SubscribeMessage`. O motivo: `handleConnection` não pode interromper o handshake TCP de forma limpa; o middleware rejeita a conexão antes de estabelecê-la.
> - **`@socket.io/redis-adapter`** para escala horizontal — não `socket.io-redis` (descontinuado) nem `@nestjs/redis` diretamente.
> - Redis compartilha a mesma `REDIS_URL` já definida em `caching-strategy.md` — dois clientes dedicados (pub/sub) criados separados do `AppCacheModule`.
> - **Namespaces por domínio** — `/notifications`, `/chat`, `/presence` — nunca tudo no namespace raiz `/`.

---

## 1. Quando Usar WebSockets vs SSE (Server-Sent Events)

> **Regra de ouro:** Se o cliente também precisa enviar dados em tempo real → WebSocket. Se apenas o servidor envia → SSE.

| Critério | WebSocket | SSE |
|---|---|---|
| **Direção** | Bidirecional (full-duplex) | Unidirecional (server → client) |
| **Protocolo** | WS/WSS (upgrade HTTP) | HTTP/HTTPS puro |
| **Reconnect automático** | ❌ Deve ser implementado | ✅ Nativo no `EventSource` |
| **HTTP/2 multiplexing** | ❌ Uma conexão TCP por socket | ✅ Streams múltiplos por conexão |
| **Proxies/Firewalls** | ⚠️ Alguns bloqueiam `Upgrade` | ✅ Passa como HTTP normal |
| **CDN/Load Balancer** | ⚠️ Configuração especial | ✅ Transparente |
| **Overhead por frame** | Maior (mascaramento XOR) | Menor (texto puro) |
| **Estado no servidor** | Stateful (conexão persistente) | Stateless por request |
| **NestJS support** | `@WebSocketGateway` | `@Sse()` no Controller |

### Use WebSocket quando:
- Chat, mensagens em tempo real, colaboração (múltiplos usuários editando)
- Jogos multiplayer ou presença de usuários
- O cliente precisa enviar eventos frequentes (> 1 por segundo) ao servidor
- Você precisa de latência mínima bidirecional (< 10ms)

### Use SSE quando:
- Notificações push (servidor → cliente apenas)
- Dashboards com atualizações periódicas (preços, métricas, logs)
- Streaming de respostas de IA (LLM token-by-token)
- Rastreamento de pedidos, status de builds/jobs
- Você quer simplicidade e não precisa de client→server em tempo real

```typescript
// SSE no NestJS — simples, sem Gateway, sem Redis adapter
// src/modules/notifications/controllers/notifications.controller.ts
import { Controller, Sse, UseGuards, Req } from '@nestjs/common';
import { Observable, fromEvent, map } from 'rxjs';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { JwtAuthGuard } from '@common/guards/jwt-auth.guard';
import { Request } from 'express';

@Controller('v1/notifications')
export class NotificationsController {
  constructor(private readonly eventEmitter: EventEmitter2) {}

  // SSE: apenas GET, sem corpo, Content-Type: text/event-stream
  @UseGuards(JwtAuthGuard)  // ← guard HTTP normal funciona aqui
  @Sse('stream')
  stream(@Req() req: Request): Observable<MessageEvent> {
    const userId = req.user.id;
    return fromEvent(this.eventEmitter, `notification.${userId}`).pipe(
      map((data) => ({ data } as MessageEvent)),
    );
  }
}
```

> **Nota SSE + Redis:** Para escalar SSE horizontalmente, use o `EventEmitter2` combinado com Redis Pub/Sub (ver `events-pattern.md` e `caching-strategy.md`). Cada instância subscreve o canal Redis e re-emite via `EventEmitter2` — os clientes SSE conectados naquela instância recebem a mensagem.

---

## 2. Instalação

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
npm install @socket.io/redis-adapter  # adapter Redis para escala horizontal
# redis e ioredis já instalados via caching-strategy.md
```

---

## 3. Variáveis de Ambiente

Complementa `config-management.md` — as variáveis abaixo **já existem** em `env.config.ts` via `caching-strategy.md`:

```typescript
// src/config/env.config.ts — já existente, nenhuma adição necessária
// REDIS_URL: z.string().url().optional()  ← já definida em caching-strategy.md
```

Adicione apenas a configuração de CORS para WebSocket se necessário:

```typescript
// src/config/env.config.ts — adicione ao schema existente se precisar controlar CORS:
const envSchema = z.object({
  // ... variáveis existentes
  WS_CORS_ORIGIN: z.string().default('*'),  // ex: 'https://app.exemplo.com.br'
});
```

---

## 4. Estrutura de Pastas

Seguindo `nest-project-structure.md`, cada domínio com Gateway fica autocontido:

```
src/
├── infra/
│   └── websocket/
│       └── redis-io.adapter.ts       ← RedisIoAdapter (adaptador global)
│
└── modules/
    ├── notifications/
    │   ├── gateways/
    │   │   └── notifications.gateway.ts    ← @WebSocketGateway({ namespace: '/notifications' })
    │   ├── use-cases/
    │   │   └── send-notification/
    │   │       └── send-notification.use-case.ts
    │   └── notifications.module.ts
    │
    └── chat/
        ├── gateways/
        │   └── chat.gateway.ts             ← @WebSocketGateway({ namespace: '/chat' })
        ├── use-cases/
        │   └── send-message/
        │       └── send-message.use-case.ts
        └── chat.module.ts
```

> **Regra:** A pasta `gateways/` fica dentro do módulo de domínio, assim como `controllers/`. O Gateway é a camada de Presentation para WebSockets — mesma posição na Clean Architecture que o Controller é para HTTP.

---

## 5. RedisIoAdapter — Escala Horizontal

O `RedisIoAdapter` é o adaptador global que faz o Socket.IO usar Redis Pub/Sub para sincronizar mensagens entre instâncias. Sem ele, clientes conectados em instâncias diferentes não recebem mensagens uns dos outros.

```typescript
// src/infra/websocket/redis-io.adapter.ts
import { IoAdapter } from '@nestjs/platform-socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import { INestApplicationContext, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { ServerOptions } from 'socket.io';

/**
 * Adaptador Redis para Socket.IO.
 * Permite que mensagens emitidas em uma instância sejam recebidas
 * por clientes conectados em outras instâncias (escala horizontal).
 *
 * IMPORTANTE: Usa dois clientes Redis dedicados (pub/sub) — separados
 * do AppCacheModule para evitar conflitos com o cache-manager.
 */
export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;
  private readonly logger = new Logger(RedisIoAdapter.name);

  constructor(private readonly app: INestApplicationContext) {
    super(app);
  }

  async connectToRedis(): Promise<void> {
    const configService = this.app.get(ConfigService);
    const redisUrl = configService.get<string>('cache.redisUrl') ?? 'redis://localhost:6379';

    // Dois clientes dedicados: pub e sub
    // Nunca compartilhe estes clientes com o cache-manager (AppCacheModule)
    const pubClient = createClient({ url: redisUrl });
    const subClient = pubClient.duplicate();

    pubClient.on('error', (err) => this.logger.error('Redis pub client error', err));
    subClient.on('error', (err) => this.logger.error('Redis sub client error', err));

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
    this.logger.log('RedisIoAdapter conectado ao Redis');
  }

  createIOServer(port: number, options?: ServerOptions) {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

### Registro no `main.ts`

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { RedisIoAdapter } from '@infra/websocket/redis-io.adapter';
import { ConfigService } from '@nestjs/config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // ── WebSocket: Redis Adapter (deve vir antes do app.listen) ──
  const redisIoAdapter = new RedisIoAdapter(app);
  await redisIoAdapter.connectToRedis();
  app.useWebSocketAdapter(redisIoAdapter);

  // ... outros setups (ValidationPipe, Swagger, etc.)

  const configService = app.get(ConfigService);
  await app.listen(configService.get('PORT') ?? 3000);
}
bootstrap();
```

> **Em desenvolvimento local (sem Redis):** Se `REDIS_URL` não estiver definida, defina `connectToRedis()` para usar `redis://localhost:6379` ou torne o adapter condicional: se não houver `REDIS_URL`, use o `IoAdapter` padrão (in-memory, sem escala).

---

## 6. Autenticação no Gateway — Socket.IO Middleware

> **Padrão canônico:** Autentique no `afterInit` via `server.use()` (middleware Socket.IO), não no `handleConnection`. O middleware executa **antes** do handshake ser concluído — permite rejeitar conexões não autorizadas antes de alocarem recursos.

### 6.1 Tipo Estendido do Socket

```typescript
// src/common/types/authenticated-socket.type.ts
import { Socket } from 'socket.io';
import { UserPayload } from './user-payload.type';  // já definido em auth-jwt-pattern.md

/**
 * Socket com usuário autenticado já populado no handshake.
 * Disponível após o middleware de autenticação executar.
 */
export interface AuthenticatedSocket extends Socket {
  data: {
    user: UserPayload;  // { id, email, roles } — mesmo tipo do request.user HTTP
  };
}
```

### 6.2 Middleware de Autenticação WS

```typescript
// src/common/middlewares/ws-auth.middleware.ts
import { Injectable, Logger } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigType } from '@nestjs/config';
import { Inject } from '@nestjs/common';
import { Socket } from 'socket.io';
import jwtConfig from '@config/jwt.config';
import type { UserPayload } from '@common/types/user-payload.type';

export type SocketMiddleware = (
  socket: Socket,
  next: (err?: Error) => void,
) => void;

/**
 * Middleware Socket.IO para autenticação JWT.
 * Extrai o token de socket.handshake.auth.token (padrão Socket.IO v4)
 * ou do header Authorization como fallback.
 *
 * Em caso de sucesso, popula socket.data.user com o UserPayload decodificado.
 * Em caso de falha, chama next(new Error('Unauthorized')) — Socket.IO recusa a conexão.
 */
@Injectable()
export class WsAuthMiddleware {
  private readonly logger = new Logger(WsAuthMiddleware.name);

  constructor(
    private readonly jwtService: JwtService,
    @Inject(jwtConfig.KEY)
    private readonly jwt: ConfigType<typeof jwtConfig>,
  ) {}

  create(): SocketMiddleware {
    return async (socket: Socket, next) => {
      try {
        // 1. Extrai token: auth.token (Socket.IO padrão) ou Authorization header
        const token =
          socket.handshake.auth?.token ??
          socket.handshake.headers?.authorization?.replace('Bearer ', '');

        if (!token) {
          return next(new Error('Unauthorized: token ausente'));
        }

        // 2. Verifica assinatura e expiração (mesmo secret do JwtAuthGuard HTTP)
        const payload = await this.jwtService.verifyAsync<UserPayload>(token, {
          secret: this.jwt.accessSecret,
        });

        // 3. Popula socket.data.user — disponível em todos os handlers do Gateway
        socket.data.user = {
          id: payload.id,
          email: payload.email,
          roles: payload.roles,
        };

        next();
      } catch (err) {
        this.logger.warn(`WS auth falhou: ${(err as Error).message}`);
        next(new Error('Unauthorized: token inválido ou expirado'));
      }
    };
  }
}
```

---

## 7. Gateway Base com Namespaces

### 7.1 Gateway de Notificações (`/notifications`)

```typescript
// src/modules/notifications/gateways/notifications.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  OnGatewayInit,
  OnGatewayConnection,
  OnGatewayDisconnect,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  WsException,
} from '@nestjs/websockets';
import { UseGuards, Logger, Inject } from '@nestjs/common';
import { Server } from 'socket.io';
import { WsAuthMiddleware } from '@common/middlewares/ws-auth.middleware';
import { WsRolesGuard } from '@common/guards/ws-roles.guard';
import { Roles } from '@common/decorators/roles.decorator';
import { Role } from '@modules/auth/enums/role.enum';
import type { AuthenticatedSocket } from '@common/types/authenticated-socket.type';

/**
 * Gateway de notificações.
 * Namespace: /notifications
 * Autenticação: JWT via handshake.auth.token (middleware no afterInit)
 *
 * Rooms automáticas:
 * - user:<userId>  ← ao conectar, o socket entra nesta room
 */
@WebSocketGateway({
  namespace: '/notifications',
  cors: {
    origin: process.env.WS_CORS_ORIGIN ?? '*',
    credentials: true,
  },
})
export class NotificationsGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server;

  private readonly logger = new Logger(NotificationsGateway.name);

  constructor(private readonly wsAuthMiddleware: WsAuthMiddleware) {}

  // ── Lifecycle ────────────────────────────────────────────────────────────

  afterInit(server: Server) {
    // Registra o middleware de autenticação no namespace ANTES de qualquer conexão
    // Padrão canônico: middleware no afterInit, não no handleConnection
    server.use(this.wsAuthMiddleware.create());
    this.logger.log('NotificationsGateway inicializado com middleware de auth');
  }

  handleConnection(client: AuthenticatedSocket) {
    // Aqui o middleware já executou → client.data.user está populado
    const { user } = client.data;
    this.logger.log(`Cliente conectado: ${client.id} (userId: ${user.id})`);

    // Entra automaticamente na room pessoal do usuário
    // Permite notificações diretas via server.to(`user:${userId}`).emit(...)
    client.join(`user:${user.id}`);
  }

  handleDisconnect(client: AuthenticatedSocket) {
    const userId = client.data?.user?.id ?? 'desconhecido';
    this.logger.log(`Cliente desconectado: ${client.id} (userId: ${userId})`);
  }

  // ── Mensagens (SubscribeMessage) ─────────────────────────────────────────

  @SubscribeMessage('notification:mark-read')
  async handleMarkRead(
    @ConnectedSocket() client: AuthenticatedSocket,
    @MessageBody() data: { notificationId: string },
  ) {
    // client.data.user já está disponível — autenticado pelo middleware
    const { user } = client.data;

    if (!data?.notificationId) {
      throw new WsException('notificationId é obrigatório');
    }

    // Delega ao UseCase (Clean Architecture — Gateway não contém lógica de negócio)
    // await this.markNotificationReadUseCase.execute({ userId: user.id, notificationId: data.notificationId });

    // Confirma para o cliente
    return { event: 'notification:marked-read', data: { notificationId: data.notificationId } };
  }

  // ── Métodos de emissão (usados por UseCases e outros serviços) ────────────

  /**
   * Emite notificação para um usuário específico.
   * Funciona em múltiplas instâncias graças ao RedisIoAdapter.
   */
  emitToUser(userId: string, event: string, payload: unknown): void {
    this.server.to(`user:${userId}`).emit(event, payload);
  }

  /**
   * Emite notificação para todos os usuários com um role específico.
   * Requer que os sockets entrem na room `role:<roleName>` no handleConnection.
   */
  emitToRole(role: Role, event: string, payload: unknown): void {
    this.server.to(`role:${role}`).emit(event, payload);
  }

  /**
   * Broadcast global para todos os clientes conectados ao namespace.
   */
  broadcastAll(event: string, payload: unknown): void {
    this.server.emit(event, payload);
  }
}
```

### 7.2 Gateway de Chat (`/chat`) — Rooms Dinâmicas

```typescript
// src/modules/chat/gateways/chat.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  OnGatewayInit,
  OnGatewayConnection,
  OnGatewayDisconnect,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  WsException,
} from '@nestjs/websockets';
import { Logger } from '@nestjs/common';
import { Server } from 'socket.io';
import { WsAuthMiddleware } from '@common/middlewares/ws-auth.middleware';
import type { AuthenticatedSocket } from '@common/types/authenticated-socket.type';

/**
 * Gateway de chat.
 * Namespace: /chat
 * Rooms: dinâmicas por roomId — o cliente entra via evento 'room:join'
 */
@WebSocketGateway({
  namespace: '/chat',
  cors: {
    origin: process.env.WS_CORS_ORIGIN ?? '*',
    credentials: true,
  },
})
export class ChatGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server;

  private readonly logger = new Logger(ChatGateway.name);

  constructor(private readonly wsAuthMiddleware: WsAuthMiddleware) {}

  afterInit(server: Server) {
    server.use(this.wsAuthMiddleware.create());
    this.logger.log('ChatGateway inicializado com middleware de auth');
  }

  handleConnection(client: AuthenticatedSocket) {
    const { user } = client.data;
    this.logger.log(`Chat: cliente conectado ${client.id} (userId: ${user.id})`);
    // Room pessoal — para mensagens diretas (DM)
    client.join(`user:${user.id}`);
  }

  handleDisconnect(client: AuthenticatedSocket) {
    this.logger.log(`Chat: cliente desconectado ${client.id}`);
  }

  // ── Rooms Dinâmicas ───────────────────────────────────────────────────────

  @SubscribeMessage('room:join')
  async handleJoinRoom(
    @ConnectedSocket() client: AuthenticatedSocket,
    @MessageBody() data: { roomId: string },
  ) {
    if (!data?.roomId) throw new WsException('roomId é obrigatório');

    // Validação de autorização: usuário tem permissão para entrar nesta room?
    // await this.validateRoomAccessUseCase.execute({ userId: client.data.user.id, roomId: data.roomId });

    await client.join(`room:${data.roomId}`);
    this.logger.log(`${client.data.user.id} entrou na room ${data.roomId}`);

    // Notifica outros membros da room
    client.to(`room:${data.roomId}`).emit('room:user-joined', {
      userId: client.data.user.id,
      roomId: data.roomId,
    });

    return { event: 'room:joined', data: { roomId: data.roomId } };
  }

  @SubscribeMessage('room:leave')
  async handleLeaveRoom(
    @ConnectedSocket() client: AuthenticatedSocket,
    @MessageBody() data: { roomId: string },
  ) {
    if (!data?.roomId) throw new WsException('roomId é obrigatório');

    await client.leave(`room:${data.roomId}`);

    client.to(`room:${data.roomId}`).emit('room:user-left', {
      userId: client.data.user.id,
      roomId: data.roomId,
    });

    return { event: 'room:left', data: { roomId: data.roomId } };
  }

  @SubscribeMessage('message:send')
  async handleSendMessage(
    @ConnectedSocket() client: AuthenticatedSocket,
    @MessageBody() data: { roomId: string; content: string },
  ) {
    if (!data?.roomId || !data?.content) {
      throw new WsException('roomId e content são obrigatórios');
    }

    const { user } = client.data;

    // Delega ao UseCase — Gateway não salva mensagem diretamente
    // const message = await this.sendMessageUseCase.execute({ senderId: user.id, roomId: data.roomId, content: data.content });

    // Emite para todos na room (incluindo o remetente)
    this.server.to(`room:${data.roomId}`).emit('message:new', {
      // ...message,
      senderId: user.id,
      roomId: data.roomId,
      content: data.content,
      timestamp: new Date().toISOString(),
    });
  }

  // ── Emissão a partir de Use Cases ─────────────────────────────────────────

  emitToRoom(roomId: string, event: string, payload: unknown): void {
    this.server.to(`room:${roomId}`).emit(event, payload);
  }

  emitToUser(userId: string, event: string, payload: unknown): void {
    this.server.to(`user:${userId}`).emit(event, payload);
  }
}
```

---

## 8. WsRolesGuard — Autorização por Role em `@SubscribeMessage`

> **Quando usar:** O `WsAuthMiddleware` cuida de autenticação (conexão). O `WsRolesGuard` cuida de autorização (qual mensagem o usuário pode enviar). Use `@UseGuards(WsRolesGuard)` + `@Roles()` em handlers específicos.

```typescript
// src/common/guards/ws-roles.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { WsException } from '@nestjs/websockets';
import { ROLES_KEY } from '@common/decorators/roles.decorator'; // já definido em rbac-authorization.md
import { Role } from '@modules/auth/enums/role.enum';
import type { AuthenticatedSocket } from '@common/types/authenticated-socket.type';

/**
 * Guard de autorização para WebSocket.
 * Funciona nos handlers @SubscribeMessage — NÃO no handleConnection.
 * O @UseGuards(JwtAuthGuard) NÃO funciona no contexto WS; use WsAuthMiddleware para auth.
 *
 * Exemplo:
 * @UseGuards(WsRolesGuard)
 * @Roles(Role.ADMIN)
 * @SubscribeMessage('admin:broadcast')
 * handleAdminBroadcast(...) { ... }
 */
@Injectable()
export class WsRolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // Sem @Roles() → handler liberado para qualquer autenticado
    if (!requiredRoles || requiredRoles.length === 0) return true;

    const client: AuthenticatedSocket = context.switchToWs().getClient();
    const { user } = client.data;

    if (!user) {
      throw new WsException('Não autenticado');
    }

    const hasRole = requiredRoles.some((role) => user.roles?.includes(role));
    if (!hasRole) {
      throw new WsException('Sem permissão para esta operação');
    }

    return true;
  }
}
```

---

## 9. WsExceptionFilter — Erros Padronizados

```typescript
// src/common/filters/ws-exception.filter.ts
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseWsExceptionFilter, WsException } from '@nestjs/websockets';
import { Socket } from 'socket.io';

/**
 * Filter global para exceções WebSocket.
 * Converte WsException em um envelope { error: { code, message } }
 * consistente com o padrão de erros HTTP do projeto (rest-api-patterns.md).
 *
 * Registro: app.useGlobalFilters(new WsExceptionFilter()) no main.ts,
 * ou via @UseFilters(WsExceptionFilter) no Gateway.
 */
@Catch(WsException)
export class WsExceptionFilter extends BaseWsExceptionFilter {
  catch(exception: WsException, host: ArgumentsHost) {
    const client: Socket = host.switchToWs().getClient();
    const message = exception.getError();

    client.emit('exception', {
      error: {
        code: 'WS_ERROR',
        message: typeof message === 'string' ? message : JSON.stringify(message),
      },
    });
  }
}
```

---

## 10. Módulo — Registro correto

```typescript
// src/modules/notifications/notifications.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule } from '@nestjs/config';
import { NotificationsGateway } from './gateways/notifications.gateway';
import { WsAuthMiddleware } from '@common/middlewares/ws-auth.middleware';
import jwtConfig from '@config/jwt.config';

@Module({
  imports: [
    // JwtModule e jwtConfig necessários para o WsAuthMiddleware
    JwtModule.register({}),
    ConfigModule,
  ],
  providers: [
    NotificationsGateway,
    WsAuthMiddleware,  // ← obrigatório: injetado no Gateway
  ],
  exports: [
    NotificationsGateway,  // ← exportado para que UseCases possam injetar e emitir
  ],
})
export class NotificationsModule {}
```

> **Dependência circular:** Se um UseCase no módulo A precisa emitir via Gateway do módulo A, injete o Gateway diretamente no UseCase. Se o Gateway do módulo A precisa de um UseCase do módulo B, importe o módulo B. Nunca crie dependências circulares — use `forwardRef()` apenas como último recurso.

---

## 11. Uso nos Use Cases — Emissão a partir de Lógica de Negócio

```typescript
// src/modules/notifications/use-cases/send-notification/send-notification.use-case.ts
import { Injectable } from '@nestjs/common';
import { NotificationsGateway } from '@modules/notifications/gateways/notifications.gateway';
import { NotificationsRepository } from '@modules/notifications/repositories/notifications.repository';

export interface SendNotificationInput {
  userId: string;
  title: string;
  body: string;
  type: string;
}

@Injectable()
export class SendNotificationUseCase {
  constructor(
    private readonly notificationsRepository: NotificationsRepository,
    private readonly notificationsGateway: NotificationsGateway,
  ) {}

  async execute(input: SendNotificationInput): Promise<void> {
    // 1. Persiste no banco (opcional, depende do domínio)
    const notification = await this.notificationsRepository.create({
      userId: input.userId,
      title: input.title,
      body: input.body,
      type: input.type,
    });

    // 2. Emite via WebSocket — funciona em múltiplas instâncias (RedisIoAdapter)
    this.notificationsGateway.emitToUser(input.userId, 'notification:new', {
      id: notification.id,
      title: notification.title,
      body: notification.body,
      type: notification.type,
      createdAt: notification.createdAt,
    });
  }
}
```

---

## 12. Cliente (React Native / Web) — Conexão com Autenticação

```typescript
// Exemplo de cliente — como enviar o JWT no handshake
import { io, Socket } from 'socket.io-client';

function createNotificationsSocket(accessToken: string): Socket {
  return io('https://api.exemplo.com.br/notifications', {
    // Token enviado no handshake — lido pelo WsAuthMiddleware em socket.handshake.auth.token
    auth: { token: accessToken },

    // Reconexão automática — WebSocket não tem isso nativamente como SSE
    reconnection: true,
    reconnectionAttempts: 5,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 5000,
  });
}

// Tratamento de erros de conexão (401 do middleware)
const socket = createNotificationsSocket(getStoredToken());

socket.on('connect_error', (err) => {
  if (err.message === 'Unauthorized: token inválido ou expirado') {
    // Token expirado → refresh e reconecta com novo token
    refreshToken().then((newToken) => {
      socket.auth = { token: newToken };
      socket.connect();
    });
  }
});
```

---

## 13. Rooms e Namespaces — Convenções

| Tipo | Convenção de nome | Exemplo | Criada em |
|---|---|---|---|
| **Room pessoal** | `user:<userId>` | `user:abc123` | `handleConnection` automático |
| **Room de role** | `role:<role>` | `role:admin` | `handleConnection` se tem o role |
| **Room de entidade** | `<entidade>:<id>` | `room:chat42`, `order:xyz` | `handleJoinRoom` explícito |
| **Namespace** | `/<domínio>` | `/chat`, `/notifications` | `@WebSocketGateway({ namespace })` |

```typescript
// Exemplo de emissão por tipo de room a partir de qualquer lugar que injete o Gateway
// Notificação para usuário específico
gateway.emitToUser('user-id-123', 'notification:new', payload);

// Mensagem para uma sala de chat
gateway.emitToRoom('chat42', 'message:new', payload);

// Broadcast para todos os admins
this.server.to('role:admin').emit('admin:alert', payload);
```

---

## 14. Anti-Padrões Comuns

### ❌ Autenticar no `handleConnection` em vez do middleware

```typescript
// ❌ ERRADO — handleConnection não pode rejeitar o handshake de forma limpa
handleConnection(client: Socket) {
  const token = client.handshake.auth.token;
  if (!this.jwtService.verify(token)) {
    client.disconnect(true); // força desconexão APÓS estabelecer conexão (desperdício)
  }
}

// ✅ CORRETO — middleware no afterInit rejeita ANTES de estabelecer a conexão
afterInit(server: Server) {
  server.use(this.wsAuthMiddleware.create()); // rejeita no handshake
}
```

### ❌ Usar `JwtAuthGuard` (HTTP) em `@SubscribeMessage`

```typescript
// ❌ ERRADO — JwtAuthGuard usa context.switchToHttp() → falha em contexto WS
@UseGuards(JwtAuthGuard)
@SubscribeMessage('message:send')
handleMessage(client: Socket, data: any) { ... }

// ✅ CORRETO — WsRolesGuard ou nenhum guard (auth já foi feita no middleware)
@UseGuards(WsRolesGuard)
@Roles(Role.ADMIN)
@SubscribeMessage('admin:broadcast')
handleAdminBroadcast(client: AuthenticatedSocket, data: any) { ... }
```

### ❌ Um namespace único para tudo

```typescript
// ❌ ERRADO — tudo no namespace raiz mistura domínios
@WebSocketGateway()  // namespace padrão '/'
export class AppGateway { ... }  // chat + notifications + presence + ...

// ✅ CORRETO — namespaces separados por domínio
@WebSocketGateway({ namespace: '/notifications' })
export class NotificationsGateway { ... }

@WebSocketGateway({ namespace: '/chat' })
export class ChatGateway { ... }
```

### ❌ Criar conexão Redis dentro do Gateway

```typescript
// ❌ ERRADO — nova conexão Redis por Gateway = vazamento de recursos
export class NotificationsGateway {
  private redis = new Redis(process.env.REDIS_URL); // NÃO faça isso
}

// ✅ CORRETO — RedisIoAdapter gerencia as conexões pub/sub globalmente
// O Gateway simplesmente usa this.server.to(...).emit(...) — o adapter resolve
```

### ❌ Gateway com lógica de negócio

```typescript
// ❌ ERRADO — Gateway acessa banco diretamente (viola Clean Architecture)
@SubscribeMessage('message:send')
async handleMessage(client: AuthenticatedSocket, data: any) {
  const message = await this.messageRepository.save({ ... }); // ERRADO
  this.server.to(data.roomId).emit('message:new', message);
}

// ✅ CORRETO — delega ao UseCase, mesma regra dos Controllers HTTP
@SubscribeMessage('message:send')
async handleMessage(client: AuthenticatedSocket, data: SendMessageDto) {
  const result = await this.sendMessageUseCase.execute({
    senderId: client.data.user.id,
    roomId: data.roomId,
    content: data.content,
  });
  this.server.to(`room:${data.roomId}`).emit('message:new', result);
}
```

### ❌ `socket.io-redis` (pacote descontinuado)

```bash
# ❌ ERRADO — pacote descontinuado
npm install socket.io-redis

# ✅ CORRETO — pacote oficial mantido pela equipe Socket.IO
npm install @socket.io/redis-adapter
```

---

## 15. Testes

### 15.1 Teste Unitário do Gateway

```typescript
// src/modules/notifications/__tests__/notifications.gateway.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { NotificationsGateway } from '../gateways/notifications.gateway';
import { WsAuthMiddleware } from '@common/middlewares/ws-auth.middleware';

const mockServer = {
  use: jest.fn(),
  to: jest.fn().mockReturnThis(),
  emit: jest.fn(),
};

const mockWsAuthMiddleware = {
  create: jest.fn().mockReturnValue(jest.fn()),
};

describe('NotificationsGateway', () => {
  let gateway: NotificationsGateway;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        NotificationsGateway,
        { provide: WsAuthMiddleware, useValue: mockWsAuthMiddleware },
      ],
    }).compile();

    gateway = module.get(NotificationsGateway);
    // Injeta o server mock
    (gateway as any).server = mockServer;
    jest.clearAllMocks();
  });

  describe('afterInit', () => {
    it('deve registrar o middleware de autenticação no server', () => {
      gateway.afterInit(mockServer as any);
      expect(mockWsAuthMiddleware.create).toHaveBeenCalled();
      expect(mockServer.use).toHaveBeenCalled();
    });
  });

  describe('handleConnection', () => {
    it('deve adicionar o socket à room pessoal do usuário', () => {
      const mockClient = {
        id: 'socket-123',
        data: { user: { id: 'user-abc', email: 'user@test.com', roles: [] } },
        join: jest.fn(),
      };

      gateway.handleConnection(mockClient as any);

      expect(mockClient.join).toHaveBeenCalledWith('user:user-abc');
    });
  });

  describe('emitToUser', () => {
    it('deve emitir evento para a room do usuário', () => {
      gateway.emitToUser('user-abc', 'notification:new', { title: 'Teste' });

      expect(mockServer.to).toHaveBeenCalledWith('user:user-abc');
      expect(mockServer.emit).toHaveBeenCalledWith('notification:new', { title: 'Teste' });
    });
  });
});
```

### 15.2 Checklist de Testes

- [ ] `afterInit`: middleware de auth é registrado via `server.use()`?
- [ ] `handleConnection`: socket entra nas rooms corretas?
- [ ] `handleDisconnect`: estado interno é limpo corretamente?
- [ ] Handlers `@SubscribeMessage`: delega ao UseCase sem lógica inline?
- [ ] `emitToUser` / `emitToRoom`: chama `server.to(...).emit(...)` com os args certos?
- [ ] `WsAuthMiddleware`: token ausente → chama `next(new Error(...))`?
- [ ] `WsAuthMiddleware`: token inválido → chama `next(new Error(...))`?
- [ ] `WsAuthMiddleware`: token válido → popula `socket.data.user` e chama `next()`?

---

## 16. Checklist de Implementação

### Configuração
- [ ] `@nestjs/websockets`, `@nestjs/platform-socket.io`, `socket.io` instalados?
- [ ] `@socket.io/redis-adapter` instalado?
- [ ] `RedisIoAdapter` em `src/infra/websocket/redis-io.adapter.ts`?
- [ ] `RedisIoAdapter` registrado em `main.ts` via `app.useWebSocketAdapter()` ANTES do `app.listen()`?
- [ ] `WsAuthMiddleware` em `src/common/middlewares/ws-auth.middleware.ts`?
- [ ] `AuthenticatedSocket` tipagem em `src/common/types/`?

### Por Gateway
- [ ] Namespace explícito via `@WebSocketGateway({ namespace: '/dominio' })`?
- [ ] `WsAuthMiddleware` registrado nos `providers[]` do módulo?
- [ ] Middleware aplicado no `afterInit`, não no `handleConnection`?
- [ ] `handleConnection` apenas entra em rooms (sem lógica de negócio)?
- [ ] Handlers `@SubscribeMessage` delegam ao UseCase?
- [ ] Gateway exportado no módulo para injeção em UseCases?
- [ ] `WsExceptionFilter` aplicado (global ou via `@UseFilters`)?

### Escala
- [ ] `RedisIoAdapter` usa dois clientes Redis dedicados (pub/sub)?
- [ ] Clientes Redis do adapter são **separados** do `AppCacheModule`?
- [ ] Em ambiente sem Redis (`REDIS_URL` ausente), adapter usa fallback in-memory?

### Rooms
- [ ] Room pessoal `user:<userId>` criada no `handleConnection`?
- [ ] Rooms dinâmicas criadas via evento explícito do cliente (`room:join`)?
- [ ] Acesso às rooms validado (usuário tem permissão para entrar)?

---

## Referências

- [auth-jwt-pattern.md](.agent/skills/backend/authentication-and-security/auth-jwt-pattern.md) — `JwtService`, `jwtConfig`, `UserPayload`, guard manual sem Passport
- [caching-strategy.md](.agent/skills/backend/performance-and-infrastructure/caching-strategy.md) — `REDIS_URL`, `AppCacheModule`, `ioredis` já instalado
- [events-pattern.md](.agent/skills/backend/performance-and-infrastructure/events-pattern.md) — `EventEmitter2` para eventos internos; SSE + Redis Pub/Sub como alternativa
- [clean-architecture.md](.agent/skills/backend/foundation-and-architecture/clean-architecture.md) — Gateway na camada Presentation; UseCase como orquestrador
- [nest-project-structure.md](.agent/skills/backend/foundation-and-architecture/nest-project-structure.md) — `src/infra/websocket/`, `src/modules/<domínio>/gateways/`
- [rbac-authorization.md](.agent/skills/backend/authentication-and-security/rbac-authorization.md) — `@Roles()` decorator, `ROLES_KEY` — reutilizado no `WsRolesGuard`
- [testing-strategy-nest.md](.agent/skills/backend/quality-and-testing/testing-strategy-nest.md) — `Test.createTestingModule`, padrão de mocks
- [queue-workers.md](.agent/skills/backend/performance-and-infrastructure/queue-workers.md) — BullMQ + Redis: complementar para side effects que precisam de retry
- [NestJS Docs — Gateways](https://docs.nestjs.com/websockets/gateways)
- [NestJS Docs — Guards WS](https://docs.nestjs.com/websockets/guards)
- [NestJS Docs — Adapters](https://docs.nestjs.com/websockets/adapter)
- [NestJS Docs — Server-Sent Events](https://docs.nestjs.com/techniques/server-sent-events)
- [@socket.io/redis-adapter](https://socket.io/docs/v4/redis-adapter/)
- [Socket.IO — Namespaces](https://socket.io/docs/v4/namespaces/)
- [Socket.IO — Rooms](https://socket.io/docs/v4/rooms/)