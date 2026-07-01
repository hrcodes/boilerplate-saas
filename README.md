# Guidelines de Desenvolvimento

> Leia este arquivo primeiro. Ele descreve a stack, o mapa de arquivos e o histórico de decisões.

---

## Stack de Referência

| Camada | Tecnologia |
|---|---|
| Frontend | Vue 3 + Vite + TypeScript + Pinia |
| Backend (BFF) | Node.js + Express + TypeScript (Clean Architecture) |
| Validação | Zod (isomórfico — front e back) |
| Database | PostgreSQL (Supabase) + MongoDB |
| Logging | Pino (logs estruturados) |
| Pagamentos | Stripe |
| Infra | Railway (backend) + Vercel / Netlify / Cloudflare Pages (frontend) |

---

## Mapa de Arquivos

```
guidelines/
├── README.md          ← este arquivo
│
├── frontend.md        ← Vue 3, Composition API, Pinia, TypeScript, Estilos, Testes
├── backend.md         ← Clean Architecture, Zod, Error Handling, Logging, Segurança, Testes
├── security.md        ← Auth, Autorização, Injeção, Secrets, Headers, OWASP
├── cross-cutting.md   ← API Contract, Feature Flags, Health Check, Swagger, Commits
│
├── mongodb.md         ← Injeção NoSQL, sanitizeFilter, conexão, índices
├── supabase.md        ← Chaves de acesso, RLS, queries seguras, menor privilégio
└── stripe.md          ← Amount do backend, webhook signature, idempotência
```

### Por que arquivos separados?

Ferramentas de AI-assisted coding (Cursor, Claude Code, Copilot) usam RAG — recuperam chunks por relevância semântica. Arquivos focados garantem que o agente carregue o contexto certo no momento certo:

| Editando... | Arquivo recuperado |
|---|---|
| `VButton.vue`, composable, store Pinia | `frontend.md` |
| `Controller`, `UseCase`, `Repository`, middleware | `backend.md` |
| `authMiddleware`, fluxo de login, JWT | `security.md` |
| CI, commits, health check, swagger | `cross-cutting.md` |
| Qualquer arquivo com `mongoose`, `Schema` | `mongodb.md` |
| Qualquer arquivo com `supabase`, `rls`, `anon` | `supabase.md` |
| Qualquer arquivo com `stripe`, `webhook`, `payment` | `stripe.md` |

---

## Nota sobre Código Legado

Comportamentos marcados como "Ruim" nos guias não são permitidos em novos desenvolvimentos. Refatore gradativamente o que ainda não estiver aderente.

---

## Histórico de Decisões

### Removido

| Item | Motivo |
|---|---|
| Options API (`data()`, `methods`, `computed` como objeto) | Substituído por `<script setup lang="ts">` |
| Vuex (`mapGetters`, `mapActions`, `mapState`) | Substituído por Pinia |
| Prefixo `action` obrigatório em Vuex | Desnecessário no Pinia |
| Joi para validação | Substituído por Zod — isomórfico, type inference nativa |
| `console.log/warn/error` no código | Substituído por Pino — logs estruturados |
| AbacatePay | Removido do boilerplate — Stripe como padrão |

### Mantido

| Item | Arquivo |
|---|---|
| Prefixo `V` em componentes | `frontend.md` |
| Dumb components / Containers inteligentes | `frontend.md` |
| Proibição de `v-html` | `frontend.md` |
| `computed` para condicionais de template | `frontend.md` |
| Proibição de `setTimeout` no lifecycle | `frontend.md` |
| CSS variables, estilos `scoped`, SVG `currentColor`, Storybook | `frontend.md` |
| `.spec.ts` co-locado, `describe` em PT-BR | `frontend.md` + `backend.md` |
| `@/` imports absolutos | `frontend.md` + `backend.md` |
| Clean Architecture (Dependency Rule, Factories, Adapters) | `backend.md` |
| SRP e decomposição de helpers | `backend.md` |
| Unit vs. Integration tests separados | `backend.md` |
| Conventional Commits com número da task | `cross-cutting.md` |

### Adicionado

| Item | Arquivo |
|---|---|
| `<script setup lang="ts">` obrigatório | `frontend.md` |
| Composables no lugar de Mixins | `frontend.md` |
| Pinia com Setup Stores | `frontend.md` |
| Zod para validação isomórfica | `frontend.md` + `backend.md` |
| Hierarquia de erros customizada | `backend.md` |
| Middleware global de Error Handler | `backend.md` |
| Logging estruturado com Pino + Correlation ID | `backend.md` |
| Validação de variáveis de ambiente no boot | `backend.md` |
| Rate limiting por `userId` | `backend.md` |
| Helmet para headers de segurança | `backend.md` |
| Auth com httpOnly cookies + refresh token rotation | `security.md` |
| Brute force protection por email | `security.md` |
| Autenticação ≠ Autorização (resource ownership) | `security.md` |
| RBAC com enum de roles | `security.md` |
| Hashing com Argon2id | `security.md` |
| Redact de PII no Pino | `security.md` |
| DTO explícito em respostas de API | `security.md` |
| Anti-enumeração (tempo de resposta constante) | `security.md` |
| `npm audit` bloqueante no CI | `security.md` |
| Checklist OWASP Top 10 mapeado para os guias | `security.md` |
| API Contract First com schemas Zod por projeto | `cross-cutting.md` |
| Feature Flags | `cross-cutting.md` |
| Health Check (`/health` e `/ready`) | `cross-cutting.md` |
| Graceful Shutdown | `cross-cutting.md` |
| Swagger restrito a homologação (`zod-to-openapi`) | `cross-cutting.md` |
| NoSQL injection + `sanitizeFilter` | `mongodb.md` |
| Proibição de `$where` com input de usuário | `mongodb.md` |
| Conexão com TLS e credenciais via env | `mongodb.md` |
| `anon key` vs `service_role key` | `supabase.md` |
| RLS obrigatório em todas as tabelas | `supabase.md` |
| Queries parametrizadas (Supabase client) | `supabase.md` |
| Role de aplicação com menor privilégio | `supabase.md` |
| Buckets privados por padrão | `supabase.md` |
| Amount de cobrança calculado no backend | `stripe.md` |
| Verificação de assinatura de webhook | `stripe.md` |
| Idempotency keys em cobranças | `stripe.md` |
| Ativação de plano exclusivamente via webhook | `stripe.md` |
