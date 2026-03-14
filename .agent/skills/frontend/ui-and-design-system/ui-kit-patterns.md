---
name: ui-kit-patterns
description: Criação e manutenção do UI Kit global em src/components/ usando Reactix como fonte de componentes headless: padrão de composição, tokens do theme/, dark mode com useColorScheme + ThemeContext, e documentação com Storybook. Alinhado com a arquitetura de features definida na skill arquiteto-react-native-features.
---

# UI Kit — Reactix, Tokens de Tema e Dark Mode

Você é um Engenheiro de Software Senior especialista em sistemas de design para React Native com Expo SDK 54. Sua responsabilidade é construir e manter o UI Kit global em `src/components/`, usando componentes da biblioteca **Reactix** como base headless, aplicando tokens do `theme/` de forma consistente e garantindo suporte a dark mode via `useColorScheme` e `ThemeContext`.

---

## Quando usar esta skill

- **Criar novo componente global:** Componente reutilizável em múltiplas features (`src/components/`).
- **Integrar componente Reactix:** Copiar, adaptar tokens e exportar um componente da biblioteca.
- **Dúvidas de tema:** Onde definir uma cor, espaçamento ou tipografia; como consumir tokens.
- **Dark mode:** Como um componente deve se adaptar ao tema claro/escuro.
- **Storybook:** Documentar variantes de um componente do UI Kit.

---

## Sobre Reactix

Reactix (pacote: `reacticx`) é uma biblioteca **headless e orientada a animações** para React Native. Ela **não impõe estilos** — fornece comportamento, gestos e animações prontos para produção, e você adiciona o visual usando os tokens do projeto.

### Características principais
- **Copy & Paste:** Sem instalação de pacote como dependência. O código do componente é copiado para o projeto via CLI.
- **Powered by:** React Native Reanimated 3, Gesture Handler, Skia, Expo Haptics, Expo Blur, Expo Symbols.
- **TypeScript-first:** Todos os componentes possuem tipagem completa.
- **+60 componentes:** atoms, molecules, micro-interactions, shaders, texts.

### Setup inicial

Instale as dependências nativas compartilhadas (uma única vez):

```bash
npx expo install react-native-reanimated react-native-gesture-handler \
  react-native-svg expo-haptics expo-blur expo-symbols \
  @expo/vector-icons @shopify/react-native-skia
```

Inicialize a configuração:

```bash
npx reacticx init
```

Isso cria `component.config.json` na raiz do projeto:

```json
{
  "outDir": "src/components"
}
```

> **Regra:** O `outDir` deve ser sempre `src/components` para respeitar a arquitetura da skill `arquiteto-react-native-features`.

---

## Estrutura de Diretórios

### `src/components/` — UI Kit Global

```text
src/
├── components/                         # UI KIT GLOBAL (componentes sem lógica de domínio)
│   ├── button/
│   │   ├── button.tsx                  # Componente principal
│   │   ├── button.types.ts             # Props e variantes tipadas
│   │   └── index.ts                    # Barrel público
│   │
│   ├── typography/
│   │   ├── text.tsx
│   │   ├── text.types.ts
│   │   └── index.ts
│   │
│   ├── card/                           # Integrado do Reactix
│   │   ├── card.tsx
│   │   ├── card.types.ts
│   │   └── index.ts
│   │
│   ├── loading-screen/                 # Usado pela skill navigation-patterns
│   │   ├── loading-screen.tsx
│   │   └── index.ts
│   │
│   └── icons/                          # Wrappers de ícones com token de cor
│       ├── home-icon.tsx
│       ├── user-icon.tsx
│       └── index.ts
│
├── theme/                              # DESIGN TOKENS
│   ├── colors.ts                       # Paleta e papéis semânticos (light + dark)
│   ├── spacing.ts                      # Escala de espaçamento
│   ├── typography.ts                   # Escala tipográfica
│   ├── radii.ts                        # Border-radius tokens
│   ├── context.tsx                     # ThemeProvider + ThemeContext
│   ├── use-theme.ts                    # Hook useTheme()
│   └── index.ts                        # Re-export de tudo
│
└── features/
    └── settings/
        └── store/
            └── theme.store.ts          # Override manual de tema (light/dark/system)
```

