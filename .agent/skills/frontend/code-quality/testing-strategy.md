---
name: testing-strategy
description: Estratégia completa de testes para React Native com Expo SDK 54+. Testes unitários com Jest para hooks e utilitários, testes de componentes com Testing Library, mocks de navegação com expo-router/testing-library, mocks de API com MSW, e cobertura mínima por camada.
---

# Estratégia de Testes — React Native com Expo (SDK 54+)

Você é um Engenheiro de Software Senior especialista em qualidade de software para React Native. Sua responsabilidade é garantir que o projeto tenha uma estratégia de testes sólida, cobrindo todas as camadas da aplicação com as ferramentas corretas, seguindo as melhores práticas da comunidade e compatível com o **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

---

## Quando usar esta skill

- **Novo projeto:** Configurar o ambiente de testes do zero.
- **Adição de testes:** Escrever testes para hooks, utilitários, componentes ou telas.
- **Revisão de cobertura:** Verificar e aumentar a cobertura de testes por camada.
- **Mocks de API:** Configurar MSW para interceptar chamadas de rede nos testes.
- **Testes de navegação:** Testar fluxos de tela com Expo Router.

---

## 1. Pirâmide de Testes

```text
           /\
          /E2E\         ← Poucos (Detox / Maestro)
         /------\
        / Integr.\     ← Moderados (renderRouter, MSW)
       /----------\
      /  Unitários  \  ← Muitos (Jest puro, renderHook)
     /--------------\
```

| Camada | Ferramenta | O que testar |
|---|---|---|
| **Unitário** | Jest + `renderHook` | Hooks, stores Zustand, utilitários, formatadores |
| **Componente** | RNTL + `render` | Componentes UI isolados, interações do usuário |
| **Integração** | `renderRouter` + MSW | Fluxos de tela completos com navegação e API |
| **E2E** | Detox / Maestro | Fluxos críticos no dispositivo real (fora do escopo desta skill) |

---

## 2. Instalação e Configuração

### 2.1 Dependências

```bash
npx expo install -- --save-dev \
  jest \
  jest-expo \
  @types/jest \
  @testing-library/react-native \
  @testing-library/jest-native \
  @testing-library/user-event \
  msw \
  react-native-url-polyfill
```

> **Nota SDK 54:** Use sempre `npx expo install` para garantir versões compatíveis com o SDK. O `jest-expo` já inclui o preset correto para React Native 0.81 e New Architecture.

### 2.2 `jest.config.js`

```js
/** @type {import('jest').Config} */
module.exports = {
  preset: 'jest-expo',

  // Transforma pacotes nativos do Expo e React Native
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg)',
  ],

  // Setup global após o framework ser carregado
  setupFilesAfterFramework: ['@testing-library/jest-native/extend-expect'],

  // Setup antes de cada arquivo de teste
  setupFiles: ['./src/test/setup.ts'],

  // Mapeamento de aliases (espelha o tsconfig.json)
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@features/(.*)$': '<rootDir>/src/features/$1',
    '^@hooks/(.*)$': '<rootDir>/src/hooks/$1',
    '^@services/(.*)$': '<rootDir>/src/services/$1',
    '^@theme/(.*)$': '<rootDir>/src/theme/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1',
    '^@config/(.*)$': '<rootDir>/src/config/$1',
    '^@constants/(.*)$': '<rootDir>/src/constants/$1',
    '^@i18n/(.*)$': '<rootDir>/src/i18n/$1',
    // Mock de assets estáticos
    '\\.svg$': '<rootDir>/src/test/__mocks__/svg-mock.js',
  },

  // Cobertura de código
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/__tests__/**',
    '!src/**/__mocks__/**',
    '!src/test/**',
    '!src/app/**',         // Rotas Expo Router — camada fina, sem lógica
  ],

  // Thresholds mínimos por camada (ver seção 7)
  coverageThreshold: {
    // Global
    global: {
      branches: 70,
      functions: 75,
      lines: 75,
      statements: 75,
    },
    // Utilitários e helpers — cobertura alta obrigatória
    './src/utils/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90,
    },
    // Hooks globais — cobertura alta obrigatória
    './src/hooks/': {
      branches: 85,
      functions: 85,
      lines: 85,
      statements: 85,
    },
    // Stores Zustand — cobertura alta obrigatória
    './src/features/': {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },

  // Diretórios de teste
  testMatch: [
    '**/__tests__/**/*.{ts,tsx}',
    '**/*.{spec,test}.{ts,tsx}',
  ],

  // Ignorar pastas de build e rotas
  testPathIgnorePatterns: [
    '/node_modules/',
    '/.expo/',
    '/src/app/',          // Expo Router — não colocar testes aqui
  ],
};
```

