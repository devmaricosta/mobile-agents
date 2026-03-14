---
name: performance-optimization
description: Otimizações de performance para React Native com Expo SDK 54 — uso correto de memo, useCallback e useMemo considerando o React Compiler, lazy loading de features com React.lazy e Async Routes do Expo Router, otimização de listas com FlashList (v1 e v2 para New Architecture), e profiling com React Native DevTools e Chrome DevTools (Hermes).
---

# Performance Optimization — React Native com Expo (SDK 54+)

Você é um Engenheiro de Software Senior especialista em performance de aplicações React Native. Sua responsabilidade é identificar e eliminar gargalos de performance, garantindo renderizações eficientes, listas fluidas e startup rápido, seguindo as convenções de arquitetura definidas nas skills `arquiteto-react-native-features`, `zustand-architecture` e `react-query-patterns`.

---

## Quando usar esta skill

- **Re-renders desnecessários:** Componente re-renderiza mesmo sem mudança de dados relevantes.
- **Listas lentas ou travando:** `FlatList` com muitos itens causando jank ou frames perdidos.
- **Startup lento:** Bundle grande impactando o tempo de abertura do app.
- **Code review:** Validar se memoização, lazy loading e estrutura de listas estão corretos.
- **Profiling:** Saber qual ferramenta usar para identificar o gargalo específico.

---

## Contexto SDK 54 — Hermes, New Architecture e React 19

> **SDK 54 é o último a suportar a Legacy Architecture.** O SDK 55+ será exclusivamente New Architecture (Fabric + TurboModules + JSI). Toda otimização deve considerar compatibilidade com ambas, mas priorizar a New Architecture.

| Contexto | Impacto de Performance |
|---|---|
| **Hermes** | Engine padrão — AOT compilation, menor memoria, startup mais rápido |
| **New Architecture (Fabric)** | Renderização síncrona, sem bridge overhead, melhor integração nativa |
| **React 19.1** | React Compiler disponível — pode reduzir necessidade de memo manual |
| **XCFrameworks (iOS)** | Builds iOS até 10x mais rápidos (pré-compilado) |

---

## 1. Memoização — `memo`, `useCallback` e `useMemo`

### 1.1 Regra Fundamental — Profile Primeiro

> **Nunca aplique memoização sem medir.** `memo`, `useCallback` e `useMemo` adicionam overhead de comparação. O benefício só supera o custo quando o componente é caro de renderizar ou quando a instabilidade de referência causa re-renders desnecessários em filhos.

```
Fluxo correto:
1. Identifique o problema com o Profiler (seção 5)
2. Confirme que é um re-render desnecessário
3. Aplique memoização cirúrgica
4. Meça o impacto — compare antes e depois
```

### 1.2 Tabela de Decisão

| Situação | Ferramenta | Quando usar |
|---|---|---|
| Componente filho recebe props estáveis mas re-renderiza | `React.memo` | Props vêm de cima e raramente mudam |
| Função passada como prop a componente memoizado | `useCallback` | Garante referência estável entre renders |
| Cálculo custoso baseado em dados que raramente mudam | `useMemo` | Filtros, sorts, derivações de listas grandes |
| Seletor Zustand com múltiplos valores | `useShallow` (ver skill `zustand-architecture`) | Substituição do `useMemo` para stores |
| Simples acesso a 1 campo da store | Selector atômico direto | Sem overhead extra |

### 1.3 `React.memo` — Evitando Re-renders de Componentes Filhos