### Regra de Localização de Componentes

| Situação | Onde fica |
|---|---|
| Usado em 1 sub-feature (ex: `LoginForm`) | `features/auth/login/components/` |
| Usado em múltiplas sub-features de auth | `features/auth/components/` |
| Usado em múltiplas features diferentes | `src/components/` ✅ |
| Genérico de UI sem lógica de domínio | `src/components/` ✅ |

> **Regra (consistente com `arquiteto-react-native-features`):** Se dois domínios precisam de um componente, ele pertence ao UI Kit global em `src/components/`.

---

## 1. Design Tokens — `src/theme/`

### 1.1 Cores — `src/theme/colors.ts`

Define paleta bruta + papéis semânticos separados por modo:

```ts
// src/theme/colors.ts

/** Paleta bruta — nunca use diretamente nos componentes */
const palette = {
  // Brand
  brand50:  '#F0EAFF',
  brand100: '#D4BFFF',
  brand500: '#6200EE',
  brand600: '#5000C9',
  brand900: '#1A0050',

  // Neutros
  neutral0:   '#FFFFFF',
  neutral50:  '#F8F8F8',
  neutral100: '#F0F0F0',
  neutral300: '#CCCCCC',
  neutral500: '#888888',
  neutral700: '#444444',
  neutral900: '#111111',
  neutral950: '#060606',

  // Feedback
  success: '#00C851',
  error:   '#FF4444',
  warning: '#FF8800',
  info:    '#33B5E5',
} as const;

/** Papéis semânticos — USE SEMPRE ESTES nos componentes */
export const colors = {
  light: {
    // Superfícies
    background:        palette.neutral0,
    surface:           palette.neutral50,
    surfaceElevated:   palette.neutral100,

    // Conteúdo
    textPrimary:       palette.neutral900,
    textSecondary:     palette.neutral500,
    textDisabled:      palette.neutral300,
    textOnBrand:       palette.neutral0,

    // Brand
    brandPrimary:      palette.brand500,
    brandPrimaryHover: palette.brand600,

    // Bordas
    border:            palette.neutral100,
    borderStrong:      palette.neutral300,

    // Feedback
    success:  palette.success,
    error:    palette.error,
    warning:  palette.warning,
    info:     palette.info,
  },
  dark: {
    background:        palette.neutral950,
    surface:           '#111111',
    surfaceElevated:   '#1A1A1A',

    textPrimary:       palette.neutral0,
    textSecondary:     palette.neutral300,
    textDisabled:      palette.neutral500,
    textOnBrand:       palette.neutral0,

    brandPrimary:      palette.brand100,
    brandPrimaryHover: palette.brand50,

    border:            '#2A2A2A',
    borderStrong:      palette.neutral700,

    success:  palette.success,
    error:    palette.error,
    warning:  palette.warning,
    info:     palette.info,
  },
} as const;

export type ColorRole = keyof typeof colors.light;
export type ColorScheme = 'light' | 'dark';
```

### 1.2 Espaçamento — `src/theme/spacing.ts`

```ts
// src/theme/spacing.ts

/**
 * Escala de espaçamento baseada em múltiplos de 4px.
 * Use os tokens nomeados — não use números mágicos nos componentes.
 */
export const spacing = {
  xxs:  2,
  xs:   4,
  sm:   8,
  md:   12,
  lg:   16,
  xl:   24,
  xxl:  32,
  xxxl: 48,
  huge: 64,
} as const;

export type SpacingToken = keyof typeof spacing;
```

### 1.3 Tipografia — `src/theme/typography.ts`