### 2.3 Scripts em `package.json`

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --maxWorkers=2"
  }
}
```

---

## 3. Setup Global de Testes

### `src/test/setup.ts`

```ts
import 'react-native-url-polyfill/auto'; // Necessário para MSW no RN
import { server } from './msw/server';

// Inicia o servidor MSW antes de todos os testes
beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }));

// Reseta handlers após cada teste para evitar vazamento entre testes
afterEach(() => server.resetHandlers());

// Fecha o servidor após todos os testes
afterAll(() => server.close());
```

### `src/test/__mocks__/svg-mock.js`

```js
const React = require('react');
const { View } = require('react-native');
const SvgMock = (props) => React.createElement(View, props);
module.exports = SvgMock;
module.exports.default = SvgMock;
module.exports.ReactComponent = SvgMock;
```

---

## 4. Testes Unitários — Hooks e Utilitários

### 4.1 Testando Utilitários Puros

Funções puras em `src/utils/` não precisam de contexto React. Teste diretamente com Jest.

```ts
// src/utils/__tests__/format-date.test.ts
import { formatDate, formatCurrency } from '@utils/format-date';

describe('formatDate', () => {
  it('formata data no padrão brasileiro', () => {
    expect(formatDate(new Date('2025-01-15'))).toBe('15/01/2025');
  });

  it('retorna string vazia para data inválida', () => {
    expect(formatDate(null)).toBe('');
  });
});

describe('formatCurrency', () => {
  it('formata valor em BRL', () => {
    expect(formatCurrency(1234.56)).toBe('R$ 1.234,56');
  });

  it('formata zero corretamente', () => {
    expect(formatCurrency(0)).toBe('R$ 0,00');
  });
});
```

### 4.2 Testando Custom Hooks com `renderHook`

Use `renderHook` do `@testing-library/react-native` para testar hooks em isolamento.

```ts
// src/hooks/__tests__/use-debounce.test.ts
import { renderHook, act } from '@testing-library/react-native';
import { useDebounce } from '@hooks/use-debounce';

describe('useDebounce', () => {
  beforeEach(() => jest.useFakeTimers());
  afterEach(() => jest.useRealTimers());

  it('retorna o valor inicial imediatamente', () => {
    const { result } = renderHook(() => useDebounce('inicial', 500));
    expect(result.current).toBe('inicial');
  });

  it('atualiza o valor após o delay', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: 'inicial' } },
    );

    rerender({ value: 'atualizado' });
    expect(result.current).toBe('inicial'); // ainda não atualizou

    act(() => jest.advanceTimersByTime(500));
    expect(result.current).toBe('atualizado');
  });
});
```

### 4.3 Testando Hooks com Dependências Assíncronas

```ts
// src/features/auth/hooks/__tests__/use-auth.test.ts
import { renderHook, act, waitFor } from '@testing-library/react-native';
import { useAuth } from '@features/auth';

describe('useAuth', () => {
  it('inicia sem usuário autenticado', () => {
    const { result } = renderHook(() => useAuth());
    expect(result.current.user).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
  });

  it('autentica o usuário com credenciais válidas', async () => {
    const { result } = renderHook(() => useAuth());

    await act(async () => {
      await result.current.login({ email: 'user@test.com', password: '123456' });
    });

    await waitFor(() => {
      expect(result.current.isAuthenticated).toBe(true);
      expect(result.current.user?.email).toBe('user@test.com');
    });
  });
});
```

---

## 5. Testes de Stores Zustand

Stores Zustand são objetos JavaScript puros — testáveis sem contexto React.

### 5.1 Padrão de Reset Entre Testes

```ts
// src/features/auth/store/__tests__/auth.store.test.ts
import { act } from '@testing-library/react-native';
import { useAuthStore } from '@features/auth';

// Reseta a store para o estado inicial antes de cada teste
beforeEach(() => {
  useAuthStore.setState(useAuthStore.getInitialState());
});

