---
name: forms-pattern
description: Padrão de formulários com React Hook Form + Zod para React Native com Expo SDK 54 — onde fica o schema de validação, como integrar com os hooks da feature, componentes de campo controlados e reutilizáveis via useController, e tratamento de erros de validação na UI com tokens do theme/. Alinhado com a arquitetura de features, ui-kit-patterns, react-query-patterns e error-handling.
---

# Padrões de Formulários — React Hook Form + Zod

Você é um Engenheiro de Software Senior especialista em formulários para React Native com Expo SDK 54. Sua responsabilidade é garantir que **todo formulário** da aplicação use React Hook Form com validação Zod, componentes de campo controlados via `Controller` ou `useController`, e exibição de erros consistente com os tokens do `theme/` e o padrão de erros da skill `error-handling`.

---

## Quando usar esta skill

- **Novo formulário:** Login, cadastro, edição de perfil, busca com filtros, checkout, etc.
- **Componente de campo:** Criar ou adaptar um campo `TextInput`, `Select`, `Checkbox`, etc. para ser controlado pelo React Hook Form.
- **Validação:** Definir regras de validação, mensagens de erro e validação cross-field.
- **Integração com API:** Conectar o `handleSubmit` do formulário a um `useMutation` do TanStack Query.
- **Code review:** Validar se o formulário segue os padrões do projeto.

---

## 1. Instalação

```bash
npx expo install react-hook-form @hookform/resolvers zod
```

> **SDK 54:** React Hook Form v7+ é totalmente compatível com React 19.1 (usado no SDK 54) e com a New Architecture do React Native.

---

## 2. Localização dos Arquivos

A estrutura segue rigorosamente a regra de co-localização da skill `arquiteto-react-native-features`:

```text
src/
├── features/
│   └── auth/
│       └── login/
│           ├── schemas/
│           │   └── login.schema.ts          ← Schema Zod + tipo inferido
│           ├── components/
│           │   └── login-form.tsx           ← Componente de UI do formulário (recebe props)
│           ├── hooks/
│           │   └── use-login-form.ts        ← Hook que orquestra RHF + useMutation
│           ├── screens/
│           │   └── login-screen.tsx         ← Monta: hook + formulário
│           └── index.ts                     ← Barrel público
│
└── components/
    └── form/                                ← CAMPOS REUTILIZÁVEIS DO UI KIT
        ├── form-field.tsx                   ← Campo controlado genérico (useController)
        ├── form-field.types.ts
        ├── form-error.tsx                   ← Texto de erro inline
        ├── form-label.tsx                   ← Label com indicador opcional/obrigatório
        └── index.ts
```

### Regra de localização

| Arquivo | Onde fica |
|---|---|
| Schema Zod do formulário | `features/<feature>/<sub>/schemas/<name>.schema.ts` |
| Hook do formulário (`useForm` + `useMutation`) | `features/<feature>/<sub>/hooks/use-<name>-form.ts` |
| Componente de UI do formulário | `features/<feature>/<sub>/components/<name>-form.tsx` |
| Campos genéricos reutilizáveis (`FormField`, `FormError`) | `src/components/form/` |

---

## 3. Schema Zod — `schemas/`

### 3.1 Padrão de Schema

O schema Zod é a **única fonte da verdade** para tipos e validação do formulário. O tipo TypeScript é **inferido** do schema — não defina o tipo manualmente em paralelo.

```ts
// src/features/auth/login/schemas/login.schema.ts

import { z } from 'zod';

export const loginSchema = z.object({
  email: z
    .string({ required_error: 'E-mail é obrigatório.' })
    .email('Informe um e-mail válido.'),
  password: z
    .string({ required_error: 'Senha é obrigatória.' })
    .min(8, 'A senha deve ter no mínimo 8 caracteres.'),
});

// ✅ Tipo inferido do schema — nunca defina LoginFormData manualmente em paralelo
export type LoginFormData = z.infer<typeof loginSchema>;
```

### 3.2 Schemas Complexos