```ts
// src/theme/typography.ts
import type { TextStyle } from 'react-native';

/**
 * Escala tipográfica com roles semânticos.
 * NUNCA use valores de fontSize ou fontWeight hardcoded nos componentes.
 */
export const typography = {
  // Títulos
  displayLarge:  { fontSize: 57, lineHeight: 64, fontWeight: '400' } as TextStyle,
  displayMedium: { fontSize: 45, lineHeight: 52, fontWeight: '400' } as TextStyle,
  headlineLarge: { fontSize: 32, lineHeight: 40, fontWeight: '600' } as TextStyle,
  headlineMedium:{ fontSize: 28, lineHeight: 36, fontWeight: '600' } as TextStyle,
  headlineSmall: { fontSize: 24, lineHeight: 32, fontWeight: '600' } as TextStyle,

  // Corpo
  titleLarge:  { fontSize: 22, lineHeight: 28, fontWeight: '500' } as TextStyle,
  titleMedium: { fontSize: 16, lineHeight: 24, fontWeight: '500' } as TextStyle,
  titleSmall:  { fontSize: 14, lineHeight: 20, fontWeight: '500' } as TextStyle,
  bodyLarge:   { fontSize: 16, lineHeight: 24, fontWeight: '400' } as TextStyle,
  bodyMedium:  { fontSize: 14, lineHeight: 20, fontWeight: '400' } as TextStyle,
  bodySmall:   { fontSize: 12, lineHeight: 16, fontWeight: '400' } as TextStyle,

  // Labels e Caption
  labelLarge:  { fontSize: 14, lineHeight: 20, fontWeight: '500' } as TextStyle,
  labelMedium: { fontSize: 12, lineHeight: 16, fontWeight: '500' } as TextStyle,
  labelSmall:  { fontSize: 11, lineHeight: 16, fontWeight: '500' } as TextStyle,
} as const;

export type TypographyToken = keyof typeof typography;
```

### 1.4 Border Radius — `src/theme/radii.ts`

```ts
// src/theme/radii.ts
export const radii = {
  none:   0,
  xs:     4,
  sm:     8,
  md:     12,
  lg:     16,
  xl:     24,
  xxl:    32,
  full:   9999,
} as const;

export type RadiusToken = keyof typeof radii;
```

### 1.5 Export Centralizado — `src/theme/index.ts`

```ts
// src/theme/index.ts
export { colors }     from './colors';
export type { ColorRole, ColorScheme } from './colors';
export { spacing }    from './spacing';
export type { SpacingToken } from './spacing';
export { typography } from './typography';
export type { TypographyToken } from './typography';
export { radii }      from './radii';
export type { RadiusToken } from './radii';
export { ThemeProvider, ThemeContext } from './context';
export { useTheme }   from './use-theme';
export type { Theme } from './context';
```

---

## 2. Dark Mode — `ThemeContext` e `useTheme`

### 2.1 Configuração do `app.json`

Para que o app siga automaticamente o tema do sistema operacional:

```json
// app.json
{
  "expo": {
    "userInterfaceStyle": "automatic",
    "android": {
      "userInterfaceStyle": "automatic"
    },
    "ios": {
      "userInterfaceStyle": "automatic"
    },
    "plugins": [
      "expo-system-ui"
    ]
  }
}
```

> **Expo SDK 54:** O plugin `expo-system-ui` é obrigatório no Android para que o `userInterfaceStyle` funcione corretamente em development builds.

### 2.2 `ThemeContext` — `src/theme/context.tsx`

```tsx
// src/theme/context.tsx
import React, { createContext, useContext, useMemo } from 'react';
import { useColorScheme } from 'react-native';
import { colors } from './colors';
import type { ColorScheme } from './colors';
import { spacing } from './spacing';
import { typography } from './typography';
import { radii } from './radii';

export type Theme = {
  colors: typeof colors.light;  // Sempre o tipo do light (ambos têm as mesmas chaves)
  spacing: typeof spacing;
  typography: typeof typography;
  radii: typeof radii;
  scheme: ColorScheme;
  isDark: boolean;
};

export const ThemeContext = createContext<Theme | null>(null);

type ThemeProviderProps = {
  children: React.ReactNode;
  /**
   * Override manual — use o valor da store settings/theme.store.ts.
   * Se undefined, segue o sistema operacional.
   */
  overrideScheme?: ColorScheme | null;
};

export function ThemeProvider({ children, overrideScheme }: ThemeProviderProps) {
  // useColorScheme retorna 'light' | 'dark' | null (null = preferência não definida)
  const systemScheme = useColorScheme();

  // Prioridade: override manual > sistema > fallback light
  const scheme: ColorScheme = overrideScheme ?? systemScheme ?? 'light';

  const theme = useMemo<Theme>(
    () => ({
      colors:     colors[scheme],
      spacing,
      typography,
      radii,
      scheme,
      isDark:     scheme === 'dark',
    }),
    [scheme],
  );

  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>;
}
```

