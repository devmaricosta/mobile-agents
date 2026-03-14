---
name: react-query-patterns
description: Convenções de uso do TanStack Query v5 no projeto React Native com Expo SDK 54 — fábricas de query keys, separação de queries/ e mutations/ por feature, estratégias de invalidação de cache e padrão de optimistic updates integrado com Result<T, E> e ApiError tipado.
---

# Padrões TanStack Query — React Native com Expo (SDK 54+)

Você é um Engenheiro de Software Senior especialista em gerenciamento de estado de servidor com TanStack Query v5 em projetos React Native. Sua responsabilidade é garantir que **todo dado que vem do servidor** seja gerenciado exclusivamente pelo TanStack Query, seguindo as convenções de nomenclatura, organização de arquivos e estratégias de cache definidas neste documento, compatíveis com **Expo SDK 54** (React Native 0.81, React 19.1, TypeScript ~5.9).

---

## Quando usar esta skill

- **Novo `useQuery` ou `useMutation`:** Decidir onde o arquivo vai e como nomear a query key.
- **Invalidação de cache:** Após uma mutation, qual estratégia usar para manter a UI sincronizada.
- **Optimistic updates:** Implementar atualização otimista com rollback automático.
- **Organização de arquivos:** Separar `queries/` de `mutations/` dentro de uma feature.
- **Code review:** Validar se hooks de dados seguem as convenções do projeto.

---

## 1. Instalação e Configuração do Provider

### 1.1 Dependências

```bash
npx expo install @tanstack/react-query
```

> **SDK 54:** O TanStack Query v5 requer React 18+. O Expo SDK 54 usa React 19.1, portanto é totalmente compatível.

### 1.2 Configuração Global — `QueryClient`

```ts
// src/services/query-client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Dados ficam "frescos" por 2 minutos — evita refetches desnecessários
      staleTime: 2 * 60 * 1000,

      // Cache mantido por 5 minutos após componente desmontar (padrão)
      gcTime: 5 * 60 * 1000,

      // Retry inteligente: 2 tentativas com backoff exponencial
      // (o interceptor do Axios já trata 401/403 — não fazer retry neles)
      retry: 2,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30_000),

      // Refetch ao reconectar à rede (importante em mobile)
      refetchOnReconnect: true,

      // Não refetch ao focar a janela (comportamento web — irrelevante em mobile)
      refetchOnWindowFocus: false,
    },
    mutations: {
      // Não faz retry em mutations por padrão (evita duplicar ações)
      retry: false,
    },
  },
});
```

### 1.3 Provider no Layout Raiz

```tsx
// src/app/_layout.tsx
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@services/query-client';

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      {/* ... outros providers (Zustand não precisa de Provider) */}
      <Stack />
    </QueryClientProvider>
  );
}
```

---

## 2. Query Key Factories — Nomenclatura e Organização

### 2.1 Conceito

Query key factories são **objetos com métodos** que geram arrays de query keys de forma consistente e type-safe. Eles permitem invalidações precisas por prefixo sem hardcoding de strings espalhadas pelo código.

### 2.2 Localização dos Arquivos

> **Regra de localização:** Query keys pertencem à feature que **possui** aquela entidade. A localização dentro da feature é `queries/query-keys.ts`, seguindo o mesmo princípio de co-localização usado para stores Zustand.
>
> **Exceção — Cross-feature:** Para entidades consultadas por **3 ou mais features distintas** sem um dono claro (ex: `userKeys` usados em `profile`, `settings`, `notifications`), use `src/constants/query-keys.ts`. Este arquivo deve ser a exceção, não a regra.

Cada feature define sua própria fábrica de query keys **dentro da própria feature**:

```text
src/features/products/
├── queries/
│   ├── query-keys.ts           ← Fábrica de query keys da feature
│   ├── use-products.ts         ← useQuery: listar produtos
│   └── use-product-detail.ts   ← useQuery: detalhe de um produto
├── mutations/
│   ├── use-create-product.ts   ← useMutation: criar produto
│   ├── use-update-product.ts   ← useMutation: atualizar produto
│   └── use-delete-product.ts   ← useMutation: deletar produto
├── services/
│   └── products.service.ts     ← Funções puras com Result<T, E>
└── types/
    └── products.types.ts
```