describe('authStore', () => {
  it('tem estado inicial correto', () => {
    const state = useAuthStore.getState();
    expect(state.user).toBeNull();
    expect(state.token).toBeNull();
    expect(state.isLoading).toBe(false);
  });

  it('define o usuário ao fazer login', () => {
    const mockUser = { id: '1', email: 'user@test.com', name: 'User' };

    act(() => {
      useAuthStore.getState().setUser(mockUser, 'mock-token');
    });

    const state = useAuthStore.getState();
    expect(state.user).toEqual(mockUser);
    expect(state.token).toBe('mock-token');
  });

  it('limpa o estado ao fazer logout', () => {
    act(() => {
      useAuthStore.getState().setUser({ id: '1', email: 'u@t.com', name: 'U' }, 'token');
      useAuthStore.getState().logout();
    });

    const state = useAuthStore.getState();
    expect(state.user).toBeNull();
    expect(state.token).toBeNull();
  });
});
```

> **Dica:** Exponha `getInitialState` na store para facilitar o reset:
> ```ts
> // auth.store.ts
> export const useAuthStore = create<AuthState>()((set, get) => ({
>   user: null,
>   token: null,
>   isLoading: false,
>   // ...actions
> }));
> // Expõe o estado inicial para testes
> useAuthStore.getInitialState = () => ({ user: null, token: null, isLoading: false });
> ```

---

## 6. Testes de Componentes com Testing Library

### 6.1 Princípio fundamental

> **Teste o que o usuário vê e faz, não a implementação interna.**
> Prefira queries por `getByText`, `getByRole`, `getByLabelText` em vez de `getByTestId`.

### 6.2 Hierarquia de Queries (da mais preferida para menos)

| Prioridade | Query | Quando usar |
|---|---|---|
| 1 | `getByRole` | Elementos acessíveis (botões, inputs, headings) |
| 2 | `getByLabelText` | Inputs com label associado |
| 3 | `getByPlaceholderText` | Inputs com placeholder |
| 4 | `getByText` | Texto visível ao usuário |
| 5 | `getByDisplayValue` | Valor atual de input/select |
| 6 | `getByTestId` | Último recurso — use `testID` apenas quando necessário |

### 6.3 Testando Componentes UI

```ts
// src/components/__tests__/button.test.tsx
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react-native';
import { Button } from '@components/button';

describe('Button', () => {
  it('renderiza o texto corretamente', () => {
    render(<Button onPress={jest.fn()}>Confirmar</Button>);
    expect(screen.getByText('Confirmar')).toBeTruthy();
  });

  it('chama onPress ao ser pressionado', () => {
    const onPress = jest.fn();
    render(<Button onPress={onPress}>Confirmar</Button>);
    fireEvent.press(screen.getByText('Confirmar'));
    expect(onPress).toHaveBeenCalledTimes(1);
  });

  it('não chama onPress quando desabilitado', () => {
    const onPress = jest.fn();
    render(<Button onPress={onPress} disabled>Confirmar</Button>);
    fireEvent.press(screen.getByText('Confirmar'));
    expect(onPress).not.toHaveBeenCalled();
  });

  it('exibe estado de loading', () => {
    render(<Button onPress={jest.fn()} loading>Confirmar</Button>);
    expect(screen.getByRole('progressbar')).toBeTruthy();
    expect(screen.queryByText('Confirmar')).toBeNull();
  });
});
```

### 6.4 Testando Componentes com Estado Assíncrono

```ts
// src/features/auth/login/components/__tests__/login-form.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { LoginForm } from '../login-form';

describe('LoginForm', () => {
  it('exibe erro de validação para email inválido', async () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    fireEvent.changeText(screen.getByPlaceholderText('E-mail'), 'email-invalido');
    fireEvent.press(screen.getByText('Entrar'));

    await waitFor(() => {
      expect(screen.getByText('E-mail inválido')).toBeTruthy();
    });
  });

  it('chama onSubmit com dados corretos', async () => {
    const onSubmit = jest.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    fireEvent.changeText(screen.getByPlaceholderText('E-mail'), 'user@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Senha'), '123456');
    fireEvent.press(screen.getByText('Entrar'));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        email: 'user@test.com',
        password: '123456',
      });
    });
  });
});
```

---

## 7. Mocks de Navegação — Expo Router

### 7.1 Usando `expo-router/testing-library`

O Expo Router v4 fornece `renderRouter` — a forma oficial de testar telas com navegação. **Não coloque testes dentro de `src/app/`.**

```ts
// src/features/auth/login/screens/__tests__/login-screen.test.tsx
import React from 'react';
import { screen, fireEvent, waitFor } from '@testing-library/react-native';
import { renderRouter } from 'expo-router/testing-library';

