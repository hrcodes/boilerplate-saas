# Guidelines — Back-End (BFF)

**Stack:** Node.js + Express + TypeScript (Clean Architecture)

> [!NOTE]
> **Nota sobre Código Legado:** Comportamentos marcados como "Ruim" não são permitidos em novos desenvolvimentos. Refatore gradativamente o que ainda não estiver aderente.

---

## 1. Clean Architecture & Camadas

### 1.1. Respeito à Regra de Dependência

**Motivo:**
O projeto foi estruturado utilizando a Clean Architecture. Misturar as camadas destrói o isolamento do domínio e torna o código impossível de ser testado isoladamente. A camada de aplicação/domínio nunca deve conhecer detalhes da infraestrutura.

**Regra:** Arquivos dentro de `src/application` não podem importar pacotes externos de banco de dados (ORMs, ODMs), mensageria ou frameworks HTTP. A camada de domínio conhece apenas interfaces — a `infra` as implementa.

* **⛔ Ruim:** Importar um ORM ou o Express diretamente dentro de um UseCase ou Controller em `application/`.
* **✅ Bom:** O Controller recebe `IPatientRepository` via injeção. A implementação `MongoPatientRepository` fica em `infra/` e é injetada pela `main/factories/`.

### 1.2. Factories na Camada Main

**Motivo:**
Centralizar a criação das instâncias na camada `main/factories` permite que os Controllers e Casos de Uso não precisem saber como construir suas próprias dependências, facilitando o mock nos testes.

**Regra:** Nunca use `new` em dependências complexas dentro de Controllers. Toda criação de instâncias fica em `src/main/factories/`.

* **✅ Bom:**
  ```typescript
  // main/factories/makeCreateFoodLogController.ts
  export const makeCreateFoodLogController = () =>
    new CreateFoodLogController(
      new MongoFoodLogRepository(),
      new ZodCreateFoodLogValidator(),
      makeAIFoodAnalyzer() // AI também é injetado como interface
    )
  ```

### 1.3. Adapters Express na Camada Main

**Motivo:**
O domínio da aplicação não deve conhecer os contratos específicos de frameworks externos (`req`/`res` do Express). O Adapter recebe a requisição no formato nativo do framework, converte para um formato genérico e aciona o Controller.

**Regra:** Controllers não conhecem `req`/`res` do Express. Um Adapter na `main/` faz a ponte, traduzindo para `HttpRequest`/`HttpResponse` genéricos.

* **✅ Bom:**
  ```typescript
  // main/adapters/expressRouteAdapter.ts
  export const adaptRoute = (controller: IController) =>
    async (req: Request, res: Response) => {
      const httpResponse = await controller.handle({
        body: req.body, params: req.params, query: req.query,
        userId: req.userId // injetado pelo middleware de auth
      })
      res.status(httpResponse.statusCode).json(httpResponse.body)
    }
  ```

### 1.4. Componentização (S.O.L.I.D.)

**Motivo:**
Arquivos gigantes com métodos de centenas de linhas são difíceis de ler, manter e impossíveis de testar corretamente.

**Regra:** Aplique o SRP. O controller atua apenas como orquestrador. A validação é repassada a um `Validator`, o acesso a banco a um `Repository` e os cálculos lógicos a um `UseCase`.

* **⛔ Ruim:** Um único método `handle()` com 200 linhas responsável por validar dados, formatar strings, verificar existência no banco, salvar registro e disparar evento Kafka.
* **✅ Bom:** Controller delega cada responsabilidade a uma classe especializada.

### 1.5. Helpers e Utils

**Motivo:**
É comum criar um arquivo genérico `utils.ts` gigantesco ("God Object") com dezenas de funções não relacionadas.

**Regra:** Funções de Helper devem ser puras (sem efeitos colaterais) e altamente específicas. Separe-os por contexto lógico.

* **⛔ Ruim:** Um `src/utils.ts` misturando validação de CPF, cálculos matemáticos, formatação de datas e criptografia.
* **✅ Bom:** `dateFormatterHelper.ts`, `cpfValidatorHelper.ts` — alta coesão por contexto.

---

## 2. Validação com Zod

### 2.1. Zod Substitui Joi