```ts
// src/features/auth/register/schemas/register.schema.ts
import { z } from 'zod';

export const registerSchema = z
  .object({
    name: z
      .string({ required_error: 'Nome é obrigatório.' })
      .min(3, 'Nome deve ter no mínimo 3 caracteres.'),
    email: z
      .string({ required_error: 'E-mail é obrigatório.' })
      .email('Informe um e-mail válido.'),
    phone: z
      .string()
      .regex(/^\(\d{2}\)\s\d{4,5}-\d{4}$/, 'Telefone inválido. Ex: (11) 91234-5678')
      .optional()
      .or(z.literal('')), // Campo opcional — string vazia é válida
    password: z
      .string({ required_error: 'Senha é obrigatória.' })
      .min(8, 'Mínimo de 8 caracteres.')
      .regex(/[A-Z]/, 'Deve conter ao menos uma letra maiúscula.')
      .regex(/[0-9]/, 'Deve conter ao menos um número.'),
    confirmPassword: z.string({ required_error: 'Confirme sua senha.' }),
  })
  // Validação cross-field com refine
  .refine((data) => data.password === data.confirmPassword, {
    message: 'As senhas não coincidem.',
    path: ['confirmPassword'], // O erro aparece no campo confirmPassword
  });

export type RegisterFormData = z.infer<typeof registerSchema>;
```

### 3.3 Regras para Schemas

| Regra | Detalhe |
|---|---|
| **Co-localização** | Schema reside em `schemas/` dentro da sub-feature que o usa |
| **Tipo inferido** | Sempre `z.infer<typeof schema>` — nunca duplicar o tipo |
| **Mensagens em PT-BR** | Todas as mensagens de validação devem ser em português |
| **`required_error`** | Use o parâmetro para mensagens de campo vazio (sem `min(1, ...)` para strings obrigatórias) |
| **Cross-field** | Use `schema.refine()` e sempre especifique `path: ['campo']` |
| **Campos opcionais** | Use `.optional().or(z.literal(''))` para strings opcionais em TextInput |

---

## 4. Hook do Formulário — `hooks/use-<name>-form.ts`

O hook do formulário **orquestra** o React Hook Form e o `useMutation` do TanStack Query. Ele é responsável por:
1. Configurar o `useForm` com o schema e defaults.
2. Integrar o `handleSubmit` com o `mutate`.
3. Retornar apenas o que o componente de UI precisa.

```ts
// src/features/auth/login/hooks/use-login-form.ts
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useRouter } from 'expo-router';

import { loginSchema, type LoginFormData } from '../schemas/login.schema';
import { useLogin } from '../mutations/use-login'; // ← useMutation da skill react-query-patterns
import { useAuthActions } from '@features/auth';   // ← Store via barrel público
import { ROUTES } from '@constants/routes';

export function useLoginForm() {
  const router = useRouter();
  const { setAuth } = useAuthActions();

  // 1️⃣ Configuração do React Hook Form com zodResolver
  const form = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
    },
    // 'onBlur': valida ao sair do campo → feedback imediato sem ser intrusivo
    // 'onChange': valida a cada tecla → mais reativo, pode ser agressivo
    // 'onSubmit': valida apenas ao submeter → menos feedback, mais simples
    mode: 'onBlur',
  });

  // 2️⃣ Mutation do TanStack Query (seguindo react-query-patterns)
  const loginMutation = useLogin();

  // 3️⃣ handleSubmit integrado com a mutation
  const onSubmit = form.handleSubmit((data: LoginFormData) => {
    loginMutation.mutate(data, {
      onSuccess: ({ user, accessToken, refreshToken }) => {
        setAuth(user, accessToken, refreshToken);
        router.replace(ROUTES.HOME);
      },
    });
  });

  return {
    // Expõe control + formState para o componente de UI
    control: form.control,
    formState: form.formState,
    onSubmit,
    // Estado da mutation — para loading e erros de servidor
    isPending: loginMutation.isPending,
    serverError: loginMutation.error, // ApiError tipado (skill error-handling)
    reset: form.reset,
  };
}
```

> **Separação de responsabilidades:**
> - `use-login-form.ts` → orquestra RHF + TanStack Query.
> - `login-form.tsx` → UI pura: recebe `control` e `onSubmit` como props, sem lógica de negócio.
>
> **Não use `useState` dentro do hook do formulário** para estado de campo — o React Hook Form gerencia isso internamente.

