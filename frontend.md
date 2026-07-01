# Guidelines — Front-End

**Stack:** Vue 3 + Vite + TypeScript + Pinia

> [!NOTE]
> **Nota sobre Código Legado:** Comportamentos marcados como "Ruim" não são permitidos em novos desenvolvimentos. Refatore gradativamente o que ainda não estiver aderente.

---

## 1. Vue 3 & Composition API

### 1.1. Uso obrigatório de `<script setup>`

**Motivo:**
A Composition API com `<script setup>` é o padrão oficial do Vue 3. A Options API (`data()`, `methods`, `computed`, `watch` como objetos) foi retida apenas por compatibilidade. O `<script setup>` elimina boilerplate, tem melhor inferência de TypeScript e performance superior em compilação.

**Regra:** Todo componente novo deve usar `<script setup lang="ts">`. É proibido criar componentes novos com Options API.

* **⛔ Ruim:**
  ```vue
  <script lang="ts">
  export default {
    data() { return { count: 0 } },
    methods: {
      increment() { this.count++ }
    }
  }
  </script>
  ```

* **✅ Bom:**
  ```vue
  <script setup lang="ts">
  import { ref } from 'vue'

  const count = ref(0)
  const increment = () => count.value++
  </script>
  ```

### 1.2. Composables no lugar de Mixins

**Motivo:**
Mixins criam colisão de nomes e ocultam a origem de propriedades, dificultando leitura e debug. Composables (funções `use*`) são explícitos, testáveis isoladamente e completamente tipados.

**Regra:** Lógica reutilizável deve ser extraída para composables em `src/composables/`. Mixins são proibidos em código novo.

* **⛔ Ruim:**
  ```typescript
  // mixins/formValidation.ts — origem opaca no componente
  export const formValidationMixin = {
    data() { return { errors: {} } },
    methods: { validate() { ... } }
  }
  ```

* **✅ Bom:**
  ```typescript
  // composables/useFormValidation.ts
  export function useFormValidation(schema: ZodSchema) {
    const errors = ref<Record<string, string>>({})
    const validate = (data: unknown) => { ... }
    return { errors, validate }
  }

  // No componente — origem explícita:
  const { errors, validate } = useFormValidation(loginSchema)
  ```

### 1.3. Computed para lógica de template

**Motivo:**
Condicionais extensas no template poluem o HTML, dificultam leitura e não são reativos de forma otimizada. `computed` é cacheado automaticamente pelo Vue.

* **⛔ Ruim:**
  ```vue
  <div v-if="user.age > 18 && user.isActive && !user.hasDebts && user.role === 'admin'">
  ```

* **✅ Bom:**
  ```vue
  <div v-if="isEligibleAdmin">
  ```
  ```typescript
  const isEligibleAdmin = computed(
    () => user.value.age > 18 && user.value.isActive && !user.value.hasDebts && user.value.role === 'admin'
  )
  ```

### 1.4. `watch` — uso restrito

**Motivo:**
O uso excessivo de watchers causa loops de reatividade, side-effects ocultos e falhas intermitentes em testes. A maioria dos casos de `watch` pode ser substituída por `computed`.

**Regra:** Use `watch` apenas para side-effects reais (chamadas de API em resposta a mudança de rota, sincronização com storage externo). Para derivar dados, use sempre `computed`.

* **⛔ Ruim:**
  ```typescript
  watch(firstName, (val) => {
    fullName.value = val + ' ' + lastName.value
  })
  ```

* **✅ Bom:**
  ```typescript
  const fullName = computed(() => `${firstName.value} ${lastName.value}`)
  ```

### 1.5. Declaração explícita de Props e Emits

**Motivo:**
Declarações tipadas evitam warnings no console, documentam a API pública do componente e permitem validação em tempo de compilação.

* **✅ Bom:**
  ```typescript
  const props = defineProps<{
    userId: string
    isLoading?: boolean
  }>()

  const emit = defineEmits<{
    submit: [payload: UserPayload]
    cancel: []
  }>()
  ```

