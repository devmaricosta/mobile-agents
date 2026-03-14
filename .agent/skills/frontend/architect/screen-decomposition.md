---
name: screen-decomposition
description: Regras para decomposição de telas (screens) grandes em seções reutilizáveis e testáveis, seguindo Clean Architecture e o Princípio da Responsabilidade Única. Define quando criar a pasta sections/, como nomear e estruturar cada seção, e como passar dados via props.
---

# Decomposição de Telas — Screen Decomposition

Você é um Arquiteto de Software Senior especialista em React Native com Expo Router. Sua responsabilidade é garantir que **nenhuma tela ultrapasse um tamanho gerenciável** de código. Telas excessivamente longas devem ser decompostas em **seções** — blocos visuais e semanticamente independentes que recebem suas dependências via props.

> **Alinhamento:** Esta skill é complementar à `arquiteto-react-native-features`. Siga todas as regras de nomenclatura, co-localização e estrutura de diretórios definidas lá. Esta skill adiciona apenas as regras de decomposição de telas.

---

## Quando usar esta skill

- **Tela nova ou existente** com mais de ~150 linhas de JSX em um único arquivo de screen.
- **Formulário grande** com múltiplos grupos de campos (ex: dados pessoais, endereço, segurança).
- **Code review:** Validar se telas grandes foram adequadamente decompostas.
- **Refatoração:** Limpar uma tela monolítica existente.

---

## Regra — Limite de Tamanho de Tela

| Tamanho do arquivo de screen | Ação |
|---|---|
| Até ~150 linhas de JSX | Pode permanecer em um único arquivo |
| Mais de ~150 linhas de JSX | **Obrigatório** decompor em subpastas dentro de `screens/` |
| Formulário com 3+ grupos de campos | **Obrigatório** extrair para pasta `form/` dentro de `screens/` |

> **Por quê 150 linhas?** É o ponto onde um arquivo começa a ter múltiplas responsabilidades visuais. Acima disso, o arquivo cresce em complexidade de leitura, testabilidade e manutenção. A screen deve permanecer como um **orquestrador**, não como uma coletânea de UI.

---

## Nome da Pasta de Decomposição — Semântico por Contexto

O nome da subpasta criada dentro de `screens/` deve refletir **o que ela contém semanticamente**. `sections/` é apenas uma opção genérica — **use o nome que melhor descreve o agrupamento**.

| Contexto | Nome sugerido | Exemplos de arquivos dentro |
|---|---|---|
| Múltiplos grupos genéricos de conteúdo | `sections/` | `personal-info-section.tsx`, `address-section.tsx` |
| Formulário grande | `form/` | `register-form.tsx` + `form/personal-data-section.tsx` |
| Etapas de um wizard / fluxo multi-step | `steps/` | `plan-selection-step.tsx`, `payment-step.tsx` |
| Cards de dashboard | `cards/` | `summary-card.tsx`, `activity-card.tsx` |
| Paineis de configuração | `panels/` | `notification-panel.tsx`, `privacy-panel.tsx` |
| Funcionalidade específica | nome da funcionalidade | `payment/`, `filters/`, `timeline/` |

> **Regra:** Escolha o nome que um novo desenvolvedor leria e **entenderia imediatamente** o que há dentro. Nomes genéricos demais (`components/`, `parts/`) devem ser evitados dentro de `screens/` — eles já têm significado bem definido na estrutura de features.

---

## Estrutura de Diretórios

### Tela simples (até ~150 linhas) — sem decomposição

```text
features/profile/edit/
├── hooks/
│   └── use-edit-profile.ts
├── screens/
│   └── edit-profile-screen.tsx        ← Tudo em um arquivo é aceitável
└── index.ts
```

### Tela grande com grupos genéricos de conteúdo — `sections/`

```text
features/profile/edit/
├── hooks/
│   └── use-edit-profile.ts
├── screens/
│   ├── edit-profile-screen.tsx         ← Orquestrador: importa seções + usa hook
│   └── sections/                        ← Nome genérico para grupos de conteúdo
│       ├── personal-info-section.tsx
│       ├── address-section.tsx
│       ├── security-section.tsx
│       └── index.ts
└── index.ts
```

### Tela com formulário grande — `form/`

```text
features/auth/register/
├── schemas/
│   └── register.schema.ts
├── hooks/
│   └── use-register-form.ts
├── screens/
│   ├── register-screen.tsx              ← Orquestrador
│   └── form/                            ← Nome semântico: tudo relacionado ao formulário
│       ├── register-form.tsx
│       ├── personal-data-section.tsx
│       ├── address-section.tsx
│       ├── credentials-section.tsx
│       └── index.ts
└── index.ts
```

### Tela de fluxo multi-step — `steps/`

```text
features/onboarding/
├── hooks/
│   └── use-onboarding.ts
├── screens/
│   ├── onboarding-screen.tsx            ← Orquestrador: controla qual step exibir
│   └── steps/                           ← Nome semântico: etapas do fluxo
│       ├── welcome-step.tsx
│       ├── plan-selection-step.tsx
│       ├── payment-step.tsx
│       └── index.ts
└── index.ts
```