---

## 5. Componentes de Campo Controlados — `src/components/form/`

### 5.1 `FormField` — Campo Genérico Reutilizável

Use `useController` para criar campos reutilizáveis no UI Kit. O `useController` é a alternativa hook-based ao `<Controller>`, indicada para componentes extraídos (com seus próprios arquivos).

```ts
// src/components/form/form-field.types.ts
import type { UseControllerProps, FieldValues, Path } from 'react-hook-form';
import type { TextInputProps } from 'react-native';

export type FormFieldProps<TForm extends FieldValues> = UseControllerProps<TForm> & {
  label: string;
  placeholder?: string;
  secureTextEntry?: boolean;
  autoCapitalize?: TextInputProps['autoCapitalize'];
  keyboardType?: TextInputProps['keyboardType'];
};
```

```tsx
// src/components/form/form-field.tsx
import React from 'react';
import { View, TextInput, StyleSheet } from 'react-native';
import { useController, type FieldValues } from 'react-hook-form';
import { useTheme } from '@theme';
import { FormLabel } from './form-label';
import { FormError } from './form-error';
import type { FormFieldProps } from './form-field.types';

export function FormField<TForm extends FieldValues>({
  label,
  placeholder,
  secureTextEntry = false,
  autoCapitalize = 'none',
  keyboardType = 'default',
  // Props do useController (name, control, rules, defaultValue)
  ...controllerProps
}: FormFieldProps<TForm>) {
  const { colors, spacing, typography, radii } = useTheme(); // ✅ Tokens do theme/

  // useController: gerencia value, onChange e onBlur automaticamente
  const {
    field,
    fieldState: { error, isTouched },
  } = useController(controllerProps);

  const hasError = Boolean(error);

  return (
    <View style={{ marginBottom: spacing.lg }}>
      <FormLabel label={label} />

      <TextInput
        // field: contém { value, onChange, onBlur, ref, name }
        value={field.value}
        onChangeText={field.onChange}
        onBlur={field.onBlur}
        ref={field.ref}
        placeholder={placeholder}
        placeholderTextColor={colors.textDisabled}
        secureTextEntry={secureTextEntry}
        autoCapitalize={autoCapitalize}
        keyboardType={keyboardType}
        style={[
          styles.input,
          {
            backgroundColor: colors.surface,
            borderColor: hasError ? colors.error : isTouched ? colors.brandPrimary : colors.border,
            borderRadius: radii.md,
            color: colors.textPrimary,
            paddingHorizontal: spacing.lg,
            paddingVertical: spacing.md,
            ...typography.bodyMedium,
          },
        ]}
        accessibilityLabel={label}
        accessibilityState={{ selected: isTouched }}
      />

      {/* Erro de validação exibido inline */}
      <FormError message={error?.message} />
    </View>
  );
}

const styles = StyleSheet.create({
  input: {
    borderWidth: 1.5,
  },
});
```

### 5.2 `FormLabel` — Label com Indicador

```tsx
// src/components/form/form-label.tsx
import React from 'react';
import { Text, View } from 'react-native';
import { useTheme } from '@theme';

type FormLabelProps = {
  label: string;
  required?: boolean;
};

export function FormLabel({ label, required = false }: FormLabelProps) {
  const { colors, typography, spacing } = useTheme();

  return (
    <View style={{ flexDirection: 'row', marginBottom: spacing.xs }}>
      <Text style={[typography.labelMedium, { color: colors.textSecondary }]}>
        {label}
      </Text>
      {required && (
        <Text style={{ color: colors.error, marginLeft: spacing.xxs }}>*</Text>
      )}
    </View>
  );
}
```

### 5.3 `FormError` — Mensagem de Erro Inline

```tsx
// src/components/form/form-error.tsx
import React from 'react';
import { Text } from 'react-native';
import { useTheme } from '@theme';

type FormErrorProps = {
  message?: string | null;
};

/**
 * Exibe erros de validação do Zod inline, abaixo do campo.
 * Retorna null se não houver mensagem — não ocupa espaço quando vazio.
 */
export function FormError({ message }: FormErrorProps) {
  const { colors, typography, spacing } = useTheme();

  if (!message) return null;

  return (
    <Text
      style={[
        typography.bodySmall,
        { color: colors.error, marginTop: spacing.xxs },
      ]}
      accessibilityRole="alert"
    >
      {message}
    </Text>
  );
}
```