### 2.3 Hook `useTheme` — `src/theme/use-theme.ts`

```ts
// src/theme/use-theme.ts
import { useContext } from 'react';
import { ThemeContext } from './context';
import type { Theme } from './context';

/**
 * Acessa o tema atual (cores, espaçamento, tipografia, radii, scheme).
 * DEVE ser chamado dentro de <ThemeProvider>.
 *
 * @example
 * const { colors, spacing } = useTheme();
 * style={{ backgroundColor: colors.background, padding: spacing.lg }}
 */
export function useTheme(): Theme {
  const theme = useContext(ThemeContext);
  if (!theme) {
    throw new Error('useTheme() deve ser usado dentro de <ThemeProvider>.');
  }
  return theme;
}
```

### 2.4 Override Manual de Tema — `src/features/settings/store/theme.store.ts`

O override de tema é uma preferência do usuário — state global que persiste no AsyncStorage. Segue a regra da skill `arquiteto-react-native-features`: store Zustand dentro da feature dona (`settings`).

```ts
// src/features/settings/store/theme.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';
import type { ColorScheme } from '@theme';

type ThemeState = {
  /** null = seguir sistema operacional */
  overrideScheme: ColorScheme | null;
};

type ThemeActions = {
  setScheme: (scheme: ColorScheme | null) => void;
};

export const useThemeStore = create<ThemeState & ThemeActions>()(
  persist(
    (set) => ({
      overrideScheme: null,
      setScheme: (scheme) => set({ overrideScheme: scheme }),
    }),
    {
      name: 'theme-preference',
      storage: createJSONStorage(() => AsyncStorage),
    },
  ),
);

// Hooks de conveniência exportados via barrel público da feature settings
export const useOverrideScheme = () => useThemeStore((s) => s.overrideScheme);
export const useThemeActions = () => useThemeStore((s) => ({ setScheme: s.setScheme }));
```

```ts
// src/features/settings/store/index.ts
export { useThemeStore, useOverrideScheme, useThemeActions } from './theme.store';
```

```ts
// src/features/settings/index.ts (barrel público da feature)
// Exporta apenas o contrato público
export { useOverrideScheme, useThemeActions } from './store';
```

### 2.5 Integrando `ThemeProvider` no Layout Raiz

```tsx
// src/app/_layout.tsx  ← Atualizar adicionando ThemeProvider
import { Stack } from 'expo-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@services/query-client';
import { ThemeProvider } from '@theme';
import { useOverrideScheme } from '@features/settings';

export default function RootLayout() {
  // Lê o override manual persistido pelo usuário (null = seguir sistema)
  const overrideScheme = useOverrideScheme();

  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider overrideScheme={overrideScheme}>
        <Stack screenOptions={{ headerShown: false }} />
      </ThemeProvider>
    </QueryClientProvider>
  );
}
```

> **Importante:** `ThemeProvider` envolve toda a árvore de componentes, mas fica **dentro** do `QueryClientProvider` — a ordem dos providers não conflita com a skill `navigation-patterns`.

---

## 3. Adicionando um Componente Reactix

### 3.1 Listar e Adicionar via CLI

```bash
# Ver todos os componentes disponíveis
npx reacticx list

# Filtrar por categoria
npx reacticx list -c molecules

# Adicionar componente ao projeto
npx reacticx add card

# Saída esperada:
# ✔ Added card!
# Files created:
#   src/components/card/index.tsx
#   src/components/card/types.ts
```

### 3.2 Adaptar Tokens ao Componente Copiado

Após adicionar o componente via CLI, **sempre substitua** cores e espaçamentos hardcoded pelos tokens do projeto:

```tsx
// src/components/card/card.tsx — ANTES (saído do Reactix, com hardcode)
const styles = StyleSheet.create({
  container: {
    backgroundColor: '#ffffff',  // ❌ hardcode
    padding: 16,                  // ❌ hardcode
    borderRadius: 12,             // ❌ hardcode
    borderWidth: 1,
    borderColor: '#e0e0e0',       // ❌ hardcode
  },
});
```

```tsx
// src/components/card/card.tsx — DEPOIS (com tokens)
import { useTheme } from '@theme';

export function Card({ children, style }: CardProps) {
  const { colors, spacing, radii } = useTheme(); // ✅ tokens

  return (
    <View
      style={[
        {
          backgroundColor: colors.surface,         // ✅ token
          padding:          spacing.lg,             // ✅ token
          borderRadius:     radii.md,               // ✅ token
          borderWidth:      1,
          borderColor:      colors.border,          // ✅ token
        },
        style,
      ]}
    >
      {children}
    </View>
  );
}
```

---

## 4. Anatomia de um Componente do UI Kit

Estrutura padrão para qualquer componente global:

### 4.1 Arquivo de Tipos — `button.types.ts`

```ts
// src/components/button/button.types.ts
import type { PressableProps } from 'react-native';

export type ButtonVariant = 'primary' | 'secondary' | 'ghost' | 'destructive';
export type ButtonSize    = 'sm' | 'md' | 'lg';

export interface ButtonProps extends Omit<PressableProps, 'style'> {
  /** Variante visual do botão */
  variant?: ButtonVariant;
  /** Tamanho do botão */
  size?: ButtonSize;
  /** Texto exibido no botão */
  label: string;
  /** Estado de carregamento */
  isLoading?: boolean;
  /** Estilo extra para o container */
  containerStyle?: object;
}
```

### 4.2 Componente Principal — `button.tsx`

```tsx
// src/components/button/button.tsx
import React from 'react';
import { Pressable, Text, ActivityIndicator, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';
import { useTheme } from '@theme';
import type { ButtonProps, ButtonVariant, ButtonSize } from './button.types';

const AnimatedPressable = Animated.createAnimatedComponent(Pressable);

export function Button({
  variant = 'primary',
  size = 'md',
  label,
  isLoading = false,
  disabled,
  onPress,
  containerStyle,
  accessibilityLabel,
  ...rest
}: ButtonProps) {
  const { colors, spacing, typography, radii } = useTheme();
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  const variantStyles: Record<ButtonVariant, object> = {
    primary: {
      backgroundColor: colors.brandPrimary,
    },
    secondary: {
      backgroundColor: colors.surface,
      borderWidth: 1,
      borderColor: colors.border,
    },
    ghost: {
      backgroundColor: 'transparent',
    },
    destructive: {
      backgroundColor: colors.error,
    },
  };

  const sizeStyles: Record<ButtonSize, { paddingVertical: number; paddingHorizontal: number }> = {
    sm: { paddingVertical: spacing.xs,  paddingHorizontal: spacing.md  },
    md: { paddingVertical: spacing.sm,  paddingHorizontal: spacing.lg  },
    lg: { paddingVertical: spacing.md,  paddingHorizontal: spacing.xl  },
  };

  const textColor = variant === 'primary' || variant === 'destructive'
    ? colors.textOnBrand
    : colors.textPrimary;

  return (
    <AnimatedPressable
      style={[
        styles.base,
        { borderRadius: radii.md },
        variantStyles[variant],
        sizeStyles[size],
        disabled || isLoading ? styles.disabled : null,
        animatedStyle,
        containerStyle,
      ]}
      disabled={disabled || isLoading}
      onPressIn={() => { scale.value = withSpring(0.96); }}
      onPressOut={() => { scale.value = withSpring(1); }}
      onPress={onPress}
      accessibilityRole="button"
      accessibilityLabel={accessibilityLabel ?? label}
      accessibilityState={{ disabled: disabled || isLoading }}
      {...rest}
    >
      {isLoading ? (
        <ActivityIndicator color={textColor} />
      ) : (
        <Text style={[typography.labelLarge, { color: textColor }]}>{label}</Text>
      )}
    </AnimatedPressable>
  );
}

const styles = StyleSheet.create({
  base: {
    alignItems: 'center',
    justifyContent: 'center',
    flexDirection: 'row',
  },
  disabled: {
    opacity: 0.5,
  },
});
```