### 1.6. Nomenclatura de Componentes

**Motivo:**
O prefixo **"V"** padroniza o ecossistema, previne colisão com tags nativas do HTML e facilita a identificação visual de componentes proprietários no template.

* **⛔ Ruim:** `ButtonSubmit.vue`, `table-data.vue`
* **✅ Bom:** `VButtonSubmit.vue`, `VTableData.vue`

### 1.7. Componentes Burros vs. Containers Inteligentes

**Motivo:**
Componentes de apresentação ("dumb") são genéricos, reutilizáveis e fáceis de testar. Eles não conhecem stores, APIs ou regras de negócio. Essa responsabilidade é exclusiva das Views e Containers.

* **⛔ Ruim:** Um `VCard.vue` que dispara `store.fetchPatient()` no `onMounted`.
* **✅ Bom:** A View busca os dados e os passa via prop: `<VCard :data="cardData" />`. O `VCard` apenas renderiza.

### 1.8. Proibição de `v-html`

**Motivo:**
`v-html` abre vetor para XSS. Conteúdo gerado por usuário ou por IA nunca deve ser renderizado sem sanitização. Para conteúdo Markdown de respostas de IA, use `DOMPurify` + `marked`.

* **⛔ Ruim:**
  ```vue
  <div v-html="aiResponse"></div>
  ```

* **✅ Bom:**
  ```vue
  <div v-html="sanitizedResponse"></div>
  ```
  ```typescript
  import DOMPurify from 'dompurify'
  import { marked } from 'marked'

  const sanitizedResponse = computed(
    () => DOMPurify.sanitize(marked(aiResponse.value))
  )
  ```

### 1.9. Proibição de `setTimeout` para Lifecycle

**Regra:** É estritamente proibido usar `setTimeout` para aguardar dados da store, renderizações ou APIs. Use `await` na action, ou `nextTick` para aguardar o DOM.

* **⛔ Ruim:**
  ```typescript
  onMounted(() => {
    store.fetchData()
    setTimeout(() => { localData.value = store.data }, 1000)
  })
  ```

* **✅ Bom:**
  ```typescript
  onMounted(async () => {
    await store.fetchData()
    localData.value = store.data
  })
  ```

---

## 2. Pinia — Gerenciamento de Estado

### 2.1. Definição de Stores com `defineStore`

**Motivo:**
Pinia é o gerenciador de estado oficial do Vue 3. É mais simples, totalmente tipado, DevTools-friendly e dispensa os padrões verbosos do Vuex (mutations, mapState, mapGetters).

**Regra:** Vuex é proibido em projetos novos. Use Pinia. Organize as stores por domínio em `src/stores/`.

* **✅ Bom:**
  ```typescript
  // src/stores/useAuthStore.ts
  export const useAuthStore = defineStore('auth', () => {
    const user = ref<User | null>(null)
    const isAuthenticated = computed(() => user.value !== null)

    async function login(credentials: LoginPayload) {
      user.value = await authService.login(credentials)
    }

    return { user, isAuthenticated, login }
  })
  ```

### 2.2. Nunca Mutar Estado Fora da Store

**Motivo:**
Mutações diretas pelo componente destroem a rastreabilidade do DevTools e criam side-effects imprevisíveis.

* **⛔ Ruim:**
  ```typescript
  const store = useAuthStore()
  store.user = null // mutação direta
  ```

* **✅ Bom:**
  ```typescript
  // Exponha uma action na store
  function logout() { user.value = null }

  // No componente:
  store.logout()
  ```

### 2.3. Lógica Derivada como Computed (getters)

**Motivo:**
Lógica de derivação centralizada na store previne inconsistência e duplicação espalhada por componentes.

* **⛔ Ruim:**
  ```typescript
  // Em múltiplos componentes:
  const bloodExams = exams.value.filter(e => e.type === 'blood')
  ```