### 5.4 Barrel Público — `src/components/form/index.ts`

```ts
// src/components/form/index.ts
export { FormField } from './form-field';
export type { FormFieldProps } from './form-field.types';
export { FormError } from './form-error';
export { FormLabel } from './form-label';
```

---

## 6. Componente de UI do Formulário — `components/login-form.tsx`

O componente de formulário é **stateless**: recebe tudo via props e renderiza os campos usando o `<FormField>` do UI Kit.

```tsx
// src/features/auth/login/components/login-form.tsx
import React from 'react';
import { View } from 'react-native';
import { type Control, type FormState } from 'react-hook-form';
import { useTheme } from '@theme';
import { FormField } from '@components/form';
import { Button } from '@components/button';
import { FormError } from '@components/form';
import { useErrorFeedback } from '@hooks/use-error-feedback'; // skill error-handling
import type { ApiError } from '@services/api/errors';
import type { LoginFormData } from '../schemas/login.schema';

type LoginFormProps = {
  control: Control<LoginFormData>;
  formState: FormState<LoginFormData>;
  onSubmit: () => void;
  isPending: boolean;
  /** Erro de servidor (ApiError) — diferente dos erros de campo do Zod */
  serverError: ApiError | null;
};

export function LoginForm({
  control,
  onSubmit,
  isPending,
  serverError,
}: LoginFormProps) {
  const { spacing } = useTheme();
  const { getErrorMessage } = useErrorFeedback();

  return (
    <View style={{ gap: spacing.sm }}>
      {/* Campo de e-mail — controlado pelo React Hook Form */}
      <FormField<LoginFormData>
        name="email"
        control={control}
        label="E-mail"
        placeholder="seu@email.com"
        keyboardType="email-address"
        autoCapitalize="none"
      />

      {/* Campo de senha */}
      <FormField<LoginFormData>
        name="password"
        control={control}
        label="Senha"
        placeholder="Mínimo 8 caracteres"
        secureTextEntry
      />

      {/*
        Erro de servidor (ApiError): exibido globalmente abaixo dos campos.
        Erros de validação por campo são tratados dentro do FormField.
        Segue o padrão da skill error-handling.
      */}
      {serverError && (
        <FormError message={getErrorMessage(serverError)} />
      )}

      <Button
        label={isPending ? 'Entrando...' : 'Entrar'}
        variant="primary"
        size="lg"
        isLoading={isPending}
        onPress={onSubmit}
        containerStyle={{ marginTop: spacing.md }}
      />
    </View>
  );
}
```

---

## 7. Tela — Montagem Final

```tsx
// src/features/auth/login/screens/login-screen.tsx
import React from 'react';
import { View, ScrollView } from 'react-native';
import { useTheme } from '@theme';
import { Text } from '@components/typography';
import { LoginForm } from '../components/login-form';
import { useLoginForm } from '../hooks/use-login-form';

export default function LoginScreen() {
  const { colors, spacing } = useTheme();
  const { control, formState, onSubmit, isPending, serverError, reset } = useLoginForm();

  return (
    <ScrollView
      style={{ flex: 1, backgroundColor: colors.background }}
      contentContainerStyle={{ padding: spacing.xl }}
      keyboardShouldPersistTaps="handled" // ← Importante no mobile: taps fecham o teclado
    >
      <Text variant="headlineMedium" style={{ marginBottom: spacing.xl }}>
        Entrar na conta
      </Text>

      <LoginForm
        control={control}
        formState={formState}
        onSubmit={onSubmit}
        isPending={isPending}
        serverError={serverError}
      />
    </ScrollView>
  );
}
```

> **`keyboardShouldPersistTaps="handled"`:** Impede que o `ScrollView` feche o teclado ao tocar em botões/pressables dentro do formulário. **Sempre use** em telas com formulário + botão de submit.

---

## 8. Erros de Validação na UI