### 2.3 Padrão de Query Key Factory

Cada nível da fábrica chama o nível acima como prefixo, permitindo invalidação hierárquica:

```ts
// src/features/products/queries/query-keys.ts

export const productKeys = {
  // Raiz: invalida TUDO relacionado a produtos
  all: () => ['products'] as const,

  // Lista: invalida todas as listas (com ou sem filtros)
  lists: () => [...productKeys.all(), 'list'] as const,

  // Lista com filtros específicos (ex: página, categoria)
  list: (filters: ProductFilters) =>
    [...productKeys.lists(), filters] as const,

  // Detalhe: invalida todos os detalhes
  details: () => [...productKeys.all(), 'detail'] as const,

  // Detalhe de um item específico
  detail: (id: string) => [...productKeys.details(), id] as const,
} as const;
```

#### Hierarquia de invalidação

```text
productKeys.all()         → ['products']
productKeys.lists()       → ['products', 'list']
productKeys.list(filters) → ['products', 'list', { page: 1, category: 'food' }]
productKeys.details()     → ['products', 'detail']
productKeys.detail('123') → ['products', 'detail', '123']

Invalidar productKeys.all()     → invalida TUDO acima
Invalidar productKeys.lists()   → invalida apenas listas
Invalidar productKeys.detail(id)→ invalida apenas aquele item
```

### 2.4 Convenção de Nomenclatura

| Tipo | Convenção | Exemplo |
|---|---|---|
| Arquivo de factory | `query-keys.ts` dentro de `queries/` | `products/queries/query-keys.ts` |
| Nome do objeto | camelCase + `Keys` suffix | `productKeys`, `authKeys`, `orderKeys` |
| Método raiz | `all()` | `productKeys.all()` |
| Método de lista | `lists()` + `list(filters)` | `productKeys.lists()` |
| Método de detalhe | `details()` + `detail(id)` | `productKeys.detail('123')` |

---

## 3. Queries — `useQuery`

### 3.1 Localização e Nomenclatura de Arquivos

```text
features/<feature>/queries/use-<recurso>.ts
features/<feature>/queries/use-<recurso>-detail.ts
features/<feature>/queries/use-<recurso>-by-<campo>.ts
```

### 3.2 Padrão de `useQuery` com `Result<T, E>`

O `queryFn` **sempre** chama uma função de serviço que retorna `Result<T, ApiError>`. O `Result` é desempacotado dentro do `queryFn`, e o erro é relançado para que o TanStack Query possa tratá-lo:

```ts
// src/features/products/queries/use-products.ts
import { useQuery } from '@tanstack/react-query';
import type { UseQueryResult } from '@tanstack/react-query';
import type { ApiError } from '@services/api/errors';
import { getProducts } from '../services/products.service';
import type { Product, ProductFilters } from '../types/products.types';
import { productKeys } from './query-keys';

export function useProducts(
  filters: ProductFilters = {},
): UseQueryResult<Product[], ApiError> {
  return useQuery<Product[], ApiError>({
    queryKey: productKeys.list(filters),
    queryFn: async () => {
      const result = await getProducts(filters);
      // Desempacota o Result<T, E>: lança ApiError tipado se Err
      if (result.isErr()) throw result.error;
      return result.value;
    },
  });
}
```

```ts
// src/features/products/queries/use-product-detail.ts
import { useQuery } from '@tanstack/react-query';
import type { UseQueryResult } from '@tanstack/react-query';
import type { ApiError } from '@services/api/errors';
import { getProductById } from '../services/products.service';
import type { ProductDetail } from '../types/products.types';
import { productKeys } from './query-keys';

export function useProductDetail(
  id: string,
): UseQueryResult<ProductDetail, ApiError> {
  return useQuery<ProductDetail, ApiError>({
    queryKey: productKeys.detail(id),
    queryFn: async () => {
      const result = await getProductById(id);
      if (result.isErr()) throw result.error;
      return result.value;
    },
    // Não busca se o id for inválido
    enabled: Boolean(id),
  });
}
```