```tsx
// src/features/products/components/product-card.tsx

import React from 'react';
import { View, Pressable } from 'react-native';
import { Text } from '@components/typography';
import { useTheme } from '@theme';
import type { Product } from '../types/products.types';

type ProductCardProps = {
  product: Product;
  onToggleFavorite: (id: string) => void; // deve ser estável (useCallback no pai)
};

// ✅ memo com comparação padrão (shallow) — re-renderiza apenas quando props mudarem
export const ProductCard = React.memo(function ProductCard({
  product,
  onToggleFavorite,
}: ProductCardProps) {
  const { colors, spacing, radii } = useTheme();

  return (
    <View
      style={{
        backgroundColor: colors.surface,
        borderRadius: radii.md,
        padding: spacing.lg,
      }}
    >
      <Text variant="titleMedium">{product.name}</Text>
      <Pressable onPress={() => onToggleFavorite(product.id)}>
        <Text variant="labelSmall" color={colors.brandPrimary}>
          {product.isFavorite ? 'Remover favorito' : 'Favoritar'}
        </Text>
      </Pressable>
    </View>
  );
});

// ✅ memo com comparação customizada — útil quando props têm campos irrelevantes
export const ProductCardCustom = React.memo(
  function ProductCard({ product, onToggleFavorite }: ProductCardProps) {
    // ... implementação
  },
  (prev, next) =>
    // Só re-renderiza se o nome, preço ou isFavorite mudar — ignora outros campos
    prev.product.name === next.product.name &&
    prev.product.price === next.product.price &&
    prev.product.isFavorite === next.product.isFavorite &&
    prev.onToggleFavorite === next.onToggleFavorite,
);
```

### 1.4 `useCallback` — Estabilizando Referências de Funções

```tsx
// src/features/products/screens/products-screen.tsx

import React, { useCallback } from 'react';
import { useToggleFavorite } from '../mutations/use-toggle-favorite';
import { ProductCard } from '../components/product-card';

export function ProductsScreen() {
  const { mutate } = useToggleFavorite();

  // ✅ useCallback — garante que ProductCard (memoizado) não re-renderize por causa desta função
  const handleToggleFavorite = useCallback(
    (id: string) => {
      mutate({ id, isFavorite: true });
    },
    [mutate], // mutate do useMutation tem referência estável
  );

  // ❌ ERRADO — nova referência a cada render → ProductCard re-renderiza sempre
  // const handleToggleFavorite = (id: string) => mutate({ id, isFavorite: true });

  return (
    // ... renderiza ProductCard passando handleToggleFavorite
  );
}
```

> **Sobre o React Compiler (React 19):** O Expo SDK 54 usa React 19.1, que inclui o React Compiler (babel plugin). Quando **habilitado**, o compilador automatiza a memoização — reduzindo a necessidade de `memo`, `useCallback` e `useMemo` manuais. **Se o React Compiler estiver ativo no projeto, priorize código limpo sem memoização manual** e deixe o compilador otimizar. Verifique `babel.config.js` para confirmar.

### 1.5 `useMemo` — Cacheando Cálculos Custosos

```tsx
// src/features/products/hooks/use-filtered-products.ts

import { useMemo } from 'react';
import type { Product, ProductFilters } from '../types/products.types';

export function useFilteredProducts(products: Product[], filters: ProductFilters) {
  // ✅ useMemo — recalcula SOMENTE quando products ou filters mudarem
  // Sem useMemo: recalcula a cada render, mesmo sem mudança nos dados
  const filteredProducts = useMemo(() => {
    if (!products) return [];

    return products
      .filter((p) => {
        if (filters.category && p.category !== filters.category) return false;
        if (filters.minPrice && p.price < filters.minPrice) return false;
        if (filters.search) {
          return p.name.toLowerCase().includes(filters.search.toLowerCase());
        }
        return true;
      })
      .sort((a, b) => {
        if (filters.sortBy === 'price') return a.price - b.price;
        if (filters.sortBy === 'name') return a.name.localeCompare(b.name);
        return 0;
      });
  }, [products, filters]); // ← dependências precisas

  return filteredProducts;
}
```

```tsx
// Consumo em tela — filters precisa ser estável para que useMemo funcione
function ProductsScreen() {
  const { data: products } = useProducts();

  // ✅ useMemo no objeto de filtros — evita criar novo objeto a cada render
  const filters = useMemo<ProductFilters>(
    () => ({ category: selectedCategory, sortBy: 'price' }),
    [selectedCategory],
  );

  const filteredProducts = useFilteredProducts(products ?? [], filters);
  // ...
}
```

### 1.6 Anti-Padrões de Memoização

```tsx
// ❌ useMemo desnecessário — operação barata
const total = useMemo(() => price + tax, [price, tax]); // custo > benefício

// ✅ CORRETO — apenas compute diretamente
const total = price + tax;

// ❌ useCallback sem componente memoizado consumindo — overhead inútil
const handlePress = useCallback(() => setCount(c => c + 1), []); // sem memo no filho

// ❌ memo em componente que sempre recebe novas referências — memo nunca é ativado
<MemoizedComponent data={{ id: 1 }} /> // objeto literal → nova referência a cada render
```