### 4.3 Barrel Público — `index.ts`

```ts
// src/components/button/index.ts
export { Button } from './button';
export type { ButtonProps, ButtonVariant, ButtonSize } from './button.types';
```

### 4.4 Uso nos Componentes de Feature

```tsx
// src/features/auth/login/components/login-form.tsx
import { Button } from '@components/button';  // ✅ Via alias do barrel público

export function LoginForm({ onSubmit, isLoading }: LoginFormProps) {
  return (
    <Button
      label="Entrar"
      variant="primary"
      size="lg"
      isLoading={isLoading}
      onPress={onSubmit}
    />
  );
}
```

---

## 5. Componente `Text` Tipográfico

Um wrapper de `Text` que aplica tokens de tipografia e garante suporte a dark mode:

```tsx
// src/components/typography/text.tsx
import React from 'react';
import { Text as RNText } from 'react-native';
import { useTheme } from '@theme';
import type { TextProps } from './text.types';

export function Text({
  variant = 'bodyMedium',
  color,
  style,
  children,
  ...rest
}: TextProps) {
  const { colors, typography } = useTheme();

  return (
    <RNText
      style={[
        typography[variant],
        { color: color ?? colors.textPrimary },
        style,
      ]}
      {...rest}
    >
      {children}
    </RNText>
  );
}
```

```ts
// src/components/typography/text.types.ts
import type { TextProps as RNTextProps } from 'react-native';
import type { TypographyToken, ColorRole } from '@theme';

export interface TextProps extends Omit<RNTextProps, 'style'> {
  /** Token tipográfico (ex: 'headlineLarge', 'bodyMedium') */
  variant?: TypographyToken;
  /** Papel semântico de cor (ex: 'textSecondary', 'brandPrimary') */
  color?: string;
  style?: RNTextProps['style'];
}
```

---

## 6. Regras de Composição

### ✅ Correto

```tsx
// Usa tokens — responde ao dark mode automaticamente
const { colors, spacing } = useTheme();

<View style={{ backgroundColor: colors.background, padding: spacing.lg }}>
  <Text variant="headlineSmall">Título</Text>
</View>
```

### ❌ Errado

```tsx
// Hardcode — quebra no dark mode
<View style={{ backgroundColor: '#ffffff', padding: 16 }}>
  <Text style={{ fontSize: 24, color: '#111111' }}>Título</Text>
</View>
```

### Regras inegociáveis

| Regra | Motivo |
|---|---|
| **Nunca hardcode cores.** Use `colors.<role>` via `useTheme()`. | Dark mode e rebranding sem alterar componente |
| **Nunca hardcode espaçamento.** Use `spacing.<token>`. | Consistência visual global |
| **Nunca hardcode font size.** Use `typography.<token>`. | Escalabilidade de tipo |
| **Componentes globais são stateless.** Sem chamadas de API, sem stores Zustand diretamente. | Reusabilidade e testabilidade |
| **Sempre exportar via `index.ts`** | Encapsulamento e refatoração segura |
| **Acessibilidade obrigatória:** `accessibilityRole` + `accessibilityLabel` em interativos | Conformidade com WCAG 2.1 nível AA |

---

## 7. Storybook

### 7.1 Setup

```bash
npx expo install @storybook/react-native @storybook/core \
  @react-native-async-storage/async-storage
```

Estrutura de arquivos:

```text
src/
├── components/
│   └── button/
│       ├── button.tsx
│       ├── button.types.ts
│       ├── button.stories.tsx     # ← Storybook co-localizado
│       └── index.ts
│
└── .storybook/
    ├── main.ts
    └── preview.tsx                # Envolve com ThemeProvider
```

### 7.2 Preview com ThemeProvider — `.storybook/preview.tsx`

```tsx
// .storybook/preview.tsx
import React from 'react';
import { ThemeProvider } from '@theme';
import type { Preview } from '@storybook/react-native';

const preview: Preview = {
  decorators: [
    (Story) => (
      <ThemeProvider>
        <Story />
      </ThemeProvider>
    ),
  ],
  parameters: {
    controls: { matchers: { color: /(background|color)$/i } },
  },
};

export default preview;
```

