---
name: business-rules
description: Regras de negócio do CAB 2.0 — calculadora de refeições baseada em blocos nutricionais (Dieta da Zona). Esta skill é o contrato central de negócio e DEVE ser consultada antes de implementar qualquer feature que envolva cálculo de blocos, montagem de refeições, alimentos, perfil de usuário ou vinculação com nutricionista.
---

# Regras de Negócio — CAB 2.0

Esta skill é o **coração do sistema**. Toda lógica de cálculo, validação e comportamento do app deve estar alinhada com estas regras. Ao implementar qualquer feature, consulte esta skill primeiro.

---

## 1. O Sistema de Blocos Nutricionais

### 1.1 Definição de Bloco

Um **bloco** é a unidade base da Dieta da Zona. Cada bloco equivale a uma proporção fixa de macronutrientes:

| Macronutriente | Gramas por bloco |
|---|---|
| Proteína | **7g** |
| Carboidrato | **9g** |
| Gordura | **1,5g** |

> **Regra Inegociável:** Os valores por bloco são **fixos no sistema** e **não podem ser alterados** pelo usuário, pelo nutricionista ou por qualquer configuração. São constantes de domínio.

```ts
// src/constants/blocks.ts — constantes de domínio (nunca alterar)
export const BLOCK_VALUES = {
  PROTEIN_GRAMS: 7,
  CARB_GRAMS: 9,
  FAT_GRAMS: 1.5,
} as const;
```

### 1.2 Quantidade de Blocos por Refeição

A **quantidade de blocos** que o usuário consome em cada refeição pode ser:

- Definida livremente pelo próprio usuário (em qualquer sessão)
- Definida pelo **nutricionista vinculado** (via configuração remota no perfil)

A quantidade de blocos por refeição não é calculada automaticamente pelo app — é uma **configuração explícita do usuário ou do nutricionista**.

---

## 2. Banco de Alimentos

### 2.1 Fonte dos Dados

Os alimentos são provenientes de **duas fontes**:

1. **API externa de nutrição** (ex: TACO, USDA, OpenFoodFacts) — base oficial de alimentos
2. **Alimentos cadastrados pelo próprio usuário** — armazenados localmente e na conta do usuário

```ts
// Tipos de origem do alimento
type FoodSource = 'api' | 'user_custom';

type Food = {
  id: string;
  name: string;
  source: FoodSource;
  macros: {
    proteinPer100g: number;
    carbPer100g: number;
    fatPer100g: number;
  };
  predominantCategory: FoodCategory; // categoria predominante para cálculo de blocos
  categories: FoodCategory[];        // todas as categorias às quais pertence
};
```

### 2.2 Categorias de Alimentos

Os alimentos são classificados em categorias que mapeiam para os macronutrientes dos blocos:

```ts
type FoodCategory = 'protein' | 'carb' | 'fat' | 'fruit' | 'vegetable';
```

> **Mapeamento de Categorias → Blocos:**
> - `protein` → bloco de proteína
> - `carb` | `fruit` | `vegetable` → bloco de carboidrato
> - `fat` → bloco de gordura

### 2.3 Categoria Predominante e Alimentos Multi-Bloco

**Um alimento pode pertencer a múltiplas categorias** e possui uma **categoria predominante** que define o ponto de partida do cálculo.

#### Regra do Alimento Multi-Bloco

Além da categoria predominante, um alimento pode contribuir com blocos de macronutrientes **secundários** se a porção que entrega 1 bloco do macro predominante também entregar **≥ 1 bloco** de outro macro. Nesse caso, o alimento é classificado como **Multi-Bloco** e ambas as contribuições são descontadas na montagem da refeição.

```ts
const MULTI_BLOCK_THRESHOLD = 1.0; // blocos secundários mínimos para ser Multi-Bloco

type SecondaryBlockContribution = {
  category: FoodCategory;
  blocksPerPredominantBlock: number; // contribuição por 1 bloco do macro predominante
};

type Food = {
  // ... campos anteriores
  predominantCategory: FoodCategory;
  categories: FoodCategory[];
  isMultiBlock: boolean;                         // true se qualquer secundário >= MULTI_BLOCK_THRESHOLD
  secondaryContributions: SecondaryBlockContribution[]; // só os que passaram o threshold
};
```