### 3.3 Consumo em Telas

```tsx
// src/features/products/screens/products-screen.tsx
import { useProducts } from '../queries/use-products';
import type { ApiError } from '@services/api/errors';
import { ErrorState } from '@components/error-state';

export function ProductsScreen() {
  const { data: products, isLoading, error, refetch } = useProducts();

  if (isLoading) return <LoadingSpinner />;

  // error é ApiError tipado — sem cast, sem unknown
  if (error) return <ErrorState error={error} onRetry={refetch} />;

  return <ProductList products={products} />;
}
```

---

## 4. Mutations — `useMutation`

### 4.1 Localização e Nomenclatura de Arquivos

```text
features/<feature>/mutations/use-create-<recurso>.ts
features/<feature>/mutations/use-update-<recurso>.ts
features/<feature>/mutations/use-delete-<recurso>.ts
```

### 4.2 Padrão Base de `useMutation`

```ts
// src/features/products/mutations/use-create-product.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import type { UseMutationResult } from '@tanstack/react-query';
import type { ApiError } from '@services/api/errors';
import { createProduct } from '../services/products.service';
import type { CreateProductPayload, Product } from '../types/products.types';
import { productKeys } from '../queries/query-keys';

// useMutation<TData, TError, TVariables>
export function useCreateProduct(): UseMutationResult<
  Product,
  ApiError,
  CreateProductPayload
> {
  const queryClient = useQueryClient();

  return useMutation<Product, ApiError, CreateProductPayload>({
    mutationFn: async (payload) => {
      const result = await createProduct(payload);
      if (result.isErr()) throw result.error;
      return result.value;
    },
    onSuccess: () => {
      // Invalida TODAS as listas de produtos — próximo acesso refaz o fetch
      queryClient.invalidateQueries({ queryKey: productKeys.lists() });
    },
    onError: (error) => {
      // error é ApiError tipado — trate apenas o que não for tratado na UI
      if (error.type === 'UNKNOWN') {
        console.error('[useCreateProduct]', error);
      }
    },
  });
}
```

### 4.3 Consumo em Hooks de Formulário

```ts
// src/features/products/hooks/use-product-form.ts
import { useCreateProduct } from '../mutations/use-create-product';
import { useErrorFeedback } from '@hooks/use-error-feedback';

export function useProductForm() {
  const { mutate, isPending, error } = useCreateProduct();
  const { getErrorMessage } = useErrorFeedback();

  function handleSubmit(payload: CreateProductPayload) {
    mutate(payload, {
      // Callbacks locais sobrepõem os do useMutation — útil para navegação
      onSuccess: (createdProduct) => {
        router.push(ROUTES.PRODUCT_DETAIL(createdProduct.id));
      },
    });
  }

  return {
    handleSubmit,
    isPending,
    // Converte ApiError → mensagem amigável para exibir na UI
    errorMessage: error ? getErrorMessage(error) : null,
  };
}
```

---

## 5. Estratégias de Invalidação de Cache

### 5.1 Tabela de Decisão

| Situação | Estratégia | Exemplo |
|---|---|---|
| Criar um item | Invalida a lista inteira | `productKeys.lists()` |
| Atualizar um item | Invalida o detalhe + lista | `productKeys.detail(id)` + `productKeys.lists()` |
| Deletar um item | Invalida a lista + remove do cache | `queryClient.removeQueries` + `productKeys.lists()` |
| Ação que afeta múltiplas entidades | Invalida pela raiz | `productKeys.all()` |
| Ação que afeta o usuário logado | Invalida dados de auth | `authKeys.detail(userId)` |

### 5.2 Invalidação Simples (Pós-Mutation)