### 8.1 Dois Tipos de Erro — Onde Exibir Cada Um

| Tipo de Erro | Origem | Onde Exibir |
|---|---|---|
| **Erro de campo (Zod)** | `formState.errors[campo]` | Inline, dentro de `<FormField>` via `useController` |
| **Erro de servidor (ApiError)** | `mutation.error` | Abaixo dos campos, via `<FormError message={getErrorMessage(error)} />` |
| **Erro de validação servidor (422)** | `ApiError.type === 'VALIDATION_ERROR'` | Por campo com `form.setError()` |

### 8.2 Integrando Erros do Servidor por Campo (`setError`)

Quando o servidor retorna erros de validação por campo (status 422), mapeie-os para os campos do formulário com `form.setError`:

```ts
// src/features/auth/register/hooks/use-register-form.ts
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { registerSchema, type RegisterFormData } from '../schemas/register.schema';
import { useRegister } from '../mutations/use-register';

export function useRegisterForm() {
  const form = useForm<RegisterFormData>({
    resolver: zodResolver(registerSchema),
    defaultValues: { name: '', email: '', phone: '', password: '', confirmPassword: '' },
    mode: 'onBlur',
  });

  const registerMutation = useRegister();

  const onSubmit = form.handleSubmit((data) => {
    registerMutation.mutate(data, {
      onError: (error) => {
        // Mapear erros de validação do servidor para os campos do formulário
        if (error.type === 'VALIDATION_ERROR') {
          const { fields } = error;
          // Itera os campos retornados pelo servidor e mapeia para o formulário
          (Object.keys(fields) as (keyof RegisterFormData)[]).forEach((field) => {
            form.setError(field, {
              type: 'server',
              message: fields[field]?.[0], // Primeira mensagem de erro do campo
            });
          });
        }
      },
    });
  });

  return {
    control: form.control,
    formState: form.formState,
    onSubmit,
    isPending: registerMutation.isPending,
    // Somente erros não-validação são exibidos globalmente
    serverError: registerMutation.error?.type !== 'VALIDATION_ERROR'
      ? registerMutation.error
      : null,
    reset: form.reset,
  };
}
```

### 8.3 Modo de Validação Recomendado

```ts
// ✅ Recomendado: valida ao sair do campo + revalida em tempo real após primeira submissão
useForm({ mode: 'onBlur', reValidateMode: 'onChange' });

// Cenários alternativos:
// - 'onSubmit': sem feedback de campo até o submit (mais simples)
// - 'onChange': feedback em tempo real desde o primeiro toque (mais agressivo)
```

---

## 9. Campos Customizados com `<Controller>`

Para campos que não são `TextInput` (ex: `Switch`, `Picker`, campos customizados do UI Kit), use `<Controller>` diretamente no componente de formulário:

```tsx
// src/features/settings/components/settings-form.tsx
import { Controller, type Control } from 'react-hook-form';
import { Switch, View } from 'react-native';
import { useTheme } from '@theme';
import { Text } from '@components/typography';
import type { SettingsFormData } from '../schemas/settings.schema';

type SettingsFormProps = {
  control: Control<SettingsFormData>;
  onSubmit: () => void;
};

export function SettingsForm({ control, onSubmit }: SettingsFormProps) {
  const { colors, spacing } = useTheme();

  return (
    <View style={{ gap: spacing.lg }}>
      {/* Controller para componentes não TextInput */}
      <Controller
        name="notificationsEnabled"
        control={control}
        render={({ field: { value, onChange } }) => (
          <View style={{ flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between' }}>
            <Text variant="bodyMedium">Notificações</Text>
            <Switch
              value={value}
              onValueChange={onChange}
              trackColor={{ true: colors.brandPrimary }}
              thumbColor={colors.textOnBrand}
            />
          </View>
        )}
      />
    </View>
  );
}
```

> **Regra:** Use `useController` (hook) para criar **componentes de campo extraídos com arquivo próprio** (ex: `FormField`, `FormSelect`). Use `<Controller>` (JSX) **diretamente no componente de formulário** para campos únicos e específicos como `Switch`, `Picker`, etc.

---

## 10. Formulário de Edição — Preenchimento com Dados do Servidor

