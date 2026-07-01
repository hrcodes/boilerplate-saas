# SaaS Boilerplate Guidelines

> Conjunto de guidelines de desenvolvimento para dar bootstrap em projetos **SaaS AI-first**. Não é um starter de código — são as regras arquiteturais, de segurança e de estilo que todo projeto novo (frontend, backend, banco, pagamentos) deve seguir desde o commit zero.

---

## Propósito

Toda vez que um projeto SaaS novo começa do zero, as mesmas decisões precisam ser retomadas: como estruturar o backend, como validar dados, como não vazar segredo, como configurar RLS, como tratar webhook do Stripe. Este repositório existe para:

- **Ser a fonte única de verdade** dessas decisões, em vez de reconstruí-las (ou esquecê-las) a cada projeto novo.
- **Servir de contexto para agentes de IA** (Cursor, Claude Code, Copilot). Os arquivos são propositalmente separados por domínio — veja [por que arquivos separados](#por-que-arquivos-separados) — para que ferramentas de RAG carreguem só o contexto relevante ao arquivo que está sendo editado.
- **Documentar o "porquê"**, não só o "o quê". Cada regra vem com motivo, um exemplo ruim e um exemplo bom, para que a decisão continue fazendo sentido mesmo meses depois.

## Como usar

1. Clone este repositório dentro (ou ao lado) do projeto novo.
2. Se o projeto usa Claude Code, Cursor ou similar, referencie os arquivos relevantes no `CLAUDE.md` / regras do editor (ex: `@boilerplate-saas/backend.md`).
3. Ao divergir de uma regra por necessidade real do projeto, documente o motivo no README do próprio projeto — este repositório reflete o padrão, não uma lei imutável.

> [!NOTE]
> **Sobre código legado:** comportamentos marcados como "⛔ Ruim" nos guias não são permitidos em desenvolvimento novo. Código legado que ainda não segue o padrão deve ser refatorado gradativamente, não reescrito às pressas.

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

## Índice

### [frontend.md](frontend.md) — Vue 3, Composition API, Pinia, TypeScript, Estilos, Testes

- **1. Vue 3 & Composition API** — `<script setup>` obrigatório, composables no lugar de mixins, `computed` para lógica de template, uso restrito de `watch`, props/emits tipados, prefixo `V` em componentes, dumb components vs. containers, proibição de `v-html` sem sanitização, proibição de `setTimeout` no lifecycle
- **2. Pinia** — `defineStore`, nunca mutar estado fora da store, lógica derivada como `computed`
- **3. AI UI Patterns** — streaming de resposta de IA, skeleton loading, estado de erro e retry, indicador de custo/créditos
- **4. TypeScript** — proibição de `any`, validação em runtime com Zod, sem magic strings, nomenclatura
- **5. Estilos** — CSS variables (sem cores hardcoded), `scoped` obrigatório, SVG com `currentColor`, Storybook
- **6. Testes** — `.spec.ts` colocado, `describe` em português, teste de comportamento (não implementação), console limpo
- **7. Estrutura** — imports absolutos via `@/`

### [backend.md](backend.md) — Clean Architecture, Zod, Error Handling, Logging, Segurança, Testes

- **1. Clean Architecture & Camadas** — regra de dependência, factories em `main/`, adapters do Express, SRP/componentização, helpers puros
- **2. Validação com Zod** — Zod substitui Joi, imports absolutos, nomenclatura
- **3. Error Handling** — hierarquia de erros customizada, middleware global de error handler
- **4. Logging Estruturado** — proibição de `console.log`, correlation ID por request
- **5. Segurança** — validação de env vars no boot, rate limiting por `userId` em endpoints de IA, Helmet
- **6. Testes** — separação estrita unitário vs. integração, `describe` em português
- **7. Estrutura de Repositório** — padrão de commits (ver `cross-cutting.md`)

### [security.md](security.md) — Auth, Autorização, Injeção, Secrets, Headers, OWASP

- **1. Autenticação & Sessão** — tokens em cookies `httpOnly` (nunca `localStorage`), access token curto + refresh rotation, proteção contra brute force por e-mail, logout que invalida no servidor
- **2. Autorização** — autenticação ≠ autorização, nunca confiar em dados de autorização vindos do frontend, RBAC com enum de roles
- **3. Injeção** — SQL injection (queries parametrizadas), path traversal em upload de arquivos
- **4. Gerenciamento de Secrets** — nunca no código ou no Git, `.env.example`
- **5. Headers de Segurança** — Helmet configurado explicitamente (CSP), CORS com whitelist por ambiente
- **6. Proteção de Dados Sensíveis** — hashing com Argon2id, nunca logar PII, DTOs explícitos, anti-enumeração
- **7. Segurança de Dependências** — `npm audit` bloqueante no CI, lockfile obrigatório, auditoria de pacotes novos
- **8. Segurança no Frontend** — nunca expor secrets no bundle (`VITE_`), prevenção de open redirect
- **9. Audit Log & Monitoramento** — eventos de segurança obrigatórios, thresholds de alerta
- **10. Checklist OWASP Top 10** — mapa de cada categoria para o guideline correspondente

### [cross-cutting.md](cross-cutting.md) — API Contract, Feature Flags, Health Check, Swagger, Commits

- **1. API Contract First** — schemas Zod como contrato compartilhado entre front e back
- **2. Feature Flags** — controle de features de IA sem deploy
- **3. Health Check e Graceful Shutdown** — `/health` vs. `/ready`, encerramento gracioso do processo
- **4. Documentação de API com Swagger** — restrito a homologação, gerado via `zod-to-openapi`, protegido com Basic Auth
- **5. Conventional Commits** — padrão de mensagem com número da task, obrigatório via Husky/Commitlint

### [mongodb.md](mongodb.md) — Injeção NoSQL, sanitizeFilter, conexão, índices

- **1. Segurança** — `sanitizeFilter` obrigatório contra NoSQL injection, proibição de `$where` com input de usuário
- **2. Conexão** — connection string com TLS e credenciais via env, graceful disconnect
- **3. Schema e Validação** — Zod como primeira camada de validação, índices em campos de busca frequente

### [supabase.md](supabase.md) — Chaves de acesso, RLS, queries seguras, menor privilégio

- **1. Chaves de Acesso** — `anon key` (frontend) vs. `service_role key` (backend only), nunca trocar
- **2. Row Level Security (RLS)** — obrigatório em todas as tabelas, policies por operação, nunca testar RLS com `service_role`
- **3. Queries Parametrizadas** — client do Supabase já parametriza; cuidado com `.rpc()`
- **4. Menor Privilégio no Banco** — role de aplicação sem `SUPERUSER`
- **5. Storage** — buckets privados por padrão, URLs assinadas com TTL curto

### [stripe.md](stripe.md) — Amount do backend, webhook signature, idempotência

- **1. Segurança em Pagamentos** — valor de cobrança sempre calculado no backend, verificação obrigatória de assinatura de webhook (raw body)
- **2. Boas Práticas** — idempotency keys em cobranças, separação de chaves test/live por ambiente, ativação de plano exclusivamente via webhook

---

## Por que arquivos separados?

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

## Decisões Notáveis

Escolhas que substituíram uma abordagem anterior — úteis para entender o "porquê" ao ler código legado que ainda não foi migrado:

| Removido | Substituído por |
|---|---|
| Options API (`data()`, `methods`, `computed` como objeto) | `<script setup lang="ts">` (`frontend.md §1.1`) |
| Vuex (`mapGetters`, `mapActions`, `mapState`) | Pinia (`frontend.md §2`) |
| Joi para validação | Zod — isomórfico, com inferência de tipos (`backend.md §2.1`) |
| `console.log/warn/error` no código | Pino — logs estruturados (`backend.md §4.1`) |
| AbacatePay | Stripe como padrão de pagamentos |