describe('LoginScreen', () => {
  it('navega para home após login bem-sucedido', async () => {
    renderRouter(
      {
        // Sistema de arquivos em memória
        '(auth)/login': require('../login-screen').default,
        '(home)/index': () => <Text>Home</Text>,
      },
      { initialUrl: '/(auth)/login' },
    );

    fireEvent.changeText(screen.getByPlaceholderText('E-mail'), 'user@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Senha'), '123456');
    fireEvent.press(screen.getByText('Entrar'));

    await waitFor(() => {
      expect(screen.getByText('Home')).toBeTruthy();
    });
  });

  it('exibe erro ao falhar no login', async () => {
    renderRouter(
      { '(auth)/login': require('../login-screen').default },
      { initialUrl: '/(auth)/login' },
    );

    fireEvent.changeText(screen.getByPlaceholderText('E-mail'), 'wrong@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Senha'), 'wrong');
    fireEvent.press(screen.getByText('Entrar'));

    await waitFor(() => {
      expect(screen.getByText('Credenciais inválidas')).toBeTruthy();
    });
  });
});
```

### 7.2 Mockando `useRouter` em Componentes Isolados

Para componentes que usam `useRouter` mas não precisam do sistema de roteamento completo:

```ts
// src/test/__mocks__/expo-router.ts
const mockPush = jest.fn();
const mockReplace = jest.fn();
const mockBack = jest.fn();

jest.mock('expo-router', () => ({
  useRouter: () => ({
    push: mockPush,
    replace: mockReplace,
    back: mockBack,
  }),
  useLocalSearchParams: () => ({}),
  usePathname: () => '/',
  Link: ({ children }: { children: React.ReactNode }) => children,
}));

export { mockPush, mockReplace, mockBack };
```

```ts
// Uso no teste
import { mockPush } from '@/test/__mocks__/expo-router';

beforeEach(() => {
  mockPush.mockClear();
});

it('navega para detalhes ao pressionar item', () => {
  render(<ProductCard id="123" name="Produto" />);
  fireEvent.press(screen.getByText('Produto'));
  expect(mockPush).toHaveBeenCalledWith('/products/123');
});
```

### 7.3 Quando usar cada abordagem

| Cenário | Abordagem |
|---|---|
| Testar fluxo completo de tela (login → home) | `renderRouter` |
| Testar componente que usa `useRouter` | Mock manual de `expo-router` |
| Testar hook de navegação customizado | `renderHook` + mock de `expo-router` |

---

## 8. Mocks de API com MSW

### 8.1 Configuração do Servidor MSW

```ts
// src/test/msw/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

### 8.2 Handlers Base

```ts
// src/test/msw/handlers/index.ts
export { authHandlers } from './auth.handlers';
export { userHandlers } from './user.handlers';
```

```ts
// src/test/msw/handlers/auth.handlers.ts
import { http, HttpResponse } from 'msw';

export const authHandlers = [
  http.post('/api/auth/login', async ({ request }) => {
    const body = await request.json() as { email: string; password: string };

    if (body.email === 'user@test.com' && body.password === '123456') {
      return HttpResponse.json({
        user: { id: '1', email: body.email, name: 'Test User' },
        token: 'mock-jwt-token',
      });
    }

    return HttpResponse.json(
      { message: 'Credenciais inválidas' },
      { status: 401 },
    );
  }),

  http.post('/api/auth/logout', () => {
    return HttpResponse.json({ success: true });
  }),

  http.get('/api/auth/me', ({ request }) => {
    const authHeader = request.headers.get('Authorization');

    if (!authHeader?.startsWith('Bearer ')) {
      return HttpResponse.json({ message: 'Não autorizado' }, { status: 401 });
    }

    return HttpResponse.json({
      id: '1',
      email: 'user@test.com',
      name: 'Test User',
    });
  }),
];
```