Para formulários de edição, os `defaultValues` são preenchidos com dados retornados por um `useQuery`:

```ts
// src/features/profile/hooks/use-edit-profile-form.ts
import { useEffect } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { editProfileSchema, type EditProfileFormData } from '../schemas/edit-profile.schema';
import { useProfile } from '../queries/use-profile'; // useQuery
import { useUpdateProfile } from '../mutations/use-update-profile';

export function useEditProfileForm() {
  const { data: profile } = useProfile();
  const updateMutation = useUpdateProfile();

  const form = useForm<EditProfileFormData>({
    resolver: zodResolver(editProfileSchema),
    // defaultValues dinâmico: preenchido quando os dados chegam
    defaultValues: {
      name: '',
      bio: '',
      phone: '',
    },
  });

  // Quando os dados do servidor chegarem, redefine o formulário
  useEffect(() => {
    if (profile) {
      form.reset({
        name: profile.name,
        bio: profile.bio ?? '',
        phone: profile.phone ?? '',
      });
    }
  }, [profile, form.reset]);

  const onSubmit = form.handleSubmit((data) => {
    updateMutation.mutate(data);
  });

  return {
    control: form.control,
    formState: form.formState,
    onSubmit,
    isPending: updateMutation.isPending,
    serverError: updateMutation.error,
  };
}
```

> **`form.reset()` no `useEffect`:** É a forma correta de preencher `defaultValues` assíncronos. Não passe dados de API diretamente como `defaultValues` no `useForm` — os dados podem não estar disponíveis na primeira renderização.

---

## 11. Fluxo Completo — Diagrama

```text
[loginSchema.ts]          [use-login-form.ts]         [login-form.tsx]
      │                          │                            │
      │  z.object({...})         │  useForm({                 │  <FormField
      │  loginSchema ────────────►    resolver: zodResolver,  │    name="email"
      │                          │    defaultValues,          │    control={control}
  z.infer<typeof loginSchema>    │    mode: 'onBlur',         │  />
  → LoginFormData ───────────────►  })                        │
                                 │                            │  <FormError
                                 │  useLogin() (useMutation)  │    message={serverError}
                                 │  ────────────────────────  │  />
                                 │  { mutate, isPending,      │
                                 │    error: ApiError }       │  <Button onPress={onSubmit} />
                                 │                            │
                                 │  form.handleSubmit(mutate) │
                                 │  → onSubmit ───────────────►
```

---

## 12. Organização Completa de Arquivos

```text
src/
├── components/
│   └── form/
│       ├── form-field.tsx           ← Campo genérico com useController
│       ├── form-field.types.ts
│       ├── form-error.tsx           ← Erro inline (Zod ou servidor)
│       ├── form-label.tsx           ← Label com indicador de obrigatório
│       └── index.ts                 ← Barrel público
│
└── features/
    └── auth/
        └── login/
            ├── schemas/
            │   └── login.schema.ts      ← Schema Zod + tipo inferido
            ├── components/
            │   └── login-form.tsx       ← UI pura: campos + botão
            ├── hooks/
            │   └── use-login-form.ts    ← useForm + useMutation
            ├── mutations/
            │   └── use-login.ts         ← useMutation (react-query-patterns)
            ├── services/
            │   └── login.service.ts     ← Result<T, E> (error-handling)
            ├── screens/
            │   └── login-screen.tsx     ← Monta hook + form + layout
            └── index.ts                 ← Barrel público
```

---

---

## 15. Formulários Grandes — Decomposição em Seções

Formulários com **3 ou mais grupos semânticos de campos** devem ser divididos em **seções** dentro de `form/sections/`. Isso alinha com a skill `screen-decomposition` e garante que nenhum arquivo cresça de forma descontrolada.

### Quando Dividir

| Critério | Ação |
|---|---|
| Formulário com 1–2 grupos de campos | Pode permanecer em um único `*-form.tsx` |
| Formulário com 3+ grupos de campos distintos | **Obrigatório** criar `form/sections/` |
| Formulário com mais de ~120 linhas de JSX | **Obrigatório** criar `form/sections/` |

### Estrutura de Diretórios