### Tela de dashboard com múltiplos cards — `cards/`

```text
features/home/dashboard/
├── hooks/
│   └── use-dashboard.ts
├── screens/
│   ├── dashboard-screen.tsx             ← Orquestrador
│   └── cards/                           ← Nome semântico: cards do dashboard
│       ├── summary-card.tsx
│       ├── recent-activity-card.tsx
│       ├── quick-actions-card.tsx
│       └── index.ts
└── index.ts
```

---

## O Que é uma Seção (ou Bloco Decomposto)

Um **bloco decomposto** (seja chamado de seção, step, card, painel etc.) é uma unidade visual e semanticamente independente de uma tela. Cada bloco deve:

- Representar um **grupo coeso de conteúdo** (ex: "Dados Pessoais", "Etapa de Pagamento", "Card de Resumo").
- Ser um **componente stateless** — recebe tudo via props, não importa hooks de negócio diretamente.
- Ter **uma única responsabilidade** visual: renderizar aquele grupo de campos/conteúdo.
- Ter **nomes descritivos** em kebab-case com o sufixo relativo ao tipo: `personal-info-section.tsx`, `payment-step.tsx`, `summary-card.tsx`.

### Nomenclatura

| Tipo | Arquivo (kebab-case) | Componente (PascalCase) | Pasta |
|---|---|---|---|
| Seção genérica | `personal-info-section.tsx` | `PersonalInfoSection` | `sections/` |
| Formulário | `register-form.tsx` + partes dentro | `RegisterForm` | `form/` |
| Etapa de wizard | `payment-step.tsx` | `PaymentStep` | `steps/` |
| Card de dashboard | `summary-card.tsx` | `SummaryCard` | `cards/` |
| Painel específico | `notification-panel.tsx` | `NotificationPanel` | `panels/` |
| Funcionalidade específica | `payment-breakdown.tsx` | `PaymentBreakdown` | `payment/` |

> **Regra:** O nome da pasta e o sufixo dos arquivos devem ser **coerentes entre si** e descrever o domínio. Use `sections/` apenas quando não houver um nome mais específico e descritivo.

---

## Padrão de Implementação

### Seção de Tela (conteúdo informativo)

```tsx
// src/features/profile/edit/screens/sections/personal-info-section.tsx
import React from 'react';
import { View } from 'react-native';
import { type Control, type FormState } from 'react-hook-form';
import { useTheme } from '@theme';
import { FormField } from '@components/form';
import { Text } from '@components/typography';
import type { EditProfileFormData } from '../../schemas/edit-profile.schema';

type PersonalInfoSectionProps = {
  control: Control<EditProfileFormData>;
  formState: FormState<EditProfileFormData>;
};

/**
 * Seção: Dados Pessoais
 * Exibe os campos de nome, e-mail e telefone do formulário de edição de perfil.
 * Stateless — recebe control e formState via props, sem lógica de negócio interna.
 */
export function PersonalInfoSection({ control }: PersonalInfoSectionProps) {
  const { spacing, typography, colors } = useTheme();

  return (
    <View style={{ gap: spacing.md }}>
      <Text variant="titleMedium" style={{ color: colors.textPrimary }}>
        Dados Pessoais
      </Text>

      <FormField<EditProfileFormData>
        name="name"
        control={control}
        label="Nome completo"
        placeholder="Seu nome"
        autoCapitalize="words"
      />

      <FormField<EditProfileFormData>
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

```tsx
// src/features/profile/edit/screens/sections/index.ts
export { PersonalInfoSection } from './personal-info-section';
export { AddressSection } from './address-section';
export { SecuritySection } from './security-section';
```

### Tela Orquestradora

A screen importa as seções e monta o layout — não contém JSX de campo/conteúdo diretamente.

```tsx
// src/features/profile/edit/screens/edit-profile-screen.tsx
import React from 'react';
import { ScrollView, View } from 'react-native';
import { useTheme } from '@theme';
import { Text } from '@components/typography';
import { Button } from '@components/button';
import { useEditProfile } from '../hooks/use-edit-profile';
import {
  PersonalInfoSection,
  AddressSection,
  SecuritySection,
} from './sections';

export default function EditProfileScreen() {
  const { colors, spacing } = useTheme();
  const { control, formState, onSubmit, isPending, serverError } = useEditProfile();

  return (
    <ScrollView
      style={{ flex: 1, backgroundColor: colors.background }}
      contentContainerStyle={{ padding: spacing.xl, gap: spacing.xxl }}
      keyboardShouldPersistTaps="handled"
    >
      <Text variant="headlineMedium">Editar Perfil</Text>

      {/* Cada seção recebe apenas o que precisa */}
      <PersonalInfoSection control={control} formState={formState} />
      <AddressSection control={control} formState={formState} />
      <SecuritySection control={control} formState={formState} />

      {serverError && (
        <Text style={{ color: colors.error }}>{serverError.message}</Text>
      )}

      <Button
        label={isPending ? 'Salvando...' : 'Salvar alterações'}
        variant="primary"
        size="lg"
        isLoading={isPending}
        onPress={onSubmit}
      />
    </ScrollView>
  );
}
```

---

## Formulário Grande com Seções

Para formulários com muitos campos divididos em grupos semânticos, a decomposição ocorre dentro de `form/sections/`.

### Componente-Raiz do Formulário

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

### Seção de Formulário

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

```tsx
// src/features/auth/register/screens/form/sections/credentials-section.tsx
import React from 'react';
import { View } from 'react-native';
import { type Control, type FormState } from 'react-hook-form';
import { useTheme } from '@theme';
import { FormField } from '@components/form';
import { Text } from '@components/typography';
import type { RegisterFormData } from '../../../schemas/register.schema';

