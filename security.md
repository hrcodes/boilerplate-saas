# Guidelines — Segurança

**Stack:** Vue 3 + Vite + TypeScript · Node.js + Express + TypeScript

> Regras já estabelecidas em outros guidelines são referenciadas, não repetidas:
> - Validação de input com Zod → `backend.md §2.1`
> - Proibição de `v-html` sem sanitização → `frontend.md §1.8`
> - Validação de env vars no boot → `backend.md §5.1`
> - Rate limiting por userId → `backend.md §5.2`
> - Proibição de `console.log` → `backend.md §4.1`
> - Error handler global (não vaza stack trace) → `backend.md §3.2`
>
> Regras específicas de banco e pagamentos:
> - MongoDB injection, sanitizeFilter → `mongodb.md`
> - Supabase RLS, chaves de acesso → `supabase.md`
> - Stripe webhook, amount do backend → `stripe.md`

---

## 1. Autenticação & Sessão

### 1.1. Tokens em httpOnly Cookies, Nunca em localStorage

**Motivo:**
`localStorage` e `sessionStorage` são acessíveis por qualquer JavaScript na página. Um único vetor de XSS (plugin de terceiro, CDN comprometido) expõe o token do usuário. Cookies `httpOnly` são invisíveis ao JavaScript — só o browser os envia.

**Regra:** Access tokens e refresh tokens devem ser armazenados exclusivamente em cookies `httpOnly` + `Secure` + `SameSite=Strict`. Nunca em `localStorage`, `sessionStorage` ou variáveis de store Pinia persistidas em disco.

* **⛔ Ruim:**
  ```typescript
  // Token roubável por qualquer XSS
  localStorage.setItem('token', response.data.accessToken)
  axios.defaults.headers.common['Authorization'] = `Bearer ${token}`
  ```

* **✅ Bom:**
  ```typescript
  // Backend seta o cookie — frontend nunca vê o token
  res.cookie('access_token', accessToken, {
    httpOnly: true,
    secure: env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 15 * 60 * 1000, // 15 minutos
  })

  // Frontend envia o cookie automaticamente — sem token explícito
  await axios.post('/api/v1/auth/login', credentials, { withCredentials: true })
  ```

### 1.2. Access Token Curto + Refresh Token com Rotação

**Motivo:**
Access tokens longos são uma janela enorme para abuso se comprometidos. Refresh token rotation invalida o token anterior a cada uso — se um token for roubado e usado primeiro pelo atacante, o token do usuário legítimo falha e o sistema detecta a anomalia.

**Regra:** Access token com TTL máximo de **15 minutos**. Refresh token com TTL de 7 dias, **invalidado e substituído a cada uso**. Refresh tokens revogados ficam em blacklist no Redis (ou equivalente de cache).

* **✅ Bom:**
  ```typescript
  async execute(refreshToken: string): Promise<TokenPair> {
    const isRevoked = await this.tokenBlacklist.has(refreshToken)
    if (isRevoked) {
      // Possível roubo de token — invalida toda a sessão
      await this.sessionRepository.revokeAll(userId)
      throw new UnauthorizedError()
    }

    const payload = this.jwtService.verify(refreshToken, 'refresh')

    // Rotação: invalida o token atual antes de emitir novo
    await this.tokenBlacklist.add(refreshToken, payload.exp)

    return this.tokenService.generatePair(payload.userId)
  }
  ```

### 1.3. Proteção contra Brute Force em Endpoints de Auth

**Motivo:**
Rate limit genérico por IP não é suficiente — bypass trivial com múltiplos IPs. O bloqueio deve ser por **email** (credencial alvo), com bloqueio progressivo após tentativas falhas.

**Regra:** Endpoints de autenticação têm rate limiters próprios, mais restritivos que o rate limit geral, com bloqueio por email além de por IP.