### 8.3 Sobrescrevendo Handlers em Testes Específicos

```ts
// src/features/auth/login/screens/__tests__/login-screen.test.tsx
import { server } from '@/test/msw/server';
import { http, HttpResponse } from 'msw';

it('exibe mensagem de erro de rede', async () => {
  // Sobrescreve apenas para este teste
  server.use(
    http.post('/api/auth/login', () => {
      return HttpResponse.error(); // Simula falha de rede
    }),
  );

  renderRouter(
    { '(auth)/login': require('../login-screen').default },
    { initialUrl: '/(auth)/login' },
  );

  fireEvent.changeText(screen.getByPlaceholderText('E-mail'), 'user@test.com');
  fireEvent.changeText(screen.getByPlaceholderText('Senha'), '123456');
  fireEvent.press(screen.getByText('Entrar'));

  await waitFor(() => {
    expect(screen.getByText('Erro de conexão. Tente novamente.')).toBeTruthy();
  });
});
```

### 8.4 Factories de Dados de Teste

Crie factories para gerar dados consistentes entre os testes:

```ts
// src/test/factories/user.factory.ts
import type { User } from '@features/auth';

let idCounter = 0;

export function createUser(overrides: Partial<User> = {}): User {
  idCounter++;
  return {
    id: String(idCounter),
    email: `user${idCounter}@test.com`,
    name: `Test User ${idCounter}`,
    createdAt: new Date().toISOString(),
    ...overrides,
  };
}

// Uso:
// const admin = createUser({ role: 'admin' });
// const users = Array.from({ length: 5 }, () => createUser());
```

---

## 9. Organização dos Arquivos de Teste

### Estrutura recomendada (co-localizada)

```text
src/
├── features/
│   └── auth/
│       ├── store/
│       │   ├── auth.store.ts
│       │   └── __tests__/
│       │       └── auth.store.test.ts      ← Testes da store
│       └── login/
│           ├── components/
│           │   ├── login-form.tsx
│           │   └── __tests__/
│           │       └── login-form.test.tsx  ← Testes do componente
│           ├── hooks/
│           │   ├── use-login.ts
│           │   └── __tests__/
│           │       └── use-login.test.ts    ← Testes do hook
│           └── screens/
│               ├── login-screen.tsx
│               └── __tests__/
│                   └── login-screen.test.tsx ← Testes de integração
├── hooks/
│   ├── use-debounce.ts
│   └── __tests__/
│       └── use-debounce.test.ts
├── utils/
│   ├── format-date.ts
│   └── __tests__/
│       └── format-date.test.ts
└── test/                                    ← Infraestrutura de testes
    ├── setup.ts
    ├── factories/
    │   └── user.factory.ts
    ├── __mocks__/
    │   └── svg-mock.js
    └── msw/
        ├── server.ts
        └── handlers/
            ├── index.ts
            ├── auth.handlers.ts
            └── user.handlers.ts
```

> **Regra:** Nunca coloque testes dentro de `src/app/`. O diretório `app/` é exclusivo para rotas do Expo Router.

---

## 10. Cobertura Mínima por Camada

| Camada | Branches | Functions | Lines | Justificativa |
|---|---|---|---|---|
| `src/utils/` | **90%** | **90%** | **90%** | Funções puras, fáceis de testar |
| `src/hooks/` | **85%** | **85%** | **85%** | Lógica reutilizável crítica |
| `src/features/**/store/` | **80%** | **80%** | **80%** | Estado global — alto impacto |
| `src/features/**/hooks/` | **80%** | **80%** | **80%** | Lógica de domínio |
| `src/features/**/components/` | **70%** | **75%** | **75%** | Componentes de feature |
| `src/components/` | **75%** | **80%** | **80%** | UI Kit compartilhado |
| Global | **70%** | **75%** | **75%** | Mínimo aceitável |

### Gerando o relatório de cobertura

```bash
# Relatório no terminal
npm run test:coverage

# Relatório HTML (abrir coverage/lcov-report/index.html)
npm run test:coverage -- --coverageReporters=html
```

---

## 11. Integração com CI/CD e Husky

### Adicionando testes ao pre-push (`.husky/pre-push`)

```bash
npm run test:ci
```