```ts
// Após criar → invalida todas as listas
queryClient.invalidateQueries({ queryKey: productKeys.lists() });

// Após atualizar → invalida lista + detalhe específico
queryClient.invalidateQueries({ queryKey: productKeys.lists() });
queryClient.invalidateQueries({ queryKey: productKeys.detail(id) });

// Após deletar → remove do cache e invalida lista
queryClient.removeQueries({ queryKey: productKeys.detail(id) });
queryClient.invalidateQueries({ queryKey: productKeys.lists() });
```

### 5.3 Atualização Direta do Cache (setQueryData)

Use `setQueryData` quando quiser evitar um round-trip à API e já tem o dado atualizado:

```ts
// src/features/products/mutations/use-update-product.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import type { ApiError } from '@services/api/errors';
import { updateProduct } from '../services/products.service';
import type { UpdateProductPayload, Product } from '../types/products.types';
import { productKeys } from '../queries/query-keys';

export function useUpdateProduct(id: string) {
  const queryClient = useQueryClient();

  return useMutation<Product, ApiError, UpdateProductPayload>({
    mutationFn: async (payload) => {
      const result = await updateProduct(id, payload);
      if (result.isErr()) throw result.error;
      return result.value;
    },
    onSuccess: (updatedProduct) => {
      // Atualiza o cache do detalhe diretamente — sem refetch
      queryClient.setQueryData<Product>(
        productKeys.detail(id),
        updatedProduct,
      );
      // Invalida as listas para garantir consistência
      queryClient.invalidateQueries({ queryKey: productKeys.lists() });
    },
  });
}
```

### 5.4 Invalidação Cross-Feature

Quando uma mutation em uma feature afeta dados de outra, importe a query key factory da feature afetada via barrel público:

```ts
// src/features/orders/mutations/use-place-order.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
// ✅ Importa a factory via barrel público da feature cart
import { cartKeys } from '@features/cart';
import { orderKeys } from '../queries/query-keys';

export function usePlaceOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: placeOrder,
    onSuccess: () => {
      // Invalida os pedidos do usuário
      queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
      // Invalida o carrinho (feature diferente)
      queryClient.invalidateQueries({ queryKey: cartKeys.all() });
    },
  });
}
```

> **Regra:** Exporte `queryKeys` no barrel público (`index.ts`) da feature se outras features precisarem invalidá-las.

---

## 6. Optimistic Updates

### 6.1 Quando Usar

Use optimistic updates em ações frequentes e de baixa latência onde o sucesso é esperado (ex: curtir, reordenar, marcar como favorito). **Evite** em operações críticas com alta chance de falha (ex: pagamento, envio de formulário com validação complexa).

### 6.2 Padrão Completo com Rollback

```ts
// src/features/products/mutations/use-toggle-favorite.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import type { ApiError } from '@services/api/errors';
import { toggleFavorite } from '../services/products.service';
import type { Product } from '../types/products.types';
import { productKeys } from '../queries/query-keys';

export function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation<Product, ApiError, { id: string; isFavorite: boolean }>({
    mutationFn: async ({ id, isFavorite }) => {
      const result = await toggleFavorite(id, isFavorite);
      if (result.isErr()) throw result.error;
      return result.value;
    },

    // 1️⃣ onMutate: atualiza o cache ANTES da resposta do servidor
    onMutate: async ({ id, isFavorite }) => {
      // Cancela refetches em andamento para não sobrescrever o estado otimista
      await queryClient.cancelQueries({ queryKey: productKeys.detail(id) });
      await queryClient.cancelQueries({ queryKey: productKeys.lists() });

      // Salva o estado anterior para rollback
      const previousDetail = queryClient.getQueryData<Product>(
        productKeys.detail(id),
      );
      const previousList = queryClient.getQueryData<Product[]>(
        productKeys.lists(),
      );

      // Atualiza o cache otimisticamente
      queryClient.setQueryData<Product>(productKeys.detail(id), (old) =>
        old ? { ...old, isFavorite } : old,
      );
      queryClient.setQueryData<Product[]>(productKeys.lists(), (old) =>
        old?.map((p) => (p.id === id ? { ...p, isFavorite } : p)),
      );

      // Retorna o contexto para rollback em caso de erro
      return { previousDetail, previousList };
    },

    // 2️⃣ onError: reverte para o estado anterior (rollback)
    onError: (_error, { id }, context) => {
      if (context?.previousDetail) {
        queryClient.setQueryData(productKeys.detail(id), context.previousDetail);
      }
      if (context?.previousList) {
        queryClient.setQueryData(productKeys.lists(), context.previousList);
      }
    },

    // 3️⃣ onSettled: sempre sincroniza com o servidor ao final (sucesso ou erro)
    onSettled: (_data, _error, { id }) => {
      queryClient.invalidateQueries({ queryKey: productKeys.detail(id) });
      queryClient.invalidateQueries({ queryKey: productKeys.lists() });
    },
  });
}
```