```text
features/auth/register/
├── schemas/
│   └── register.schema.ts           ← Schema Zod + tipo inferido
├── hooks/
│   └── use-register-form.ts         ← useForm + useMutation
├── screens/
│   ├── register-screen.tsx          ← Orquestrador: hook + form
│   └── form/
│       ├── register-form.tsx            ← Componente-raiz: monta as seções + botão
│       └── sections/
│           ├── personal-data-section.tsx   ← Nome, e-mail, telefone
│           ├── address-section.tsx          ← CEP, logradouro, cidade, estado
│           ├── credentials-section.tsx      ← Senha e confirmação
│           └── index.ts                     ← Barrel das seções
└── index.ts
```

### Nomenclatura de Seções de Formulário

| Tipo | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case-section.tsx` | `personal-data-section.tsx` |
| Componente (função) | PascalCase + sufixo `Section` | `export function PersonalDataSection` |
| Pasta | sempre `sections/` | `form/sections/` |

### Componente-Raiz do Formulário (com seções)

```tsx
// src/features/auth/register/screens/form/register-form.tsx
import React from 'react';
import { View } from 'react-native';
import { type Control, type FormState } from 'react-hook-form';
import { useTheme } from '@theme';
import { Button } from '@components/button';
import { FormError } from '@components/form';
import { useErrorFeedback } from '@hooks/use-error-feedback';
import type { ApiError } from '@services/api/errors';
import type { RegisterFormData } from '../../schemas/register.schema';
import {
  PersonalDataSection,
  AddressSection,
  CredentialsSection,
} from './sections';

type RegisterFormProps = {
  control: Control<RegisterFormData>;
  formState: FormState<RegisterFormData>;
  onSubmit: () => void;
  isPending: boolean;
  serverError: ApiError | null;
};

export function RegisterForm({
  control,
  formState,
  onSubmit,
  isPending,
  serverError,
}: RegisterFormProps) {
  const { spacing } = useTheme();
  const { getErrorMessage } = useErrorFeedback();

  return (
    <View style={{ gap: spacing.xxl }}>
      {/* Cada seção recebe apenas control e formState — stateless */}
      <PersonalDataSection control={control} formState={formState} />
      <AddressSection control={control} formState={formState} />
      <CredentialsSection control={control} formState={formState} />

      {serverError && (
        <FormError message={getErrorMessage(serverError)} />
      )}

      <Button
        label={isPending ? 'Criando conta...' : 'Criar conta'}
        variant="primary"
        size="lg"
        isLoading={isPending}
        onPress={onSubmit}
      />
    </View>
  );
}
```

### Seção de Formulário (stateless)

```tsx
// src/features/auth/register/screens/form/sections/personal-data-section.tsx
import React from 'react';
import { View } from 'react-native';
import { type Control, type FormState } from 'react-hook-form';
import { useTheme } from '@theme';
import { FormField } from '@components/form';
import { Text } from '@components/typography';
import type { RegisterFormData } from '../../../schemas/register.schema';

type PersonalDataSectionProps = {
  control: Control<RegisterFormData>;
  formState: FormState<RegisterFormData>;
};

/**
 * Seção: Dados Pessoais
 * Stateless — recebe control e formState via props, sem lógica de negócio interna.
 */
export function PersonalDataSection({ control }: PersonalDataSectionProps) {
  const { spacing, colors } = useTheme();

  return (
    <View style={{ gap: spacing.md }}>
      <Text variant="titleMedium" style={{ color: colors.textPrimary }}>
        Dados Pessoais
      </Text>

      <FormField<RegisterFormData>
        name="name"
        control={control}
        label="Nome completo"
        placeholder="Seu nome completo"
        autoCapitalize="words"
      />

      <FormField<RegisterFormData>
        name="email"
        control={control}
        label="E-mail"
        placeholder="seu@email.com"
        keyboardType="email-address"
        autoCapitalize="none"
      />

      <FormField<RegisterFormData>
        name="phone"
        control={control}
        label="Telefone"
        placeholder="(11) 91234-5678"
        keyboardType="phone-pad"
      />
    </View>
  );
}
```

```ts
// src/features/auth/register/screens/form/sections/index.ts
export { PersonalDataSection } from './personal-data-section';
export { AddressSection } from './address-section';
export { CredentialsSection } from './credentials-section';
```

### Anti-Padrões

```tsx
// ❌ ERRADO — Seção importa hook de negócio (acoplamento indevido)
export function PersonalDataSection() {
  const { control } = useRegisterForm(); // ❌ seção não orquestra hooks
  return <FormField name="name" control={control} label="Nome" />;
}