* **✅ Bom:**
  ```typescript
  // Na store:
  const bloodExams = computed(() => exams.value.filter(e => e.type === 'blood'))
  ```

---

## 3. AI UI Patterns

### 3.1. Componente de Streaming de Resposta

**Motivo:**
Respostas de IA chegam em stream (chunks). Renderizar tudo apenas no final degrada a UX drasticamente. O componente deve consumir e exibir os chunks conforme chegam via SSE.

* **✅ Bom:**
  ```vue
  <!-- components/VAIStreamResponse.vue -->
  <script setup lang="ts">
  const props = defineProps<{ sessionId: string }>()
  const chunks = ref('')
  const isStreaming = ref(false)

  async function startStream() {
    isStreaming.value = true
    const source = new EventSource(`/api/ai/stream/${props.sessionId}`)
    source.onmessage = (e) => {
      if (e.data === '[DONE]') { isStreaming.value = false; source.close(); return }
      chunks.value += JSON.parse(e.data).text
    }
  }
  </script>

  <template>
    <div class="ai-response">
      <div v-html="sanitizedChunks" />
      <VTypingIndicator v-if="isStreaming" />
    </div>
  </template>
  ```

### 3.2. Skeleton Loading para Operações de IA

**Motivo:**
Operações de IA têm latência variável. Spinners genéricos aumentam a percepção de espera. Skeletons comunicam estrutura esperada e reduzem ansiedade do usuário.

**Regra:** Toda área que exibe resposta de IA deve ter um componente skeleton correspondente ativado durante o loading.

* **✅ Bom:**
  ```vue
  <template>
    <VAIResponseSkeleton v-if="aiStore.isLoading" />
    <VAIStreamResponse v-else :session-id="sessionId" />
  </template>
  ```

### 3.3. Estado de Erro e Retry para IA

**Motivo:**
APIs de IA falham (rate limit, timeout, contexto excedido). O usuário precisa de feedback claro e a possibilidade de tentar novamente sem recarregar a página.

* **✅ Bom:**
  ```vue
  <VAlertError
    v-if="aiStore.error"
    :message="aiStore.error.userFriendlyMessage"
    @retry="aiStore.retryLastRequest()"
  />
  ```

### 3.4. Indicador de Custo / Créditos (quando aplicável)

**Motivo:**
Em produtos com modelo de créditos ou tiers, o usuário deve ter visibilidade do consumo antes de acionar operações custosas.

* **✅ Bom:**
  ```vue
  <VCreditCostBadge :estimated-tokens="estimatedCost" />
  <VButton @click="analyze" :disabled="credits < estimatedCost">
    Analisar Refeição
  </VButton>
  ```

---

## 4. TypeScript

### 4.1. Tipagem Estrita — Proibição de `any`

**Motivo:**
`any` desativa o compilador TypeScript e mascara bugs que só aparecem em produção.

* **⛔ Ruim:** `function process(data: any) { ... }`
* **✅ Bom:**
  ```typescript
  interface FoodItem { id: string; name: string; calories: number }
  function process(data: FoodItem) { ... }
  ```

### 4.2. Validação em Runtime com Zod

**Motivo:**
TypeScript garante tipos em compilação, mas dados externos (APIs de IA, webhooks, formulários) chegam como `unknown` em runtime. Zod valida e faz parse seguro, gerando erros descritivos.

**Regra:** Toda resposta de API externa e toda entrada de usuário deve ser validada com Zod antes de ser usada na aplicação. Os schemas Zod vivem em `src/schemas/`.

* **✅ Bom:**
  ```typescript
  // src/schemas/food.ts
  export const FoodItemSchema = z.object({
    id: z.string().uuid(),
    name: z.string().min(1),
    calories: z.number().positive(),
    macros: z.object({
      protein: z.number(), carbs: z.number(), fat: z.number()
    })
  })
  export type FoodItem = z.infer<typeof FoodItemSchema>

  // No service:
  const result = FoodItemSchema.safeParse(apiResponse)
  if (!result.success) handleValidationError(result.error)
  ```