### 6.3 Optimistic Update via `variables` (Abordagem Simples v5)

O TanStack Query v5 introduziu uma forma mais simples de optimistic updates **sem alterar o cache diretamente**. Use quando a lista pode ser reconstruída a partir das `variables` do `useMutation`:

```tsx
// src/features/products/screens/products-screen.tsx
import { useProducts } from '../queries/use-products';
import { useToggleFavorite } from '../mutations/use-toggle-favorite';

export function ProductsScreen() {
  const { data: products } = useProducts();
  const { mutate, isPending, variables } = useToggleFavorite();

  return (
    <FlatList
      data={products}
      renderItem={({ item }) => {
        // Se esta mutation está pendente PARA este item, mostra o estado otimista
        const isOptimisticFavorite =
          isPending && variables?.id === item.id
            ? variables.isFavorite
            : item.isFavorite;

        return (
          <ProductCard
            product={item}
            isFavorite={isOptimisticFavorite}
            onToggleFavorite={(isFav) => mutate({ id: item.id, isFavorite: isFav })}
          />
        );
      }}
    />
  );
}
```

> **Quando usar cada abordagem:**
> - **`variables` + `isPending`:** Lista simples, estado derivado de `variables`. Mais simples, sem risco de cache inconsistente.
> - **`onMutate` + `setQueryData`:** Cenários complexos com múltiplas queries afetadas, ou quando a UI precisa do cache atualizado (ex: navegação para detalhe imediatamente após).

---

## 7. Integração com `Result<T, E>` e `ApiError`

O padrão de sempre desempacotar o `Result` **dentro do `queryFn`** ou **`mutationFn`** mantém a tipagem do erro propagada pelo TanStack Query como `ApiError` (não `unknown` nem `Error`):

```ts
// ✅ CORRETO — desempacota Result dentro do queryFn
return useQuery<Product[], ApiError>({
  queryKey: productKeys.lists(),
  queryFn: async () => {
    const result = await getProducts();
    if (result.isErr()) throw result.error; // lança ApiError tipado
    return result.value;
  },
});

// ❌ ERRADO — não desempacote fora do queryFn
const result = await getProducts();
if (result.isOk()) {
  // Não têm como passar dados tipados para o useQuery dessa forma
}
```

```ts
// src/features/products/services/products.service.ts
import { fromAxiosPromise, type ApiResultAsync } from '@services/api/result';
import { api } from '@services/api/client';
import type { Product, ProductFilters, CreateProductPayload } from '../types/products.types';

export function getProducts(filters: ProductFilters = {}): ApiResultAsync<Product[]> {
  return fromAxiosPromise(
    api.get<Product[]>('/products', { params: filters }).then((r) => r.data),
  );
}

export function getProductById(id: string): ApiResultAsync<Product> {
  return fromAxiosPromise(
    api.get<Product>(`/products/${id}`).then((r) => r.data),
  );
}

export function createProduct(payload: CreateProductPayload): ApiResultAsync<Product> {
  return fromAxiosPromise(
    api.post<Product>('/products', payload).then((r) => r.data),
  );
}
```

---

## 8. Exportações pelo Barrel Público (`index.ts`)

Exporte do barrel da feature **apenas** o que outras features podem precisar (hooks, types e query keys quando necessário para cross-feature invalidation):