**Verificação do ovo inteiro:**

| Passo | Cálculo | Resultado |
|---|---|---|
| 1 bloco de proteína = 7g → porção de ovo | 7g ÷ (13g/100g) | **53,8g de ovo** |
| Gordura nessa porção | 53,8g × (11,5g/100g) | **6,2g de gordura** |
| Blocos de gordura entregues | 6,2g ÷ 1,5g | **4,1 blocos de gordura** |
| 4,1 ≥ 1,0 (threshold)? | ✅ Sim | **Multi-Bloco** |

```ts
// ✅ CORRETO — ovo é MULTI-BLOCO: 1 bloco de proteína traz ~4 blocos de gordura
const egg: Food = {
  id: 'egg-01',
  name: 'Ovo inteiro',
  source: 'api',
  macros: { proteinPer100g: 13, carbPer100g: 1.1, fatPer100g: 11.5 },
  predominantCategory: 'protein',
  categories: ['protein', 'fat'],
  isMultiBlock: true,
  secondaryContributions: [
    { category: 'fat', blocksPerPredominantBlock: 4.1 },
  ],
};

// ✅ CORRETO — frango grelhado (peito, magro) é MONO-BLOCO
// 1 bloco de proteína → ~56g de frango → 0,37 blocos de gordura < 1 → NÃO conta
const chickenBreast: Food = {
  id: 'chicken-01',
  name: 'Frango grelhado (peito)',
  source: 'api',
  macros: { proteinPer100g: 25, carbPer100g: 0, fatPer100g: 1.0 },
  predominantCategory: 'protein',
  categories: ['protein'],
  isMultiBlock: false,
  secondaryContributions: [],
};
```

### 2.4 Cálculo de Quantidade e Contribuição Multi-Bloco

Para calcular **quantos gramas** de um alimento equivalem a N blocos, e **quais macros secundários** aquela porção desconta automaticamente:

```ts
// src/utils/block-calculator.ts
import { BLOCK_VALUES } from '@constants/blocks';

const BLOCK_VALUE_MAP: Record<FoodCategory, number> = {
  protein: BLOCK_VALUES.PROTEIN_GRAMS,
  carb: BLOCK_VALUES.CARB_GRAMS,
  fat: BLOCK_VALUES.FAT_GRAMS,
  fruit: BLOCK_VALUES.CARB_GRAMS,
  vegetable: BLOCK_VALUES.CARB_GRAMS,
};

const MACRO_MAP: Record<FoodCategory, keyof Food['macros']> = {
  protein: 'proteinPer100g',
  carb: 'carbPer100g',
  fat: 'fatPer100g',
  fruit: 'carbPer100g',
  vegetable: 'carbPer100g',
};

/**
 * Retorna quantos gramas do alimento equivalem a N blocos do macro predominante.
 */
export function calculateGramsForBlocks(food: Food, blocks: number): number {
  const gramsPer100 = food.macros[MACRO_MAP[food.predominantCategory]];
  const gramsPerBlock = (BLOCK_VALUE_MAP[food.predominantCategory] / gramsPer100) * 100;
  return gramsPerBlock * blocks;
}

/**
 * Para um alimento Multi-Bloco, retorna quantos blocos de cada macro secundário
 * são descontados automaticamente ao consumir N blocos do macro predominante.
 *
 * Ex: 2 blocos de proteína de ovo → 8,2 blocos de gordura descontados automaticamente.
 */
export function calculateSecondaryBlockContributions(
  food: Food,
  predominantBlocks: number,
): Record<FoodCategory, number> {
  const result: Partial<Record<FoodCategory, number>> = {};
  for (const contrib of food.secondaryContributions) {
    result[contrib.category] = contrib.blocksPerPredominantBlock * predominantBlocks;
  }
  return result as Record<FoodCategory, number>;
}

/**
 * Classifica automaticamente um alimento como Multi-Bloco com base nos macros.
 * Usar ao cadastrar ou importar um alimento da API.
 */
export function classifyMultiBlock(food: Omit<Food, 'isMultiBlock' | 'secondaryContributions'>): Pick<Food, 'isMultiBlock' | 'secondaryContributions'> {
  const predominantMacroValue = food.macros[MACRO_MAP[food.predominantCategory]];
  const gramsPerPredominantBlock = (BLOCK_VALUE_MAP[food.predominantCategory] / predominantMacroValue) * 100;

  const secondaryContributions: SecondaryBlockContribution[] = [];

  for (const category of food.categories) {
    if (category === food.predominantCategory) continue;
    const secondaryMacroValue = food.macros[MACRO_MAP[category]];
    const secondaryBlocksPerPortion = (gramsPerPredominantBlock * secondaryMacroValue) / 100 / BLOCK_VALUE_MAP[category];
    if (secondaryBlocksPerPortion >= MULTI_BLOCK_THRESHOLD) {
      secondaryContributions.push({ category, blocksPerPredominantBlock: secondaryBlocksPerPortion });
    }
  }

  return {
    isMultiBlock: secondaryContributions.length > 0,
    secondaryContributions,
  };
}

---

## 3. Modos de Uso — Montagem de Refeição

O app possui **dois modos distintos** de montagem de refeições:

### 3.1 Modo Rápido (Quick Mode)

**Fluxo:**
1. Usuário define o **número de blocos** da refeição
2. Usuário seleciona os **alimentos desejados** por categoria
3. Para cada alimento, o app calcula e o usuário **confirma/ajusta a quantidade** em gramas

**Características especiais:**
- Existe o conceito de **refeição incompleta** — o usuário pode finalizar sem preencher todos os blocos de todos os macros
- O app **avisa** ao usuário sobre blocos faltantes, mas **não bloqueia** o salvamento
- Um mesmo alimento pode ser **adicionado mais de uma vez** (para quantidades diferentes, ex: 50g + 30g de frango)

```ts
type QuickMealStatus = 'complete' | 'incomplete';