// ✅ CORRETO — Seção puramente stateless
export function PersonalDataSection({ control }: PersonalDataSectionProps) {
  return <FormField<RegisterFormData> name="name" control={control} label="Nome" />;
}
```

> **Regra:** Seções são componentes de UI pura — recebem dados, nunca os buscam. Toda a orquestração permanece no hook (`use-*-form.ts`) e no componente-raiz do formulário (`*-form.tsx`).

---

## 14. Padrões e Anti-Padrões

### ✅ Faça

```ts
// ✅ Tipo inferido do schema — única fonte da verdade
export type LoginFormData = z.infer<typeof loginSchema>;

// ✅ zodResolver integra Zod com React Hook Form
useForm<LoginFormData>({ resolver: zodResolver(loginSchema), mode: 'onBlur' });

// ✅ useController para campos reutilizáveis extraídos
const { field, fieldState } = useController({ name, control });

// ✅ Erro de servidor exibido separadamente do erro de campo
{serverError && <FormError message={getErrorMessage(serverError)} />}

// ✅ form.reset() no useEffect para preencher dados assíncronos (formulário de edição)
useEffect(() => { if (data) form.reset(data); }, [data]);

// ✅ keyboardShouldPersistTaps="handled" em ScrollView com formulário
<ScrollView keyboardShouldPersistTaps="handled">
```

### ❌ Evite

```ts
// ❌ Não use useState para estado de campo — React Hook Form já gerencia isso
const [email, setEmail] = useState('');

// ❌ Não duplique o tipo — use z.infer
type LoginFormData = { email: string; password: string }; // redundante com o schema

// ❌ Não use register() em React Native — use Controller ou useController
// register() é para inputs HTML nativos (ref-based), não funciona com RN TextInput
<TextInput {...register('email')} /> // ❌ register não funciona em RN

// ❌ Não submeta diretamente sem handleSubmit — perde a validação
const onPress = () => mutate({ email, password }); // ❌ sem validação Zod

// ❌ Não misture lógica de negócio no componente de formulário (login-form.tsx)
// O componente de UI deve ser stateless — toda lógica vai no hook
export function LoginForm() {
  const { mutate } = useLogin(); // ❌ lógica de negócio no componente de UI
}
```

---

## 16. Checklist de Validação

Antes de entregar qualquer formulário, verifique:

- [ ] Schema Zod está em `schemas/<name>.schema.ts` dentro da sub-feature?
- [ ] Tipo do formulário é `z.infer<typeof schema>` — sem duplicação manual?
- [ ] `useForm` usa `{ resolver: zodResolver(schema), mode: 'onBlur' }`?
- [ ] A lógica de negócio (`useMutation`, navegação, store) está no hook e não no componente?
- [ ] Campos usam `useController` (para `FormField` extraído) ou `<Controller>` (para campos inline)?
- [ ] Erros de campo Zod são exibidos inline dentro de `FormField`?
- [ ] Erros de servidor (`ApiError`) são exibidos globalmente via `<FormError>` + `getErrorMessage()`?
- [ ] Erros 422 do servidor são mapeados para campos com `form.setError()`?
- [ ] `ScrollView` usa `keyboardShouldPersistTaps="handled"`?
- [ ] Formulários de edição usam `form.reset()` em `useEffect` quando os dados chegam?
- [ ] Componentes de campo usam tokens do `theme/` (nunca hardcode de cor, espaçamento ou fonte)?
- [ ] **Formulários com 3+ grupos de campos ou >~120 linhas de JSX foram divididos em seções em `form/sections/`?**
- [ ] **Seções de formulário são stateless (recebem `control` e `formState` via props, sem hooks de negócio internos)?**
- [ ] **Arquivos de seção seguem a nomenclatura `kebab-case-section.tsx`?**