```ts
// src/features/products/index.ts
// Screens
export { ProductsScreen } from './screens/products-screen';
export { ProductDetailScreen } from './screens/product-detail-screen';

// Hooks públicos — outros domínios podem precisar
export { useProducts } from './queries/use-products';

// Query keys — exporte apenas se outras features precisam invalidar
export { productKeys } from './queries/query-keys';

// Types públicos
export type { Product, ProductFilters } from './types/products.types';

// ❌ NÃO exporte mutations internas nem services
// export { useCreateProduct } from './mutations/use-create-product'; // interno
// export { getProducts } from './services/products.service'; // interno
```

---

## 9. Organização Completa de Arquivos

```text
src/
├── services/
│   └── query-client.ts          ← QueryClient global com defaultOptions
│
└── features/
    └── products/
        ├── queries/
        │   ├── query-keys.ts            ← Fábrica de query keys
        │   ├── use-products.ts          ← useQuery: lista
        │   └── use-product-detail.ts    ← useQuery: detalhe
        ├── mutations/
        │   ├── use-create-product.ts    ← useMutation: criar
        │   ├── use-update-product.ts    ← useMutation: atualizar
        │   ├── use-delete-product.ts    ← useMutation: deletar
        │   └── use-toggle-favorite.ts   ← useMutation: com optimistic update
        ├── services/
        │   └── products.service.ts      ← Funções puras → Result<T, E>
        ├── hooks/
        │   └── use-product-form.ts      ← Hook de formulário (orquestra mutation)
        ├── components/
        ├── screens/
        ├── types/
        │   └── products.types.ts
        └── index.ts                     ← Barrel público
```

---

## 10. Padrões e Anti-Padrões

### ✅ Faça

```ts
// ✅ Use query key factories — nunca strings soltas
queryClient.invalidateQueries({ queryKey: productKeys.lists() });

// ✅ Tipar explicitamente useQuery e useMutation com ApiError
useQuery<Product[], ApiError>({ ... });
useMutation<Product, ApiError, CreateProductPayload>({ ... });

// ✅ Sempre desempacote Result<T, E> dentro de queryFn / mutationFn
queryFn: async () => {
  const result = await getProducts();
  if (result.isErr()) throw result.error;
  return result.value;
},

// ✅ Use onSettled para garantir sincronização após optimistic updates
onSettled: () => queryClient.invalidateQueries({ queryKey: productKeys.lists() }),

// ✅ Separe queries/ e mutations/ em pastas distintas dentro da feature
```

### ❌ Evite

```ts
// ❌ Não use strings literais como query keys
useQuery({ queryKey: ['products', 'list'], ... }); // não rastreável

// ❌ Não misture queries e mutations no mesmo arquivo
// use-products.ts com useCreateProduct() dentro — separe!

// ❌ Não use useState para cachear dados de API
const [products, setProducts] = useState([]);
useEffect(() => fetch('/products').then(setProducts), []);

// ❌ Não invalide pela raiz quando puder ser preciso
queryClient.invalidateQueries(); // invalida TUDO — pesado demais

// ❌ Não passe `error` como `unknown` — sempre tipar
useQuery({ queryKey: ..., queryFn: ... }); // sem <TData, TError> → error é unknown
```

---

## 11. Checklist de Validação

Antes de fazer code review ou entregar um hook de dados, verifique:

- [ ] O arquivo `use-<nome>.ts` está em `queries/` (GET) ou `mutations/` (POST/PUT/DELETE)?
- [ ] A query key usa a factory da feature (ex: `productKeys.detail(id)`)?
- [ ] `useQuery` e `useMutation` têm `<TData, ApiError>` explícitos nos generics?
- [ ] O `queryFn` / `mutationFn` desempacota o `Result<T, E>` com `if (result.isErr()) throw result.error`?
- [ ] A mutation invalida as queries corretas no `onSuccess`?
- [ ] Optimistic updates têm `onError` com rollback e `onSettled` com invalidação?
- [ ] Query keys que outras features precisam invalidar estão exportadas no `index.ts`?
- [ ] `useState` não está sendo usado para cachear dados da API?