* **✅ Bom:**
  ```typescript
  // Bloqueia após 5 tentativas falhas no mesmo email em 15 minutos
  const loginRateLimiter = new RateLimiterRedis({
    storeClient: redisClient,
    keyPrefix: 'rl:login',
    points: 5,
    duration: 900,       // janela de 15 minutos
    blockDuration: 900,  // bloqueia por mais 15 min após esgotar
  })

  // Consume por email, não por IP
  await loginRateLimiter.consume(req.body.email ?? req.ip)

  // Em login bem-sucedido, zera o contador
  await loginRateLimiter.delete(req.body.email)
  ```

### 1.4. Logout Invalidando o Token no Servidor

**Motivo:**
Apagar o cookie no cliente não invalida o token no servidor. Se o token foi interceptado antes do logout, ainda é válido até expirar.

* **⛔ Ruim:**
  ```typescript
  // Apenas remove o cookie — token ainda válido no servidor até expirar
  res.clearCookie('access_token')
  res.json({ ok: true })
  ```

* **✅ Bom:**
  ```typescript
  // Adiciona o token à blacklist pelo tempo restante de validade
  const payload = jwtService.decode(req.cookies.access_token)
  const ttl = payload.exp - Math.floor(Date.now() / 1000)
  await tokenBlacklist.add(req.cookies.access_token, ttl)

  res.clearCookie('access_token')
  res.clearCookie('refresh_token')
  res.json({ ok: true })
  ```

---

## 2. Autorização

### 2.1. Autenticação ≠ Autorização

**Motivo:**
Verificar que o usuário está autenticado (JWT válido) não verifica que ele pode acessar **aquele recurso específico**. É a vulnerabilidade #1 do OWASP (Broken Access Control).

**Regra:** Todo endpoint que acessa um recurso com dono deve ter dois middlewares separados: **autenticação** (JWT válido?) e **autorização** (este usuário pode acessar este recurso?).

* **⛔ Ruim:**
  ```typescript
  // Qualquer usuário autenticado acessa dados de qualquer outro usuário
  router.get('/orders/:orderId', authMiddleware, adaptRoute(makeGetOrderController()))
  ```

* **✅ Bom:**
  ```typescript
  router.get('/orders/:orderId',
    authMiddleware,                     // JWT válido?
    resourceOwnerMiddleware('orderId'), // este userId é dono deste order?
    adaptRoute(makeGetOrderController())
  )

  export const resourceOwnerMiddleware = (paramName: string) =>
    async (req: Request, res: Response, next: NextFunction) => {
      const isOwner = await ownershipService.check(req.userId, req.params[paramName])
      if (!isOwner) throw new ForbiddenError()
      next()
    }
  ```

### 2.2. Nunca Confiar em Autorização Vinda do Frontend

**Motivo:**
Qualquer dado que vem do cliente pode ser manipulado. Role, permissão ou userId enviados no body são falsificáveis.

**Regra:** O `userId` e o `role` do usuário autenticado vêm **exclusivamente** do JWT decodificado no middleware de auth — nunca de `req.body`, `req.query` ou `req.headers` diretos.

* **⛔ Ruim:**
  ```typescript
  const { userId, role } = req.body // trivialmente falsificável
  if (role === 'admin') { ... }
  ```

* **✅ Bom:**
  ```typescript
  const { userId, role } = req.auth // populado pelo authMiddleware após verificar o JWT
  if (role === UserRole.ADMIN) { ... }
  ```

### 2.3. RBAC — Enum de Roles e Permissões

**Motivo:**
Strings literais como `'admin'` espalhadas em condicionais são frágeis — um typo concede ou nega acesso silenciosamente.

* **✅ Bom:**
  ```typescript
  export const UserRole = {
    ADMIN: 'admin',
    PRO:   'pro',
    FREE:  'free',
  } as const
  export type UserRole = typeof UserRole[keyof typeof UserRole]

  export const ROLE_PERMISSIONS: Record<UserRole, Permission[]> = {
    admin: ['read:any', 'write:any', 'delete:any'],
    pro:   ['read:own', 'write:own'],
    free:  ['read:own'],
  }
  ```

---

## 3. Injeção

### 3.1. SQL Injection — Queries Sempre Parametrizadas

**Motivo:**
Interpolação de strings em queries SQL é o vetor mais clássico de injeção. Aplica-se a qualquer banco relacional (Postgres, MySQL).