type MealBlockSummary = {
  protein: { target: number; filled: number };
  carb: { target: number; filled: number };
  fat: { target: number; filled: number };
};

// Regra: filled < target → status = 'incomplete'
// Regra: filled >= target em todos os macros → status = 'complete'
function getMealStatus(summary: MealBlockSummary): QuickMealStatus {
  const allComplete =
    summary.protein.filled >= summary.protein.target &&
    summary.carb.filled >= summary.carb.target &&
    summary.fat.filled >= summary.fat.target;
  return allComplete ? 'complete' : 'incomplete';
}
```

### 3.2 Modo Diário (Daily Mode)

**Fluxo:**
1. Usuário define o **total de blocos diários** (configuração de perfil, válida por dias/semanas/meses)
2. A cada refeição montada, o app **desconta os blocos consumidos** do total diário
3. O saldo de blocos restantes é exibido continuamente

**Características especiais:**
- Não existe refeição incompleta no modo diário — a refeição é simplesmente somada ao consumo do dia
- O saldo diário de blocos **nunca vai abaixo de zero** no display (mas internamente pode registrar consumo acima do alvo)
- O ciclo diário **reseta à meia-noite** (horário local do dispositivo)

```ts
type DailyBlockPlan = {
  totalDailyBlocks: number;    // configurado pelo usuário ou nutricionista
  consumedBlocks: number;      // soma de todas as refeições do dia atual
  remainingBlocks: number;     // totalDailyBlocks - consumedBlocks (min: 0 no display)
  date: string;                // 'YYYY-MM-DD' — reseta a cada novo dia
};
```

---

## 4. Perfil do Usuário

### 4.1 Configuração de Blocos

O usuário pode configurar no seu perfil:
- **Total de blocos diários** — usado somente no Modo Diário

Essa configuração é **opcional** — o app funciona sem ela (Modo Rápido não a exige).

### 4.2 Vínculo com Nutricionista

O usuário pode **vincular um nutricionista** ao seu perfil.

**Regras do vínculo:**
- Quando vinculado, o nutricionista pode **sobrescrever** as configurações de blocos do usuário
- A dieta configurada pelo nutricionista fica **travada** — o usuário não pode editar enquanto o vínculo estiver ativo
- O usuário pode **desvincular o nutricionista** a qualquer momento; ao fazer isso, retoma controle total das suas configurações
- A desvinculação é uma ação do usuário, não do nutricionista

```ts
type UserProfile = {
  id: string;
  totalDailyBlocks: number | null;     // null = não configurado
  linkedNutritionistId: string | null; // null = sem vínculo
  isDietLocked: boolean;               // true quando há nutricionista vinculado
  // ... outros campos
};

