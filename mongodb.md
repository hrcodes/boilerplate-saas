# Guidelines — MongoDB

**Stack:** MongoDB + Mongoose + Node.js

---

## 1. Segurança

### 1.1. NoSQL Injection — `sanitizeFilter` Obrigatório

**Motivo:**
MongoDB aceita objetos como valores de query. Um payload como `{ "$gt": "" }` no campo de senha bypassa autenticação — o operador compara "maior que string vazia", que é verdadeiro para qualquer valor armazenado.

**Regra:** Habilitar `sanitizeFilter: true` no Mongoose **globalmente**, na inicialização da conexão. Nunca interpolar `req.body` diretamente em queries sem parsear com Zod antes.

* **⛔ Ruim:**
  ```typescript
  // Input: { email: { "$gt": "" }, password: { "$gt": "" } }
  // Retorna o primeiro documento da coleção — autenticação bypassada
  const user = await User.findOne({
    email: req.body.email,
    password: req.body.password
  })
  ```

* **✅ Bom:**
  ```typescript
  // src/infra/databases/mongodb/connection.ts
  mongoose.set('sanitizeFilter', true) // remove operadores $ de inputs

  // Zod garante que email é string antes de qualquer query
  const { email } = LoginSchema.parse(req.body)
  const user = await User.findOne({ email })
  ```

### 1.2. Nunca Usar `$where` com Input de Usuário

**Motivo:**
`$where` executa JavaScript arbitrário no servidor MongoDB. Com input do usuário, é execução remota de código.

**Regra:** `$where` é proibido em qualquer query que receba dados do usuário. Substituir por operadores nativos do Mongo.

* **⛔ Ruim:**
  ```typescript
  // Execução de JS no servidor com input do usuário
  await User.findOne({ $where: `this.name === '${req.body.name}'` })
  ```

* **✅ Bom:**
  ```typescript
  await User.findOne({ name: req.body.name }) // Zod já validou o tipo
  ```

---

## 2. Conexão

### 2.1. Connection String com Autenticação e TLS

**Motivo:**
Conexões sem TLS expõem dados em trânsito. A connection string não deve ter usuário/senha hardcoded — vem sempre de variável de ambiente validada no boot.

**Regra:** TLS obrigatório em produção. Credenciais exclusivamente via `env.DATABASE_URL`.

* **✅ Bom:**
  ```typescript
  // src/infra/databases/mongodb/connection.ts
  export async function connectMongo(): Promise<void> {
    await mongoose.connect(env.MONGODB_URL, {
      tls: env.NODE_ENV === 'production',
      serverSelectionTimeoutMS: 5000,
    })
    logger.info('MongoDB conectado')
  }
  ```

  ```bash
  # .env — nunca hardcoded no código
  MONGODB_URL=mongodb+srv://user:password@cluster.mongodb.net/db?retryWrites=true
  ```

### 2.2. Graceful Disconnect no Shutdown

**Regra:** O `mongoose.disconnect()` deve ser chamado no graceful shutdown do servidor.

> Ver `cross-cutting.md §3.2` para o padrão completo de graceful shutdown.

---

## 3. Schema e Validação

### 3.1. Zod como Camada de Validação — Não Confiar Apenas no Schema do Mongoose

**Motivo:**
O schema do Mongoose valida antes de persistir, mas não impede que dados malformados cheguem ao UseCase e causem erros em outros lugares. Zod valida na entrada do sistema — no Controller — antes de qualquer lógica de negócio.

**Regra:** A validação com Zod acontece no Controller (via `IValidator`). O schema do Mongoose é uma segunda camada de defesa, não a primeira.

> Ver `backend.md §2.1` para o padrão `ZodValidator`.

### 3.2. Índices em Campos de Busca Frequente

**Motivo:**
Queries sem índice em coleções grandes disparam full collection scan — lento e caro. Em campos usados em `findOne` ou `find` com filtro, o índice é obrigatório.

* **✅ Bom:**
  ```typescript
  const UserSchema = new Schema({
    email:     { type: String, required: true, unique: true, index: true },
    createdAt: { type: Date, default: Date.now, index: true },
  })
  ```