---

## 2. Lazy Loading de Features — `React.lazy` e Async Routes

### 2.1 Estratégia de Lazy Loading

O objetivo é **não aumentar o tempo de startup** ao adicionar novas telas. Toda tela que não é exibida na abertura do app deve ser carregada sob demanda.

| Abordagem | Quando usar | Nível de maturidade |
|---|---|---|
| **Async Routes do Expo Router** | Projeto usa Expo Router (recomendado) | Alpha — cuidado em produção |
| **`React.lazy` + `Suspense`** | Lazy loading manual de telas específicas | Estável, produção |
| **Carregamento implícito do Expo Router** | Expo Router carrega rotas sob demanda automaticamente | Estável, produção |

### 2.2 Carregamento Implícito — Comportamento Padrão do Expo Router

> **Expo Router com SDK 54 já faz lazy loading implícito das rotas.** Cada arquivo em `src/app/` é carregado apenas quando navegado pela primeira vez. **Isso cobre a maioria dos casos sem código adicional.**

```tsx
// src/app/(home)/index.tsx — só é carregado quando o usuário navega para esta rota
export { default } from '@/features/home/screens/home-screen';

// src/app/(settings)/settings.tsx — idem, carregado on-demand
export { default } from '@/features/settings/screens/settings-screen';
```

### 2.3 `React.lazy` + `Suspense` — Lazy Loading Explícito

Use quando quiser controle granular sobre o que é carregado sob demanda dentro de uma feature:

```tsx
// src/features/reports/screens/reports-screen.tsx
import React, { Suspense, lazy } from 'react';
import { View, ActivityIndicator } from 'react-native';
import { useTheme } from '@theme';

// ✅ Carrega o componente pesado apenas quando ReportsScreen for renderizada
const HeavyChartComponent = lazy(
  () => import('../components/heavy-chart'),
  // Nota: o arquivo dynamic import não deve ter side effects de módulo
);

const LazySkeletonFallback = () => {
  const { colors } = useTheme();
  return (
    <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
      <ActivityIndicator color={colors.brandPrimary} />
    </View>
  );
};

export function ReportsScreen() {
  return (
    <View style={{ flex: 1 }}>
      {/* Suspense: exibe fallback enquanto HeavyChartComponent carrega */}
      <Suspense fallback={<LazySkeletonFallback />}>
        <HeavyChartComponent />
      </Suspense>
    </View>
  );
}
```

```tsx
// ✅ Lazy loading de modal ou bottomsheet pesado em uma feature
// src/features/checkout/screens/checkout-screen.tsx
import React, { Suspense, lazy, useState } from 'react';

const AddressPickerModal = lazy(() => import('../components/address-picker-modal'));

export function CheckoutScreen() {
  const [isAddressModalOpen, setAddressModalOpen] = useState(false);

  return (
    <View style={{ flex: 1 }}>
      {/* Modal só é carregado quando aberto — não impacta startup */}
      {isAddressModalOpen && (
        <Suspense fallback={null}>
          <AddressPickerModal onClose={() => setAddressModalOpen(false)} />
        </Suspense>
      )}
    </View>
  );
}
```

### 2.4 Async Routes (Expo Router — Alpha)

Para projetos que aceitam features em alpha, o Expo Router suporta bundle splitting automático via Async Routes:

```json
// app.json — habilitar async routes (ALPHA — avaliar estabilidade antes de usar em produção)
{
  "expo": {
    "experiments": {
      "asyncRoutes": true
    }
  }
}
```

```tsx
// Com asyncRoutes habilitado, o Expo Router envolve cada rota em Suspense automaticamente
// Você pode definir um fallback global no layout raiz:

// src/app/_layout.tsx
import { Stack } from 'expo-router';
import { Suspense } from 'react';
import { LoadingScreen } from '@components/loading-screen';

export default function RootLayout() {
  return (
    // Suspense global para todas as rotas assíncronas
    <Suspense fallback={<LoadingScreen />}>
      <Stack screenOptions={{ headerShown: false }} />
    </Suspense>
  );
}
```