**Motivo:**
Zod é isomórfico (funciona em front e back), tem inferência de tipos nativa e gera erros mais descritivos. Joi é proibido em projetos novos.

**Regra:** Crie um `ZodValidator` genérico que implementa `IValidator`. Os schemas Zod vivem em `src/schemas/` e são espelhados no frontend.

* **✅ Bom:**
  ```typescript
  // application/validators/ZodCreateFoodLogValidator.ts
  export class ZodCreateFoodLogValidator implements IValidator {
    validate(input: unknown) {
      const result = CreateFoodLogSchema.safeParse(input)
      return result.success ? null : new ValidationError(result.error.format())
    }
  }
  ```

### 2.2. Importação Absoluta via `@/`

**Regra:** Proibido o uso de caminhos relativos extensos. Use sempre o alias `@/` configurado no `tsconfig.json`.

* **⛔ Ruim:** `import { UserRepository } from '../../../../infra/databases/repositories/user'`
* **✅ Bom:** `import { UserRepository } from '@/infra/databases/repositories/user'`

### 2.3. Nomenclatura — Inglês + camelCase

**Regra:** Variáveis, funções e propriedades em Inglês + camelCase. PascalCase apenas para Classes e Interfaces.

* **⛔ Ruim:** `const dataPrometida = '2023-10-01'`
* **✅ Bom:** `const promisedDate = '2023-10-01'`

---

## 3. Error Handling

### 3.1. Hierarquia de Erros Customizada

**Motivo:**
Lançar `new Error('algo deu errado')` sem tipo perde contexto, dificulta logging e impede tratamento diferenciado por camada. Uma hierarquia de erros garante mensagens consistentes e status HTTP corretos.

**Regra:** Nunca lance `Error` genérico nas camadas de Application. Use sempre a hierarquia de erros do domínio.

* **✅ Bom:**
  ```typescript
  // src/application/errors/index.ts
  export class AppError extends Error {
    constructor(public message: string, public statusCode: number, public code: string) {
      super(message)
    }
  }
  export class NotFoundError extends AppError {
    constructor(entity: string) { super(`${entity} não encontrado`, 404, 'NOT_FOUND') }
  }
  export class ValidationError extends AppError {
    constructor(public details: unknown) { super('Dados inválidos', 400, 'VALIDATION_ERROR') }
  }
  export class UnauthorizedError extends AppError {
    constructor() { super('Não autorizado', 401, 'UNAUTHORIZED') }
  }
  export class AIProviderError extends AppError {
    constructor(public provider: string, cause?: Error) {
      super(`Falha no provedor de IA: ${provider}`, 502, 'AI_PROVIDER_ERROR')
      this.cause = cause
    }
  }
  ```

### 3.2. Middleware Global de Error Handler

**Motivo:**
Sem um handler centralizado, cada Controller precisa tratar erros individualmente, gerando inconsistência no formato da resposta.

* **✅ Bom:**
  ```typescript
  // main/middlewares/errorHandler.ts
  export const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
    logger.error({ err, requestId: req.requestId })

    if (err instanceof AppError) {
      return res.status(err.statusCode).json({ error: err.code, message: err.message })
    }
    return res.status(500).json({ error: 'INTERNAL_ERROR', message: 'Erro interno do servidor' })
  }
  ```

---

## 4. Logging Estruturado

### 4.1. Proibição de `console.log`

**Motivo:**
`console.log` não tem nível de severidade, não é estruturado (JSON), não carrega correlation ID e não integra com plataformas de observabilidade (Datadog, Loki, CloudWatch). Se o projeto usa logging estruturado com Pino, `console.log` é redundante e inconsistente com o restante do pipeline de logs.

**Regra:** `console.log/warn/error` são **proibidos** no código da aplicação. Use o logger centralizado baseado em **Pino**.

* **⛔ Ruim:** `console.log('Usuário criado:', userId)`
* **✅ Bom:**
  ```typescript
  // src/infra/logger/index.ts
  import pino from 'pino'
  export const logger = pino({ level: process.env.LOG_LEVEL || 'info' })

  // No UseCase/Controller:
  logger.info({ userId, action: 'user.created' }, 'Usuário criado com sucesso')
  logger.error({ err, userId }, 'Falha ao criar usuário')
  ```

