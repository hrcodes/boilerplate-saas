# Guidelines — Cross-Cutting Concerns

> Regras que se aplicam a **todos** os repositórios do projeto — frontend, backend e qualquer serviço auxiliar.

---

## 1. API Contract First

### 1.1. Schemas Zod como Contrato por Projeto

**Motivo:**
Sem um contrato explícito, front e back divergem silenciosamente. Um campo renomeado no backend quebra o frontend sem erro de compilação.

**Regra:** Os schemas Zod do projeto vivem em `src/schemas/` (backend) e são espelhados em `src/schemas/` (frontend). Ambos os lados validam o que recebem — nenhuma das partes aceita `unknown` sem parsear. A sincronização é responsabilidade do desenvolvedor ao alterar contratos de API.

* **✅ Bom:**
  ```typescript
  // api/src/schemas/foodLog.ts  ←→  web/src/schemas/foodLog.ts (mantidos em sync)
  export const CreateFoodLogSchema = z.object({
    userId: z.string().uuid(),
    foodName: z.string().min(1),
    calories: z.number().positive(),
    loggedAt: z.string().datetime()
  })
  export type CreateFoodLogPayload = z.infer<typeof CreateFoodLogSchema>
  ```

---

## 2. Feature Flags

### 2.1. Controle de Features sem Deploy

**Motivo:**
Em SaaS AI-first, features novas de IA são experimentais. Feature flags permitem rollout gradual, A/B testing de prompts e desabilitar features em caso de custos inesperados — sem redeploy.

**Regra:** Toda feature nova de IA deve ser protegida por um feature flag antes do rollout completo.

* **✅ Bom:**
  ```typescript
  // src/application/featureFlags/IFeatureFlagService.ts
  export interface IFeatureFlagService {
    isEnabled(flag: FeatureFlag, userId?: string): Promise<boolean>
  }

  // No UseCase:
  if (!(await this.featureFlags.isEnabled('AI_FOOD_VISION_V2', userId))) {
    return this.legacyAnalyze(input)
  }
  ```

---

## 3. Health Check e Graceful Shutdown

### 3.1. Endpoints de Health para Orquestradores

**Regra:** Todo serviço deve expor dois endpoints:
- `GET /health` — retorna 200 se o **processo** está vivo (usado pelo load balancer para liveness check)
- `GET /ready` — retorna 200 apenas quando **todas as dependências** (DB, Redis, AI Provider) estão acessíveis (usado para readiness check — o processo não recebe tráfego até estar ready)

### 3.2. Graceful Shutdown

**Motivo:**
Encerrar o processo abruptamente em produção causa perda de requests em andamento e streams de IA corrompidos.

* **✅ Bom:**
  ```typescript
  // main/server.ts
  const shutdown = async (signal: string) => {
    logger.info({ signal }, 'Iniciando graceful shutdown...')
    server.close(async () => {
      await db.disconnect()     // ex: mongoose.disconnect() / prisma.$disconnect()
      await cache.quit()        // ex: redisClient.quit()
      logger.info('Servidor encerrado com sucesso.')
      process.exit(0)
    })
    setTimeout(() => process.exit(1), 10_000) // Force exit após 10s
  }
  process.on('SIGTERM', () => shutdown('SIGTERM'))
  process.on('SIGINT', () => shutdown('SIGINT'))
  ```

---

## 4. Documentação de API com Swagger (Homologação)

### 4.1. Swagger Restrito a Ambientes não-Produtivos

**Motivo:**
Expor o Swagger em produção representa risco de segurança — documenta publicamente todos os endpoints, payloads e possíveis vetores. Em homologação, porém, é essencial para que QA, stakeholders e o próprio time validem contratos sem precisar ler código.

**Regra:** O Swagger UI (`/api-docs`) deve ser montado **exclusivamente** quando `NODE_ENV !== 'production'`. A spec OpenAPI é gerada a partir dos schemas Zod já existentes via `@asteasolutions/zod-to-openapi`, eliminando duplicação de documentação.

* **⛔ Ruim:**
  ```typescript
  // Swagger sempre ativo, inclusive em produção
  app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(spec))
  ```

* **✅ Bom:**
  ```typescript
  // main/docs/swagger.ts
  import { OpenAPIRegistry, OpenApiGeneratorV3 } from '@asteasolutions/zod-to-openapi'
  import swaggerUi from 'swagger-ui-express'

  export function registerSwagger(app: Express): void {
    if (env.NODE_ENV === 'production') return // Guard obrigatório

    const registry = new OpenAPIRegistry()

    // Registrar schemas Zod já existentes — sem duplicação
    registry.register('CreateFoodLog', CreateFoodLogSchema)
    registry.register('FoodAnalysis', FoodAnalysisSchema)

    registry.registerPath({
      method: 'post',
      path: '/v1/food-logs',
      summary: 'Registra uma refeição',
      request: { body: { content: { 'application/json': { schema: CreateFoodLogSchema } } } },
      responses: { 201: { description: 'Refeição registrada com sucesso' } }
    })

    const generator = new OpenApiGeneratorV3(registry.definitions)
    const spec = generator.generateDocument({
      openapi: '3.0.0',
      info: { title: 'API Docs', version: '1.0.0', description: '⚠️ Ambiente de homologação' },
      servers: [{ url: env.API_BASE_URL }]
    })

    app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(spec))
    logger.info('Swagger disponível em /api-docs (somente homologação)')
  }
  ```

### 4.2. Proteção com Basic Auth em Homologação

**Motivo:**
Mesmo em staging, a documentação da API não deve ser pública para qualquer pessoa com a URL.

* **✅ Bom:**
  ```typescript
  import basicAuth from 'express-basic-auth'

  if (env.NODE_ENV !== 'production') {
    app.use('/api-docs', basicAuth({
      users: { [env.SWAGGER_USER]: env.SWAGGER_PASSWORD },
      challenge: true
    }))
    app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(spec))
  }
  ```

---

## 5. Conventional Commits

### 5.1. Padrão de Mensagens de Commit

**Motivo:**
O projeto possui integração contínua com Husky + Commitlint no hook `commit-msg`. Manter um padrão fixo facilita a geração de changelogs e o versionamento semântico. Commits fora do padrão são **bloqueados localmente** pelo Husky antes do push.

**Regra:** Todas as mensagens de commit devem seguir o **Conventional Commits**, obrigatoriamente referenciando o **número da task** no escopo.

* **Formato:** `tipo(numero_da_task): descrição em minúsculas`

* **⛔ Ruim:**
  ```bash
  git commit -m "corrige erro na criacao da sessao"
  git commit -m "feat: endpoint novo"
  ```

* **✅ Bom:**
  ```bash
  git commit -m "feat(1234): adiciona análise de refeição via visão computacional"
  git commit -m "fix(5678): corrige loop infinito no componente de streaming de IA"
  git commit -m "chore(9012): atualiza schema Zod para resposta do Anthropic"
  git commit -m "fix(54321): corrige crash no adapter do kafka ao perder conexão"
  ```