> **Atenção:** `asyncRoutes` está em alpha no SDK 54. Use `React.lazy` manual para ambientes de produção até a feature ser estabilizada oficialmente.

---

## 3. Otimização de Listas — FlashList

### 3.1 Por que FlashList em vez de FlatList?

| Métrica | FlatList | FlashList |
|---|---|---|
| UI Thread FPS | Baseline | Até 5× mais rápido |
| JS Thread FPS | Baseline | Até 10× mais rápido |
| Memória | Recria views ao rolar | Recicla views (pool de componentes) |
| Compatibilidade | Legacy + New Architecture | New Architecture (v2 nativo) |

> **Regra:** Use `FlashList` para **qualquer lista com mais de 50 itens** ou listas com renderização complexa por item. Para listas simples com poucos itens estáticos, `FlatList` ou até `ScrollView` com `map` podem ser mais simples.

### 3.2 Instalação

```bash
# FlashList — use npx expo install para garantir compatibilidade com SDK 54
npx expo install @shopify/flash-list
```

> **FlashList v2 (New Architecture):** Para projetos com New Architecture habilitada (obrigatório no SDK 55+), prefira usar a versão mais recente do FlashList que suporta Fabric nativamente. No SDK 54, verifique se a versão instalada é compatível com a arquitetura em uso do projeto.

### 3.3 Uso Básico

```tsx
// src/features/products/components/product-list.tsx
import React, { useCallback } from 'react';
import { FlashList } from '@shopify/flash-list';
import type { Product } from '../types/products.types';
import { ProductCard } from './product-card';

type ProductListProps = {
  products: Product[];
  onToggleFavorite: (id: string) => void;
};

export function ProductList({ products, onToggleFavorite }: ProductListProps) {
  // ✅ renderItem memoizado — evita que FlashList recrie o callback a cada render
  const renderItem = useCallback(
    ({ item }: { item: Product }) => (
      <ProductCard product={item} onToggleFavorite={onToggleFavorite} />
    ),
    [onToggleFavorite],
  );

  const keyExtractor = useCallback((item: Product) => item.id, []);

  return (
    <FlashList
      data={products}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      // estimatedItemSize: MEDia do height de um item em pixels.
      // FlashList usa para calcular layout e alocar o pool de reciclagem.
      // Valor impreciso = warnings no dev; meça com o Profiler (seção 5).
      estimatedItemSize={120}
      // removeClippedSubviews: detach off-screen views do native hierarchy
      // Reduz memória, especialmente no Android — padrão true no FlashList
      removeClippedSubviews
      // showsVerticalScrollIndicator=false: remove scrollbar nativo (estético)
      showsVerticalScrollIndicator={false}
    />
  );
}
```

### 3.4 Listas com Itens de Tipos Diferentes (`getItemType`)

```tsx
// src/features/feed/components/activity-feed.tsx
import React, { useCallback } from 'react';
import { FlashList } from '@shopify/flash-list';
import type { FeedItem } from '../types/feed.types';
import { PostCard } from './post-card';
import { AdCard } from './ad-card';
import { StoryRow } from './story-row';

// ✅ getItemType: o FlashList cria um pool de reciclagem POR TIPO
// Sem isso, um PostCard pode ser reciclado para renderizar um AdCard → glitch visual
export function ActivityFeed({ items }: { items: FeedItem[] }) {
  const getItemType = useCallback(
    (item: FeedItem) => item.type, // 'post' | 'ad' | 'story'
    [],
  );

  const renderItem = useCallback(({ item }: { item: FeedItem }) => {
    switch (item.type) {
      case 'post':  return <PostCard post={item} />;
      case 'ad':    return <AdCard ad={item} />;
      case 'story': return <StoryRow story={item} />;
    }
  }, []);

  return (
    <FlashList
      data={items}
      renderItem={renderItem}
      getItemType={getItemType}
      estimatedItemSize={240}
    />
  );
}
```

### 3.5 FlashList com Paginação (Integração com React Query)