### 4.2. Correlation ID em Cada Request

**Motivo:**
Em logs distribuídos, rastrear uma requisição completa (do HTTP ao banco ao AI Provider) é impossível sem um ID único de correlação.

* **✅ Bom:**
  ```typescript
  // main/middlewares/requestId.ts
  import { randomUUID } from 'crypto'
  export const requestIdMiddleware = (req: Request, res: Response, next: NextFunction) => {
    req.requestId = req.headers['x-request-id'] as string || randomUUID()
    res.setHeader('x-request-id', req.requestId)
    next()
  }
  ```

---

## 5. Segurança

### 5.1. Validação de Variáveis de Ambiente na Inicialização

**Motivo:**
A aplicação não deve subir com variáveis de ambiente faltando ou malformadas. Isso previne falhas silenciosas em produção (ex: chave de API de IA vazia ou JWT secret fraco).

**Regra:** Valide todas as variáveis de ambiente no boot com Zod. Se a validação falhar, a aplicação **não deve iniciar**.

* **✅ Bom:**
  ```typescript
  // src/main/config/env.ts
  const EnvSchema = z.object({
    NODE_ENV:          z.enum(['development', 'test', 'production']),
    DATABASE_URL:      z.string().url(),
    JWT_SECRET:        z.string().min(32),
    REDIS_URL:         z.string().url().optional(),
    // Adicione as chaves específicas do projeto aqui
    // ex: STRIPE_SECRET_KEY, ANTHROPIC_API_KEY, SUPABASE_SERVICE_ROLE_KEY
  })

  const result = EnvSchema.safeParse(process.env)
  if (!result.success) {
    console.error('FATAL: Variáveis de ambiente inválidas:\n', result.error.format())
    process.exit(1)
  }
  export const env = result.data
  ```

> [!NOTE]
> O `console.error` acima é a única exceção permitida para `console.*` — ocorre antes do logger Pino ser inicializado, durante o boot de validação de ambiente.

### 5.2. Rate Limiting por Usuário e por Endpoint de IA

**Motivo:**
Endpoints de IA têm custo real por request. Um usuário mal-intencionado (ou um loop acidental) pode gerar centenas de dólares em segundos.

**Regra:** Todo endpoint que chama um provedor de IA deve ter rate limiting individual por `userId` além do rate limiting global por IP.

* **✅ Bom:**
  ```typescript
  import rateLimit from 'express-rate-limit'
  import RedisStore from 'rate-limit-redis'

  export const aiRateLimiter = rateLimit({
    windowMs: 60 * 1000,
    max: 10,
    keyGenerator: (req) => req.userId || req.ip,
    // store: new RedisStore({ client: redisClient }) — recomendado em produção com múltiplos pods
    message: { error: 'RATE_LIMIT', message: 'Limite de requisições de IA atingido. Tente novamente em breve.' }
  })
  ```

### 5.3. Headers de Segurança com Helmet

**Regra:** Toda aplicação Express deve usar `helmet()` como o primeiro middleware registrado.

---

## 6. Testes

### 6.1. Separação Estrita: Unitários vs. Integração

**Motivo:**
Testes unitários devem ser extremamente rápidos e não podem depender de I/O externo.

**Regra:**
- **Unitários (`.spec.ts` em `application/`):** Mockar 100% das dependências externas. Zero I/O — sem banco, Redis ou APIs.
- **Integração (`.test.ts` em `infra/main/`):** Testam comunicação real com banco, Redis, etc.

* **⛔ Ruim:** Conectar ao banco de dados rodando `npm run test:unit`.
* **✅ Bom:** Ao testar um UseCase (unitário), instanciar um `MockRepository` em memória para verificar a lógica puramente.

### 6.2. Bloco Describe em Português

**Motivo:**
O log gerado na suíte de testes precisa deixar claro onde ocorreu a falha e qual classe ou método está sendo validado.

* **⛔ Ruim:** `describe('Controller test', () => {})`
* **✅ Bom:** `describe('GetPatientController - Fluxo de busca de pacientes', () => {})`

---

## 7. Estrutura de Repositório

### 7.1. Padrão de Commits (Conventional Commits)

> Ver `cross-cutting.md` → Seção 5 para a regra completa de commits.