### 4.3. Sem Magic Strings — Use Enums ou `as const`

* **⛔ Ruim:** `if (status !== 'Disponivel') { ... }`
* **✅ Bom:**
  ```typescript
  const Status = { AVAILABLE: 'Disponivel', UNAVAILABLE: 'Indisponivel' } as const
  type Status = typeof Status[keyof typeof Status]

  if (status !== Status.AVAILABLE) { ... }
  ```

### 4.4. Nomenclatura — Inglês + camelCase

**Regra:** Variáveis, funções e propriedades em Inglês + camelCase. PascalCase apenas para Classes, Interfaces, Types e Enums. Arquivos de componente em PascalCase (`VButton.vue`), demais em camelCase (`useAuthStore.ts`).

---

## 5. Estilos

### 5.1. Variáveis CSS — Proibição de Cores Hardcoded

**Regra:** É expressamente proibido usar cores diretas (Hex, RGB, HSL) no CSS. Utilize sempre as variáveis de CSS mapeadas em `src/assets/css/variables.css`.

* **⛔ Ruim:** `background-color: #F87171;`
* **✅ Bom:** `background-color: var(--color-danger-light);`

### 5.2. Estilos Sempre Scoped

**Regra:** Todo bloco `<style>` em componentes deve ter o atributo `scoped`.

### 5.3. Ícones SVG com `currentColor`

**Regra:** Ícones são criados como componentes Vue em `src/icons/`. O atributo `fill` deve sempre usar `currentColor` para herdar cor do CSS pai.

* **✅ Bom:**
  ```vue
  <!-- VIconAlert.vue -->
  <template>
    <svg viewBox="0 0 24 24"><path fill="currentColor" d="..." /></svg>
  </template>
  ```

### 5.4. Catálogo de Componentes (Storybook)

**Regra:** Antes de criar um componente de UI, consulte o Storybook. Componentes novos **devem** ser documentados no Storybook antes de ir para produção.

---

## 6. Testes

### 6.1. Localização e Extensão

**Regra:** Arquivos de teste usam extensão `.spec.ts` e residem na **mesma pasta** do componente ou composable que testam.

### 6.2. Bloco Describe em Português

**Regra:** O bloco `describe` e os casos `it` devem estar sempre em **Português**, refletindo a intenção do teste.

* **⛔ Ruim:** `describe('VButton tests', () => {})`
* **✅ Bom:** `describe('VButton.vue - Comportamento do botão principal de ações', () => {})`

### 6.3. Testes de Comportamento, Não de Implementação

**Regra:** Teste o que o usuário experimenta (clique, emissão de evento, texto visível), não o estado interno da VM.

* **⛔ Ruim:**
  ```typescript
  wrapper.vm.isOpen = true
  expect(wrapper.vm.isOpen).toBe(true)
  ```

* **✅ Bom:**
  ```typescript
  await wrapper.find('.btn-save').trigger('click')
  expect(wrapper.emitted().submit).toBeTruthy()
  ```

### 6.4. Sem Condicionais ou Loops dentro de `it`

**Regra:** Cada `it` deve ser plano e cobrir um único cenário. Loops e ifs dentro de `it` mascaram falhas.

### 6.5. Console Limpo

**Regra:** Nenhum teste pode gerar warnings ou erros no console. Faça mock correto de props, stores e plugins antes da montagem.

---

## 7. Estrutura

### 7.1. Importação Absoluta via `@/`

**Regra:** Proibido o uso de caminhos relativos extensos. Use sempre o alias `@/`.

* **⛔ Ruim:** `import Comp from '../../../../components/Comp.vue'`
* **✅ Bom:** `import Comp from '@/components/Comp.vue'`