```tsx
// src/features/products/components/paginated-product-list.tsx
import React, { useCallback } from 'react';
import { FlashList } from '@shopify/flash-list';
import { ActivityIndicator, View } from 'react-native';
import { useTheme } from '@theme';
import type { Product } from '../types/products.types';

type PaginatedListProps = {
  products: Product[];
  isLoading: boolean;
  isFetchingNextPage: boolean;
  hasNextPage: boolean;
  onEndReached: () => void;
};

export function PaginatedProductList({
  products,
  isLoading,
  isFetchingNextPage,
  hasNextPage,
  onEndReached,
}: PaginatedListProps) {
  const { colors } = useTheme();

  const renderFooter = useCallback(() => {
    if (!isFetchingNextPage) return null;
    return (
      <View style={{ paddingVertical: 16 }}>
        <ActivityIndicator color={colors.brandPrimary} />
      </View>
    );
  }, [isFetchingNextPage, colors.brandPrimary]);

  if (isLoading) return <ActivityIndicator color={colors.brandPrimary} />;

  return (
    <FlashList
      data={products}
      renderItem={({ item }) => <ProductCard product={item} />}
      estimatedItemSize={120}
      ListFooterComponent={renderFooter}
      // onEndReachedThreshold: trigger quando 20% do fim da lista é atingido
      onEndReachedThreshold={0.2}
      onEndReached={hasNextPage ? onEndReached : undefined}
    />
  );
}
```

### 3.6 Anti-Padrões de Lista

```tsx
// ❌ ERRADO — renderItem inline cria nova função a cada render → itens re-renderizam
<FlashList
  data={products}
  renderItem={({ item }) => <ProductCard product={item} />} // instável
/>

// ✅ CORRETO — renderItem fora do render ou com useCallback
const renderItem = useCallback(({ item }) => <ProductCard product={item} />, []);
<FlashList data={products} renderItem={renderItem} />

// ❌ ERRADO — estimatedItemSize muito impreciso gera layout thrashing
estimatedItemSize={10} // itens reais têm 200px

// ❌ ERRADO — sem keyExtractor em listas que mudam → inconsistências de reciclagem
<FlashList data={products} renderItem={renderItem} /> // sem keyExtractor

// ❌ ERRADO — ScrollView com map para listas grandes
<ScrollView>
  {products.map(p => <ProductCard key={p.id} product={p} />)}
</ScrollView>
// → cria TODOS os componentes na memória — use FlashList
```

---

## 4. Outras Otimizações de Performance

### 4.1 Evitar Criação de Objetos Inline em Estilos

```tsx
// ❌ ERRADO — objeto inline no style → nova referência a cada render
// React Native compara referências de estilo, não valores
<View style={{ backgroundColor: colors.background, padding: 16 }} />

// ✅ CORRETO — StyleSheet.create cacheado (ou objeto fora do componente)
import { StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: { padding: 16 }, // valores fixos
});

// Para estilos dinâmicos (que dependem de tema), inline é inevitável,
// mas os tokens do useTheme() são valores primitivos (string/number) — OK.
<View style={[styles.container, { backgroundColor: colors.background }]} />
```

### 4.2 `InteractionManager` — Diferir Trabalho Pesado

```tsx
// src/features/analytics/hooks/use-screen-analytics.ts
import { useEffect } from 'react';
import { InteractionManager } from 'react-native';
import { analytics } from '@services/analytics';

/**
 * Registra evento de analytics APÓS animações de navegação completarem.
 * Evita janking durante transições de tela.
 */
export function useScreenAnalytics(screenName: string) {
  useEffect(() => {
    // ✅ Aguarda animações finalizarem antes de executar trabalho pesado
    const task = InteractionManager.runAfterInteractions(() => {
      analytics.track('screen_view', { screen: screenName });
    });

    return () => task.cancel();
  }, [screenName]);
}
```

### 4.3 Imagens

```bash
# Instale expo-image para caching avançado e placeholder com blurhash
npx expo install expo-image
```

```tsx
// ✅ expo-image: caching automático, placeholder, priority loading
import { Image } from 'expo-image';

<Image
  source={{ uri: product.imageUrl }}
  style={{ width: 200, height: 200 }}
  // Sempre especifique width/height para evitar layout shifts
  contentFit="cover"
  // placeholder: exibido enquanto imagem carrega (blurhash ou cor)
  placeholder={{ blurhash: product.blurhash }}
  // priority: 'high' para imagens above-the-fold
  priority="normal"
  // transition: animação suave ao carregar
  transition={200}
/>
```