### 7.3 Story de Componente — `button.stories.tsx`

```tsx
// src/components/button/button.stories.tsx
import React from 'react';
import { View } from 'react-native';
import type { Meta, StoryObj } from '@storybook/react-native';
import { Button } from './button';

const meta: Meta<typeof Button> = {
  title: 'UI Kit/Button',
  component: Button,
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'ghost', 'destructive'],
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg'],
    },
    isLoading: { control: 'boolean' },
    disabled:  { control: 'boolean' },
  },
  args: {
    label:     'Clique aqui',
    variant:   'primary',
    size:      'md',
    isLoading: false,
    disabled:  false,
  },
  decorators: [
    (Story) => (
      <View style={{ padding: 24, justifyContent: 'center', flex: 1 }}>
        <Story />
      </View>
    ),
  ],
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {};

export const Secondary: Story = {
  args: { variant: 'secondary' },
};

export const Ghost: Story = {
  args: { variant: 'ghost' },
};

export const Destructive: Story = {
  args: { variant: 'destructive' },
};

export const Loading: Story = {
  args: { isLoading: true },
};

export const Disabled: Story = {
  args: { disabled: true },
};

export const AllVariants: Story = {
  render: (args) => (
    <View style={{ gap: 12 }}>
      {(['primary', 'secondary', 'ghost', 'destructive'] as const).map((v) => (
        <Button key={v} {...args} variant={v} label={v} />
      ))}
    </View>
  ),
};
```

---

## 8. Fluxo de Implementação (Novo Componente do UI Kit)

Ao receber a solicitação de criar ou integrar um componente no UI Kit:

**1. Verificar disponibilidade no Reactix**
```bash
npx reacticx list | grep <nome>
```
Se disponível, adicione com `npx reacticx add <nome>`. Se não, crie do zero.

**2. Adaptar tokens**
Substituir toda cor, espaçamento, tipografia e border-radius hardcoded pelos tokens do `theme/`.

**3. Criar `<nome>.types.ts`**
Definir props com variantes tipadas. Extends de props nativas do RN quando aplicável.

**4. Criar `<nome>.tsx`**
Usar `useTheme()` para acessar tokens. Adicionar `accessibilityRole` e `accessibilityLabel` em elementos interativos.

**5. Criar barrel `index.ts`**
Exportar componente e tipos públicos.

**6. Criar story `<nome>.stories.tsx`**
Documentar variantes, estados de loading/disabled e casos extremos.

**7. Usar nos componentes de feature**
Import via alias `@components/<nome>`.

---

## Checklist de Validação

Antes de entregar qualquer componente do UI Kit, verifique:

### Tokens e Tema
- [ ] Nenhuma cor hardcoded? Todas as cores vêm de `colors.<role>` via `useTheme()`?
- [ ] Nenhum espaçamento hardcoded? Todos de `spacing.<token>`?
- [ ] Nenhum `fontSize` ou `fontWeight` hardcoded? Todos de `typography.<token>`?
- [ ] Nenhum `borderRadius` hardcoded? Todos de `radii.<token>`?

### Arquitetura
- [ ] Componente está em `src/components/` (não dentro de uma feature)?
- [ ] `index.ts` exporta apenas o contrato público (componente + tipos públicos)?
- [ ] Componente **não** importa stores Zustand nem faz chamadas de API diretamente?
- [ ] Import nos componentes de feature usa `@components/<nome>` (alias — não caminho relativo)?

### Acessibilidade
- [ ] Elementos interativos têm `accessibilityRole` correto?
- [ ] `accessibilityLabel` definido ou herdado via prop?
- [ ] Estado de `disabled` refletido em `accessibilityState`?

### Dark Mode
- [ ] Testado em modo escuro no simulador (`Settings > Developer > Dark Mode`)?
- [ ] `ThemeProvider` presente no `_layout.tsx` raiz (`src/app/_layout.tsx`)?

### Documentação
- [ ] Story `.stories.tsx` co-localizado no diretório do componente?
- [ ] Todas as variantes e estados (loading, disabled, error) documentados no Storybook?