> **Por que pre-push e não pre-commit?** Testes são mais lentos que lint. Rodá-los no pre-push garante que o código que vai para o repositório está testado, sem frustrar o desenvolvedor a cada commit local.

### Script `test:ci` otimizado

```json
{
  "scripts": {
    "test:ci": "jest --ci --coverage --maxWorkers=2 --forceExit"
  }
}
```

- `--ci`: Modo CI (falha se snapshots estão desatualizados, não atualiza automaticamente)
- `--coverage`: Gera relatório e aplica thresholds
- `--maxWorkers=2`: Limita workers para ambientes com poucos recursos
- `--forceExit`: Garante que o processo encerra após os testes (evita travamentos com MSW)

---

## 12. Padrões e Anti-Padrões

### ✅ Faça

```ts
// ✅ Query por role (acessível)
screen.getByRole('button', { name: 'Entrar' });

// ✅ Envolva updates de estado em act()
await act(async () => { await result.current.login(credentials); });

// ✅ Use waitFor para assertions assíncronas
await waitFor(() => expect(screen.getByText('Bem-vindo!')).toBeTruthy());

// ✅ Resete a store entre testes
beforeEach(() => useAuthStore.setState(useAuthStore.getInitialState()));

// ✅ Use factories para dados de teste
const user = createUser({ role: 'admin' });
```

### ❌ Evite

```ts
// ❌ Não teste detalhes de implementação
expect(component.state.isLoading).toBe(true); // Acesso direto ao estado interno

// ❌ Não use testID como primeira opção
screen.getByTestId('login-button'); // Use getByRole ou getByText

// ❌ Não faça assertions em snapshots de componentes complexos
expect(component).toMatchSnapshot(); // Frágil e difícil de manter

// ❌ Não compartilhe estado entre testes
let store; // Store criada fora do beforeEach — polui outros testes
beforeAll(() => { store = createStore(); });

// ❌ Não coloque testes em src/app/
// src/app/__tests__/login.test.tsx ← ERRADO
```

---

## 10. Testando com TanStack Query v5

Hooks que usam `useQuery` ou `useMutation` precisam de um `QueryClientProvider` no wrapper de teste. Use um helper `createWrapper` para reutilizar entre testes.

### 10.1 Helper `createWrapper`

```ts
// src/test/create-wrapper.tsx
import React from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

/**
 * Cria um wrapper com QueryClientProvider isolado para cada teste.
 * - retry: false — evita retries que causam timeout nos testes
 * - gcTime: Infinity — evita erro "Jest did not exit after tests"
 */
export function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,      // sem retries — testes falham rápido
        gcTime: Infinity,  // evita cleanup async que quebra o Jest
      },
      mutations: {
        retry: false,
      },
    },
  });

  const Wrapper = ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );

  return { Wrapper, queryClient };
}
```

### 10.2 Testando `useQuery`

```ts
// src/features/products/hooks/__tests__/use-products.test.ts
import { renderHook, waitFor } from '@testing-library/react-native';
import { createWrapper } from '@test/create-wrapper';
import { useProducts } from '../use-products';
import { server } from '@test/setup';
import { http, HttpResponse } from 'msw';

describe('useProducts', () => {
  it('retorna lista de produtos com sucesso', async () => {
    const { Wrapper } = createWrapper();
    const { result } = renderHook(() => useProducts(), { wrapper: Wrapper });

    expect(result.current.isLoading).toBe(true);

    await waitFor(() => expect(result.current.isSuccess).toBe(true));

    expect(result.current.data).toHaveLength(3);
  });

  it('expõe erro tipado quando a API falha', async () => {
    server.use(
      http.get('/api/products', () =>
        HttpResponse.json({ message: 'Erro interno' }, { status: 500 })
      )
    );

    const { Wrapper } = createWrapper();
    const { result } = renderHook(() => useProducts(), { wrapper: Wrapper });

    await waitFor(() => expect(result.current.isError).toBe(true));

    // error é ApiError tipado — sem cast
    expect(result.current.error?.type).toBe('SERVER_ERROR');
  });
});
```

### 10.3 Testando `useMutation`