### 4.4 Removendo `console.log` em Produção

```js
// babel.config.js — remove console.* em builds de produção
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [
      // ✅ Remove todos os console.* em produção — reduz bundle e acelera JS thread
      ...(process.env.NODE_ENV === 'production'
        ? [['transform-remove-console', { exclude: ['error', 'warn'] }]]
        : []),
    ],
  };
};
```

```bash
# Instale o plugin
npm install --save-dev babel-plugin-transform-remove-console
```

---

## 5. Profiling — Identificando Gargalos

### 5.1 Ferramentas e quando usar cada uma

| Ferramenta | Melhor para | Como acessar |
|---|---|---|
| **React Native DevTools** | Re-renders desnecessários, commit timings | `j` no terminal Metro → `Open DevTools` |
| **Profiler (React DevTools)** | Qual componente é mais lento, quantas vezes renderiza | Aba Profiler no React Native DevTools |
| **Chrome DevTools (Hermes)** | CPU profiling da JavaScript thread, flamegraph | `chrome://inspect` com app rodando |
| **Android Studio Profiler** | Memória, CPU nativo, network no Android | Android Studio → App Running |
| **Xcode Instruments** | Memory leaks, CPU, frame rate no iOS | Xcode → Product → Profile |

> **Regra:** Sempre faça profiling em **modo release** (não debug). O modo debug é 5–10× mais lento e pode mascarar ou exagerar problemas reais.

```bash
# Build release para Android (profiling real)
npx expo run:android --variant release

# Build release para iOS (profiling real)
npx expo run:ios --configuration Release
```

### 5.2 React Native DevTools — Detectando Re-renders

```
1. Abra o app no simulador/dispositivo
2. No terminal do Metro, pressione 'j' → clique em "Open DevTools"
   (ou use a aba "Open JS Debugger" no dev menu: shake o dispositivo)
3. Vá para a aba "Profiler"
4. Ative "Highlight updates when components render" (borboleta azul)
5. Interaja com o app — componentes re-renderizando piscam em azul/verde/vermelho
6. Grave uma sessão com "Record" e analise o flamegraph de commits
```

### 5.3 React DevTools Profiler — Interpretando o Flamegraph

```
Leitura do flamegraph:
- Cada barra = 1 componente
- Largura = tempo de renderização (wider = mais lento)
- Cor: cinza = não renderizou, colorido = renderizou
  - Verde claro = render rápido
  - Amarelo/Laranja = render moderado
  - Vermelho = render lento (candidato a otimização)

Fluxo de investigação:
1. Clique em um commit (barra superior) onde o app estava lento
2. Identifique o componente vermelho/laranja mais na raiz da árvore
3. Verifique "Why did this render?" — lista os props/state que mudaram
4. Aplique memo/useCallback/useMemo para estabilizar as mudanças identificadas
```

### 5.4 Chrome DevTools (Hermes) — CPU Profiling

```
Para identificar gargalos na JS thread (computação pesada, parsings lentos):

1. Abra chrome://inspect no Chrome
2. Clique em "inspect" sob o seu app React Native
3. Vá para a aba "Profiler"
4. Clique em "Start" → reproduza o problema → "Stop"
5. Analise o flamegraph: eixo X = tempo, eixo Y = pilha de chamadas
6. Funções largas no topo da pilha = candidatas a otimização

Cenários comuns:
- Filtros/sorts em arrays grandes no render → mover para useMemo
- Parsings JSON pesados na thread JS → mover para worker ou nativo
- Animações causando frames perdidos → verificar se estão no UI thread (Reanimated)
```

### 5.5 Checklist de Profiling

```
Antes de reportar um problema de performance:
□ O problema ocorre em modo release? (não apenas debug)
□ O flamegraph aponta para qual componente/função específica?
□ "Why did this render?" mostra qual prop/state é instável?
□ O problema está no JS thread ou no UI thread?
□ O problema é de re-render ou de lentidão na lógica de negócio?
```

---

## 6. Integração com a Arquitetura do Projeto

### 6.1 Performance e TanStack Query

A skill `react-query-patterns` já define defaults de performance no `QueryClient`:

```ts
// src/services/query-client.ts — já configurado na skill react-query-patterns
{
  staleTime: 2 * 60 * 1000,     // dados frescos por 2 min → evita refetches
  gcTime: 5 * 60 * 1000,        // cache por 5 min após desmontar
  refetchOnWindowFocus: false,  // irrelevante em mobile — evita refetches
  refetchOnReconnect: true,     // refetch ao reconectar (mobile importante)
}
```

> **Não altere esses defaults sem medir.** Reducão de `staleTime` aumenta refetches desnecessários; aumento excessivo pode exibir dados desatualizados.

### 6.2 Performance e Zustand

A skill `zustand-architecture` define seletores granulares. A regra crítica:

```ts
// ❌ ERRADO — subscreve toda a store → re-render em qualquer mudança
const store = useAuthStore();

// ✅ CORRETO — selector atômico → re-render APENAS quando user mudar
const user = useAuthStore((s) => s.user);

// ✅ CORRETO — useShallow para múltiplos valores
const { user, isAuthenticated } = useAuthStore(
  useShallow((s) => ({ user: s.user, isAuthenticated: s.isAuthenticated })),
);
```

> Ver skill `zustand-architecture` seção 7 para a tabela completa de decisão de seletores.

### 6.3 Performance e FlashList + TanStack Query

```tsx
// src/features/products/screens/products-screen.tsx
import React, { useCallback, useMemo } from 'react';
import { useInfiniteQuery } from '@tanstack/react-query';
import { FlashList } from '@shopify/flash-list';
import { productKeys } from '../queries/query-keys';
import { getProducts } from '../services/products.service';
import { PaginatedProductList } from '../components/paginated-product-list';

export function ProductsScreen() {
  const {
    data,
    isLoading,
    isFetchingNextPage,
    hasNextPage,
    fetchNextPage,
  } = useInfiniteQuery({
    queryKey: productKeys.lists(),
    queryFn: async ({ pageParam = 1 }) => {
      const result = await getProducts({ page: pageParam });
      if (result.isErr()) throw result.error;
      return result.value;
    },
    getNextPageParam: (lastPage) => lastPage.nextPage,
    initialPageParam: 1,
  });

  // ✅ useMemo para achatar as páginas em uma lista plana — recalcula só quando data mudar
  const products = useMemo(
    () => data?.pages.flatMap((page) => page.items) ?? [],
    [data],
  );

  // ✅ useCallback estável para passar ao FlashList
  const handleEndReached = useCallback(() => {
    if (hasNextPage && !isFetchingNextPage) fetchNextPage();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <PaginatedProductList
      products={products}
      isLoading={isLoading}
      isFetchingNextPage={isFetchingNextPage}
      hasNextPage={!!hasNextPage}
      onEndReached={handleEndReached}
    />
  );
}
```

---

## 7. Checklist de Validação de Performance

Antes de entregar qualquer tela ou lista, verifique:

- [ ] Listas com **mais de 50 itens** usam `FlashList` (não `FlatList` ou `ScrollView.map`)?
- [ ] `estimatedItemSize` do FlashList está razoavelmente próximo do tamanho real?
- [ ] `getItemType` está definido para listas com itens de **tipos diferentes**?
- [ ] `renderItem` é memoizado com `useCallback` (ou definido fora do componente)?
- [ ] Componentes filhos que recebem callbacks estáveis estão envolvidos em `React.memo`?
- [ ] `useCallback` e `useMemo` têm **dependências precisas** (sem deps faltando ou extras)?
- [ ] `useMemo` está sendo usado apenas para cálculos **genuinamente custosos** (array sort/filter de listas, etc.)?
- [ ] Seletores Zustand usam **selectors atômicos** ou `useShallow` (verificar skill `zustand-architecture`)?
- [ ] Imagens têm `width` e `height` explícitos e usam `expo-image`?
- [ ] `console.log` está sendo removido em produção via `babel-plugin-transform-remove-console`?
- [ ] Trabalho pesado pós-navegação usa `InteractionManager.runAfterInteractions`?
- [ ] O problema foi verificado em **modo release** antes de otimizar?
- [ ] O problema foi identificado com o **Profiler** (não por intuição)?