// Regra: isDietLocked = true → bloquear edição de totalDailyBlocks e blockExtras na UI
// Regra: ao desvincular, isDietLocked → false, configurações do nutricionista são mantidas como ponto de partida
```

---

## 5. Nutricionista

### 5.1 O que o Nutricionista Pode Configurar

O nutricionista tem acesso a configurar no perfil do usuário vinculado:

- **`totalDailyBlocks`** — total de blocos diários da dieta
- **`blockExtras`** — blocos adicionais por macronutriente (ex: "+2 blocos extra de proteína")

```ts
type NutritionistDietConfig = {
  totalDailyBlocks: number;
  blockExtras?: {
    protein?: number;  // blocos adicionais de proteína
    carb?: number;     // blocos adicionais de carboidrato
    fat?: number;      // blocos adicionais de gordura
  };
};
```

### 5.2 Blocos Extras por Macronutriente

Quando o nutricionista configura blocos extras, o cálculo muda para aquele macronutriente específico:

```ts
// Exemplo: 15 blocos totais + 2 extras de proteína
// → proteína: 17 blocos, carb: 15 blocos, fat: 15 blocos
function applyBlockExtras(
  base: number,
  extras: NutritionistDietConfig['blockExtras']
): MealBlockSummary['protein'] & { carb: number; fat: number } {
  return {
    protein: { target: base + (extras?.protein ?? 0), filled: 0 },
    carb: { target: base + (extras?.carb ?? 0), filled: 0 },
    fat: { target: base + (extras?.fat ?? 0), filled: 0 },
  };
}
```

---

## 6. Refeições — Persistência e Histórico

### 6.1 Salvamento

Toda refeição montada (completa ou incompleta no Modo Rápido) pode ser salva pelo usuário.

Uma refeição salva contém:

```ts
type SavedMeal = {
  id: string;
  name: string;               // nome dado pelo usuário (ex: "Almoço terça")
  mode: 'quick' | 'daily';
  status: QuickMealStatus;    // 'complete' | 'incomplete' (sempre 'complete' no daily)
  items: MealItem[];          // lista de alimentos + quantidades
  blockSummary: MealBlockSummary;
  createdAt: string;          // ISO 8601
  isFavorite: boolean;
};