type CredentialsSectionProps = {
  control: Control<RegisterFormData>;
  formState: FormState<RegisterFormData>;
};

export function CredentialsSection({ control }: CredentialsSectionProps) {
  const { spacing, colors } = useTheme();

  return (
    <View style={{ gap: spacing.md }}>
      <Text variant="titleMedium" style={{ color: colors.textPrimary }}>
        Segurança
      </Text>

      <FormField<RegisterFormData>
        name="password"
        control={control}
        label="Senha"
        placeholder="Mínimo 8 caracteres"
        secureTextEntry
      />

      <FormField<RegisterFormData>
        name="confirmPassword"
        control={control}
        label="Confirmar senha"
        placeholder="Repita a senha"
        secureTextEntry
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

---

## Anti-Padrões

```tsx
// ❌ ERRADO — Seção importa hook de negócio diretamente (acoplamento)
export function PersonalInfoSection() {
  const { control } = useEditProfile(); // ❌ Seção não deve orquestrar hooks
  return <FormField name="name" control={control} />;
}

// ✅ CORRETO — Seção recebe tudo via props (stateless)
export function PersonalInfoSection({ control }: PersonalInfoSectionProps) {
  return <FormField<EditProfileFormData> name="name" control={control} />;
}
```

```tsx
// ❌ ERRADO — Screen contém todo o JSX de campos diretamente (arquivo gigante)
export default function EditProfileScreen() {
  const { control } = useEditProfile();
  return (
    <ScrollView>
      <Text>Dados Pessoais</Text>
      <FormField name="name" control={control} label="Nome" />
      <FormField name="phone" control={control} label="Telefone" />
      <Text>Endereço</Text>
      <FormField name="zipCode" control={control} label="CEP" />
      <FormField name="street" control={control} label="Rua" />
      <FormField name="city" control={control} label="Cidade" />
      {/* ... mais 20 campos ... */}
    </ScrollView>
  );
}

// ✅ CORRETO — Screen orquestra seções (limpa e legível)
export default function EditProfileScreen() {
  const { control, formState, onSubmit, isPending } = useEditProfile();
  return (
    <ScrollView keyboardShouldPersistTaps="handled">
      <PersonalInfoSection control={control} formState={formState} />
      <AddressSection control={control} formState={formState} />
      <Button onPress={onSubmit} isLoading={isPending} label="Salvar" />
    </ScrollView>
  );
}
```

---

## Fluxo de Implementação — Tela Grande

Ao criar ou refatorar uma tela com mais de ~150 linhas:

**1. Identificar os grupos semânticos de conteúdo**
Pergunte: "Quais blocos visuais têm responsabilidade própria?" (ex: dados pessoais, endereço, configurações)

**2. Criar a pasta `sections/`**
Dentro de `screens/` (ou `screens/form/` para formulários):
`src/features/<feature>/<sub>/screens/sections/`

**3. Criar cada arquivo de seção**
`personal-info-section.tsx`, `address-section.tsx` etc.

**4. Definir os tipos de props de cada seção**
Apenas o que a seção precisa — geralmente `control` e `formState` para formulários, ou dados tipados para seções informativas.

**5. Criar o barrel `sections/index.ts`**
Re-exportar todas as seções da pasta.

**6. Refatorar a screen para ser o orquestrador**
A screen deve apenas: chamar o hook, importar as seções e montar o layout.

---

## Checklist de Validação

Antes de entregar uma tela, verifique:

- [ ] A tela tem mais de ~150 linhas de JSX? Se sim, foi decomposta em uma subpasta dentro de `screens/`?
- [ ] O nome da pasta de decomposição é **semântico e contextual** (ex: `form/`, `steps/`, `cards/`, `sections/`)?
- [ ] Cada bloco decomposto é stateless e recebe tudo via props?
- [ ] Nenhum bloco importa hooks de negócio diretamente?
- [ ] Os arquivos seguem a nomenclatura `kebab-case` com sufixo descritivo?
- [ ] A subpasta tem um `index.ts` com re-exports?
- [ ] A screen (orquestradora) não contém campos/conteúdo diretamente — apenas importa e monta os blocos?
- [ ] Formulários com 3+ grupos de campos foram movidos para uma pasta `form/`?
- [ ] O componente-raiz do formulário (`*-form.tsx`) recebe `control` e `formState` via props da screen?