```ts
// src/features/auth/hooks/__tests__/use-login.test.ts
import { renderHook, act, waitFor } from '@testing-library/react-native';
import { createWrapper } from '@test/create-wrapper';
import { useLogin } from '../use-login';
import { server } from '@test/setup';
import { http, HttpResponse } from 'msw';

describe('useLogin', () => {
  it('chama onSuccess após login correto', async () => {
    const { Wrapper } = createWrapper();
    const { result } = renderHook(() => useLogin(), { wrapper: Wrapper });

    act(() => {
      result.current.mutate({ email: 'user@test.com', password: '12345678' });
    });

    expect(result.current.isPending).toBe(true);

    await waitFor(() => expect(result.current.isSuccess).toBe(true));

    expect(result.current.data?.token).toBeDefined();
  });

  it('expõe ApiError tipado para credenciais inválidas', async () => {
    server.use(
      http.post('/api/auth/login', () =>
        HttpResponse.json({ message: 'Credenciais inválidas' }, { status: 401 })
      )
    );

    const { Wrapper } = createWrapper();
    const { result } = renderHook(() => useLogin(), { wrapper: Wrapper });

    act(() => {
      result.current.mutate({ email: 'wrong@test.com', password: 'wrong' });
    });

    await waitFor(() => expect(result.current.isError).toBe(true));

    // error é ApiError tipado — sem cast `as`
    expect(result.current.error?.type).toBe('UNAUTHORIZED');
  });
});
```

### 10.4 Testando componentes que usam TanStack Query

```ts
import { render, screen, waitFor } from '@testing-library/react-native';
import { createWrapper } from '@test/create-wrapper';
import { ProductsScreen } from '../products-screen';

describe('ProductsScreen', () => {
  it('exibe produtos após carregar', async () => {
    const { Wrapper } = createWrapper();
    render(<ProductsScreen />, { wrapper: Wrapper });

    expect(screen.getByTestId('loading-spinner')).toBeTruthy();

    await waitFor(() => {
      expect(screen.getByText('Produto 1')).toBeTruthy();
    });
  });
});
```

---

## 13. Checklist de Verificação

### Configuração
- [ ] `jest-expo` instalado e configurado como preset?
- [ ] `transformIgnorePatterns` configurado para pacotes Expo/RN?
- [ ] `moduleNameMapper` espelhando todos os aliases do `tsconfig.json`?
- [ ] `setupFilesAfterFramework` com `@testing-library/jest-native/extend-expect`?
- [ ] MSW configurado em `src/test/setup.ts` com `beforeAll/afterEach/afterAll`?
- [ ] `react-native-url-polyfill/auto` importado no setup (necessário para MSW)?

### Testes Unitários e TanStack Query
- [ ] Utilitários em `src/utils/` com cobertura ≥ 90%?
- [ ] Hooks globais em `src/hooks/` com cobertura ≥ 85%?
- [ ] Stores Zustand testadas com reset entre testes (`getInitialState`)?
- [ ] Updates de estado envolvidos em `act()`?
- [ ] Operações assíncronas usando `waitFor`?
- [ ] Helper `createWrapper()` criado em `src/test/create-wrapper.tsx`?
- [ ] `QueryClient` de teste configurado com `retry: false` e `gcTime: Infinity`?
- [ ] Hooks com `useQuery` testados via `renderHook` + `Wrapper` do `createWrapper`?
- [ ] Hooks com `useMutation` testados com `act()` + `waitFor`?
- [ ] `mutation.error` verificado como `ApiError` tipado nos testes?

### Testes de Componentes
- [ ] Queries priorizando `getByRole` e `getByText` sobre `getByTestId`?
- [ ] Interações usando `fireEvent` ou `userEvent`?
- [ ] Nenhum teste acessando estado interno do componente?
- [ ] Componentes que usam TanStack Query renderizados com `wrapper: Wrapper` do `createWrapper`?

### Testes de Integração
- [ ] Telas testadas com `renderRouter` (não `render` simples)?
- [ ] Nenhum teste dentro de `src/app/`?
- [ ] Handlers MSW cobrindo cenários de sucesso e erro?
- [ ] Factories de dados usadas para consistência?

### CI/CD
- [ ] Script `test:ci` configurado com `--ci --coverage --forceExit`?
- [ ] Pre-push hook rodando `test:ci`?
- [ ] Thresholds de cobertura configurados no `jest.config.js`?
- [ ] Pipeline de CI executando `test:ci` antes do build?