**Regra:** Nunca interpolar variáveis em queries SQL. Use sempre queries parametrizadas ou os métodos do ORM/client.

* **⛔ Ruim:**
  ```typescript
  const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`)
  ```

* **✅ Bom:**
  ```typescript
  // Query parametrizada nativa — o driver faz o escape
  const result = await db.query('SELECT * FROM users WHERE email = $1', [email])
  ```

> Para boas práticas específicas do Supabase client, ver `supabase.md §3`.
> Para NoSQL injection no MongoDB, ver `mongodb.md §1.1`.

### 3.2. Path Traversal em Upload de Arquivos

**Motivo:**
Um filename como `../../etc/passwd` pode sobrescrever arquivos críticos do sistema se o nome original for usado no filesystem.

**Regra:** Nunca usar o filename original do upload. Gerar sempre um UUID como nome. Validar tipo por **magic bytes** — não por extensão ou pelo `Content-Type` do cliente.

* **⛔ Ruim:**
  ```typescript
  fs.writeFileSync(`./uploads/${req.file.originalname}`, req.file.buffer)
  ```

* **✅ Bom:**
  ```typescript
  import fileType from 'file-type'

  const ALLOWED_MIME = new Set(['image/jpeg', 'image/png', 'image/webp'])
  const MAX_BYTES    = 5 * 1024 * 1024 // 5MB

  async function validateUpload(file: Express.Multer.File): Promise<string> {
    if (file.size > MAX_BYTES) throw new ValidationError('Arquivo muito grande')

    const type = await fileType.fromBuffer(file.buffer)
    if (!type || !ALLOWED_MIME.has(type.mime)) throw new ValidationError('Tipo não permitido')

    const filename = `${randomUUID()}.${type.ext}` // nome controlado pelo sistema
    await storageService.save(filename, file.buffer)
    return filename
  }
  ```

---

## 4. Gerenciamento de Secrets

### 4.1. Secrets Nunca no Código ou no Git

**Motivo:**
Um secret commitado — mesmo por um segundo — pode ser recuperado pelo histórico do Git para sempre.

**Regra:** Nenhum secret pode aparecer em qualquer arquivo rastreado pelo Git. `.env` **deve** estar no `.gitignore`. Use `.env.example` com valores fictícios como documentação da estrutura.

* **⛔ Ruim:**
  ```typescript
  const jwt = sign(payload, 'minha-senha-super-secreta') // hardcoded
  ```

* **✅ Bom:**
  ```bash
  # .gitignore
  .env
  .env.local
  .env.production

  # .env.example (commitado — valores fictícios, estrutura documentada)
  DATABASE_URL=postgresql://user:password@localhost:5432/db
  JWT_SECRET=min-32-chars-random-string-here
  STRIPE_SECRET_KEY=sk_test_...
  ```

> Para separação de chaves públicas vs privadas por serviço, ver `supabase.md §1` e `stripe.md`.

---

## 5. Headers de Segurança

### 5.1. Helmet — Configuração Explícita, Não Apenas Defaults

**Motivo:**
`app.use(helmet())` com defaults não configura CSP — o header mais importante para prevenir XSS. Configuração explícita garante que cada header esteja ativo e correto.

**Regra:** Helmet configurado explicitamente. Deve ser o **primeiro** middleware registrado.

* **✅ Bom:**
  ```typescript
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc:  ["'self'"],
        styleSrc:   ["'self'", "'unsafe-inline'"],
        imgSrc:     ["'self'", 'data:', 'https://storage.seudominio.com'],
        connectSrc: ["'self'", 'https://api.seudominio.com'],
        frameSrc:   ["'none'"],
        objectSrc:  ["'none'"],
        upgradeInsecureRequests: [],
      },
    },
    hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    frameguard: { action: 'deny' },
  }))
  ```

### 5.2. CORS — Whitelist Explícita por Ambiente

**Motivo:**
`cors({ origin: '*' })` com `credentials: true` é inválido nos browsers modernos. Com wildcard, qualquer site pode fazer requests autenticados para a API.

**Regra:** CORS com whitelist de origens por ambiente. Nunca `*` com credentials. Lista vem de variável de ambiente.

* **⛔ Ruim:**
  ```typescript
  app.use(cors({ origin: '*' }))
  ```

* **✅ Bom:**
  ```typescript
  const ALLOWED_ORIGINS = env.CORS_ALLOWED_ORIGINS.split(',')

  app.use(cors({
    origin: (origin, cb) => {
      if (!origin || ALLOWED_ORIGINS.includes(origin)) return cb(null, true)
      logger.warn({ origin }, 'CORS origin bloqueada')
      cb(new ForbiddenError())
    },
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'x-request-id'],
  }))
  ```

---

## 6. Proteção de Dados Sensíveis

### 6.1. Hashing de Senhas com Argon2id

**Motivo:**
MD5 e SHA-* são rápidos demais — um atacante com GPU pode testar bilhões de combinações por segundo. Argon2id é o estado da arte: vencedor do Password Hashing Competition, resistente a GPU e side-channel attacks.

**Regra:** Use `argon2id`. Nunca MD5, SHA-1, SHA-256 puro, ou bcrypt sem configuração adequada de custo.

* **✅ Bom:**
  ```typescript
  import argon2 from 'argon2'

  const hash    = await argon2.hash(password, { type: argon2.argon2id })
  const isValid = await argon2.verify(user.passwordHash, password)
  ```

### 6.2. Nunca Logar Dados Sensíveis

**Motivo:**
Logs são frequentemente exportados para plataformas de observabilidade com retenção longa. CPF, email, senha, token em log vira exposição permanente de PII.

**Regra:** Configure `redact` no Pino para censurar campos sensíveis automaticamente.

* **✅ Bom:**
  ```typescript
  export const logger = pino({
    level: env.LOG_LEVEL,
    redact: {
      paths: ['*.password', '*.senha', '*.cpf', '*.token',
              '*.accessToken', '*.refreshToken', '*.cardNumber', '*.secret'],
      censor: '[REDACTED]',
    }
  })
  ```

### 6.3. DTOs — Nunca Retornar o Objeto do Banco Diretamente

**Motivo:**
Objetos do banco frequentemente contêm campos que o frontend não deve receber: `passwordHash`, `internalNotes`, campos de custo interno.

**Regra:** Respostas de API projetadas com DTO explícito.

* **⛔ Ruim:**
  ```typescript
  const user = await userRepository.findById(id)
  return ok(user) // retorna passwordHash, role, custoInterno...
  ```

* **✅ Bom:**
  ```typescript
  export class UserResponseDTO {
    static from(user: User) {
      return { id: user.id, name: user.name, email: user.email, tier: user.tier }
      // passwordHash, internalCost, adminNotes — nunca expostos
    }
  }
  return ok(UserResponseDTO.from(user))
  ```

### 6.4. Anti-Enumeração — Tempo de Resposta Constante

**Motivo:**
Endpoints que respondem mais rápido para "não encontrado" vs "encontrado" revelam a existência de registros via timing — isso vira mapa para enumerar IDs válidos.

* **✅ Bom:**
  ```typescript
  export async function withConstantTime<T>(fn: () => Promise<T>, minMs = 200): Promise<T> {
    const start = Date.now()
    try {
      return await fn()
    } finally {
      const elapsed = Date.now() - start
      if (elapsed < minMs) await sleep(minMs - elapsed)
    }
  }
  ```

---

## 7. Segurança de Dependências

### 7.1. `npm audit` Bloqueante no CI

**Regra:** O pipeline de CI deve rodar `npm audit --audit-level=high` e bloquear o deploy se houver vulnerabilidades high ou critical.

* **✅ Bom:**
  ```yaml
  # .github/workflows/ci.yml
  - name: Security audit
    run: npm audit --audit-level=high
  ```

### 7.2. Lockfile Obrigatório no Git

**Regra:** `package-lock.json` ou `pnpm-lock.yaml` deve ser commitado. Nunca no `.gitignore`.

### 7.3. Sem Packages Abandonados ou com Acesso Desnecessário

**Regra:** Auditar packages novos antes de instalar. Preferir packages com alta adoção, manutenção ativa e auditoria conhecida. Usar `socket.dev` ou Snyk para análise automatizada no CI.

---

## 8. Segurança no Frontend (Vue 3)

### 8.1. Nunca Expor Secrets no Bundle

**Motivo:**
Variáveis prefixadas com `VITE_` são embutidas no bundle em texto claro — visíveis no DevTools ou no source.

**Regra:** Apenas chaves verdadeiramente públicas (anon key de banco, publishable key de pagamento) podem ser prefixadas com `VITE_`.

* **⛔ Ruim:**
  ```bash
  VITE_API_SECRET_KEY=sk-...       # aparece no bundle em texto claro
  VITE_SERVICE_ROLE_KEY=eyJ...     # acesso root ao banco exposto
  ```

* **✅ Bom:**
  ```bash
  VITE_API_URL=https://api.meusite.com    # URL pública — ok
  VITE_SUPABASE_ANON_KEY=eyJ...          # chave pública — ok
  # Secrets ficam só no backend, sem prefixo VITE_
  ```

### 8.2. Open Redirect Prevention

**Motivo:**
URLs de redirect pós-login podem ser manipuladas para apontar para sites de phishing.

**Regra:** Nunca redirecionar para URL externa recebida como parâmetro. Validar que o redirect é para uma rota interna.

* **✅ Bom:**
  ```typescript
  const redirectTo = route.query.redirect as string
  const INTERNAL_PATH = /^\/[a-zA-Z0-9\-_/]*$/

  router.push(INTERNAL_PATH.test(redirectTo) ? redirectTo : '/dashboard')
  ```

---

## 9. Audit Log & Monitoramento de Segurança

### 9.1. Eventos de Segurança Obrigatórios no Log

**Regra:** Os seguintes eventos são obrigatórios com nível `warn` ou `error`:

```typescript
export const SecurityEvent = {
  LOGIN_SUCCESS:        'auth.login.success',
  LOGIN_FAILED:         'auth.login.failed',
  LOGIN_BLOCKED:        'auth.login.blocked',
  LOGOUT:               'auth.logout',
  PASSWORD_CHANGED:     'auth.password.changed',
  TOKEN_REUSE_DETECTED: 'auth.token.reuse',
  PERMISSION_DENIED:    'authz.permission.denied',
  RATE_LIMIT_HIT:       'security.rate_limit',
  INJECTION_ATTEMPT:    'security.injection',
  BULK_ACCESS_DETECTED: 'security.bulk_access',
} as const
```

### 9.2. Alertas para Anomalias Críticas

| Evento | Threshold | Ação |
|---|---|---|
| `auth.login.failed` | 10x no mesmo email em 5min | Bloquear + alertar |
| `auth.token.reuse` | Qualquer ocorrência | Revogar sessão + alertar |
| `security.bulk_access` | 30 registros únicos em 5min | Bloquear + alertar |
| `authz.permission.denied` | 20x do mesmo userId em 1min | Investigar + alertar |

---

## 10. Checklist OWASP Top 10 (2021)

| # | Vulnerabilidade | Coberto em |
|---|---|---|
| A01 | Broken Access Control | `security.md §2`, `supabase.md §2` |
| A02 | Cryptographic Failures | `security.md §1.1`, `security.md §6.1` |
| A03 | Injection | `security.md §3`, `mongodb.md §1`, `supabase.md §3` |
| A04 | Insecure Design | `security.md §1.2`, `stripe.md §1.1`, `security.md §2.1` |
| A05 | Security Misconfiguration | `security.md §5`, `backend.md §5.1`, `supabase.md §1` |
| A06 | Vulnerable & Outdated Components | `security.md §7` |
| A07 | Identification & Auth Failures | `security.md §1` |
| A08 | Software & Data Integrity Failures | `stripe.md §1.2` |
| A09 | Security Logging & Monitoring | `security.md §9` |
| A10 | Server-Side Request Forgery | Não buscar URLs fornecidas pelo usuário sem whitelist explícita |