type MealItem = {
  food: Food;
  quantityGrams: number;
  blocks: number;             // blocos que esse item representa no seu macronutriente
  category: FoodCategory;     // categoria usada no cálculo (predominantCategory no momento do salvamento)
};
```

### 6.2 Histórico

- Todas as refeições salvas compõem o **histórico**
- O histórico é ordenado por data de criação (mais recente primeiro)
- Refeições podem ser marcadas como **favoritas** para acesso rápido

### 6.3 Reutilização

- O usuário pode **reutilizar uma refeição salva** — ela é copiada para uma nova sessão de montagem, onde pode ser ajustada antes de salvar novamente
- Reutilizar não altera a refeição original do histórico

---

## 7. Compartilhamento de Refeições

O usuário pode compartilhar uma refeição de **duas formas**:

### 7.1 Copiar para WhatsApp / Texto

Gera um texto formatado com os alimentos, quantidades e resumo de blocos, pronto para colar em qualquer app de mensagens.

```ts
// Exemplo de saída:
// 🍽️ Almoço - 4 blocos
// Frango grelhado: 140g (4 blocos de proteína)
// Batata-doce: 108g (4 blocos de carboidrato)
// Azeite: 9g (4 blocos de gordura)
```

### 7.2 Compartilhar na Comunidade

Envia a refeição para a **página de Comunidade** do app, onde outros usuários podem visualizar e se inspirar.

**Regras da Comunidade:**
- Somente refeições **completas** podem ser compartilhadas na comunidade (Modo Rápido com `status: 'complete'`; refeições do Modo Diário são sempre elegíveis)
- O usuário pode **remover** uma receita que ele mesmo publicou
- Receitas da comunidade são **somente leitura** para outros usuários (não editam a receita, apenas a copiam)

---

## 8. Invariantes e Validações Globais

> São regras que se aplicam em **qualquer contexto** do sistema. Toda lógica nova deve respeitar estas invariantes.

| # | Invariante | Onde validar |
|---|---|---|
| I-01 | `BLOCK_VALUES` nunca são alterados em runtime | Constante TypeScript imutável (`as const`) |
| I-02 | O cálculo de blocos usa **sempre** a `predominantCategory` do alimento | `block-calculator.ts` |
| I-03 | Um alimento sem `predominantCategory` não pode ser adicionado a uma refeição | Validação no serviço de montagem de refeição |
| I-04 | Alimentos com `quantityGrams <= 0` não podem ser adicionados | Validação de formulário + service |
| I-05 | Quando `isDietLocked = true`, qualquer tentativa de editar configurações de blocos é bloqueada | Guard na store do usuário |
| I-06 | O saldo diário de blocos exibido nunca é negativo | Computed no Modo Diário |
| I-07 | Ao desvincular nutricionista, `isDietLocked` volta para `false` imediatamente | Ação atômica na store |
| I-08 | Reutilizar uma refeição cria uma nova instância; o original permanece intacto | Service de cópia de refeição |
| I-09 | Apenas refeições completas ou do Modo Diário podem ser publicadas na Comunidade | Validação antes do share |
| I-10 | O ciclo diário de blocos reseta à meia-noite do horário local | Verificado na abertura do app e ao mudar de `date` |
| I-11 | Alimentos Multi-Bloco descontam **automaticamente** blocos dos macros secundários na refeição (não é opcional) | `calculateSecondaryBlockContributions` chamado junto com o cálculo principal |

---

## 9. Glossário do Domínio

> Use estes termos no código, nos comentários e nas interfaces TypeScript para manter consistência semântica.

| Termo | Descrição |
|---|---|
| **Bloco (Block)** | Unidade nutricional: 7g proteína + 9g carb + 1,5g gordura |
| **Categoria Predominante (Predominant Category)** | A categoria de macronutriente principal de um alimento, usada como base do cálculo de blocos |
| **Alimento Multi-Bloco (Multi-Block Food)** | Alimento cuja porção de 1 bloco predominante entrega ≥ 1 bloco de um macro secundário; ambos são descontados simultaneamente |
| **Alimento Mono-Bloco (Mono-Block Food)** | Alimento que contribui apenas com o macro da categoria predominante |
| **Threshold Multi-Bloco** | Valor fixo de `1,0 bloco` — mínimo de contribuição secundária para classificar como Multi-Bloco |
| **Contribuição Secundária (Secondary Contribution)** | Blocos de macro secundário entregues automaticamente por porção de 1 bloco predominante |
| **Modo Rápido (Quick Mode)** | Montagem pontual de refeição, sem vínculo com total diário |
| **Modo Diário (Daily Mode)** | Montagem de refeições com subtração do saldo diário de blocos |
| **Refeição Incompleta (Incomplete Meal)** | Refeição do Modo Rápido onde nem todos os macros foram preenchidos |
| **Saldo Diário (Daily Balance)** | Total de blocos diários menos os consumidos no dia |
| **Alimento Customizado (Custom Food)** | Alimento criado pelo próprio usuário, não oriundo da API |
| **Vínculo Nutricionista (Nutritionist Link)** | Relação entre usuário e nutricionista que bloqueia edição das configurações de dieta |
| **Dieta Travada (Locked Diet)** | Estado do perfil quando há nutricionista vinculado; configurações de blocos não editáveis pelo usuário |
| **Comunidade (Community)** | Feed público dentro do app para compartilhamento de refeições entre usuários |
| **Blocos Extras (Block Extras)** | Blocos adicionais por macronutriente configurados pelo nutricionista |

---

## 10. Mapeamento de Features → Regras

Use esta tabela para saber **quais regras consultar** ao implementar cada feature:

| Feature | Regras Obrigatórias |
|---|---|
| Calculadora de blocos / Montagem de refeição | §1, §2, §3, §8 |
| Perfil do usuário / Configurações | §4, §8 (I-05, I-07) |
| Vínculo com nutricionista | §4.2, §5, §8 (I-05, I-07) |
| Banco de alimentos / Cadastro personalizado | §2, §8 (I-02, I-03) |
| Histórico e reutilização de refeições | §6, §8 (I-08) |
| Compartilhamento (WhatsApp / Comunidade) | §7, §8 (I-09) |
| Modo Diário | §3.2, §8 (I-06, I-10) |
