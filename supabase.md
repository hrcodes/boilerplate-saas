# Guidelines — Supabase

**Stack:** Supabase (PostgreSQL + Auth + Storage + Realtime)

---

## 1. Chaves de Acesso

### 1.1. `anon key` vs `service_role key` — Nunca Trocar

**Motivo:**
São duas chaves com níveis de acesso radicalmente diferentes. A `service_role` bypassa toda a Row Level Security — ela tem acesso irrestrito ao banco. Exposta no frontend, qualquer usuário tem acesso root a todos os dados.

**Regra:**

| Chave | Frontend? | Uso correto |
|---|---|---|
| `anon key` | ✅ Sim | Auth de usuário, queries com RLS ativo |
| `service_role key` | ❌ Nunca | Operações administrativas no backend apenas |

* **⛔ Ruim:**
  ```typescript
  // frontend — service_role bypassa toda RLS
  const supabase = createClient(url, process.env.VITE_SUPABASE_SERVICE_KEY)
  ```

* **✅ Bom:**
  ```typescript
  // frontend — anon key + RLS protege os dados
  const supabase = createClient(url, import.meta.env.VITE_SUPABASE_ANON_KEY)

  // backend — service_role apenas em operações administrativas isoladas
  const supabaseAdmin = createClient(url, env.SUPABASE_SERVICE_ROLE_KEY)
  ```

---

## 2. Row Level Security (RLS)

### 2.1. RLS Obrigatório em Todas as Tabelas

**Motivo:**
Supabase expõe o banco diretamente via API REST e Realtime. Sem RLS, qualquer usuário autenticado pode ler ou escrever dados de outros usuários simplesmente manipulando o ID na query — mesmo usando a `anon key`.

**Regra:** RLS deve ser habilitado em **todas** as tabelas **antes** de qualquer dado de produção ser inserido. Toda policy deve ser testada explicitamente tentando acessar dados de outro usuário.

* **⛔ Ruim:**
  ```sql
  -- Qualquer usuário autenticado lê todos os registros
  CREATE TABLE food_logs (id uuid, user_id uuid, calories int);
  ```

* **✅ Bom:**
  ```sql
  CREATE TABLE food_logs (id uuid, user_id uuid, calories int);

  ALTER TABLE food_logs ENABLE ROW LEVEL SECURITY;

  -- Usuário só vê e modifica seus próprios registros
  CREATE POLICY "users_own_logs" ON food_logs
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
  ```

### 2.2. Policies por Operação Quando Necessário

**Motivo:**
Uma policy `FOR ALL` simplifica mas pode ser permissiva demais. Tabelas onde usuários podem ler registros de outros (ex: perfis públicos) precisam de policies separadas por operação.

* **✅ Bom:**
  ```sql
  -- Perfis públicos: qualquer um pode ler, só o dono edita
  ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

  CREATE POLICY "profiles_public_read" ON profiles
    FOR SELECT USING (true);

  CREATE POLICY "profiles_owner_write" ON profiles
    FOR INSERT, UPDATE, DELETE
    USING (auth.uid() = id)
    WITH CHECK (auth.uid() = id);
  ```

### 2.3. Nunca Testar RLS com a `service_role`

**Motivo:**
A `service_role` bypassa RLS por design. Testes feitos com ela passam mesmo que as policies estejam erradas ou ausentes.

**Regra:** Testes de RLS devem sempre ser feitos com sessão de usuário autenticado normal (`anon key` + token de usuário).

---

## 3. Queries Parametrizadas

### 3.1. Supabase Client já Parametriza — Não Contornar com `.rpc()` sem Cuidado

**Motivo:**
O client do Supabase usa queries parametrizadas internamente — `eq`, `filter`, `match` etc. são seguros contra SQL injection. O perigo está em `.rpc()` (funções SQL customizadas) que recebem strings concatenadas.

* **⛔ Ruim:**
  ```typescript
  // Função SQL que concatena input — SQL injection em funções RPC
  await supabase.rpc('search_users', { query: `%${userInput}%` })
  // Se a função fizer: WHERE name LIKE '' || query || '' — vulnerável
  ```

* **✅ Bom:**
  ```typescript
  // Usar operadores nativos do client — sempre parametrizados
  const { data } = await supabase
    .from('users')
    .select('*')
    .ilike('name', `%${name}%`) // parametrizado internamente

  // Se precisar de RPC, a função SQL deve usar parâmetros ($1, $2), não concatenação
  ```

---

## 4. Menor Privilégio no Banco

### 4.1. Role de Aplicação com Permissões Mínimas

**Motivo:**
Se a aplicação conecta com um usuário de banco com `SUPERUSER` e sofre SQL injection, o atacante tem acesso total — pode dropar tabelas, criar usuários, exportar toda a base.

**Regra:** A aplicação usa um role dedicado com apenas as permissões necessárias. Nunca `SUPERUSER` nem `postgres` direto.

* **✅ Bom:**
  ```sql
  -- Role de aplicação com permissões mínimas
  CREATE ROLE app_user WITH LOGIN PASSWORD 'senha_forte';
  GRANT USAGE ON SCHEMA public TO app_user;
  GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
  GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

  -- Audit log é append-only — app não pode deletar ou atualizar
  REVOKE UPDATE, DELETE, TRUNCATE ON data_access_audit FROM app_user;
  ```

---

## 5. Storage

### 5.1. Buckets Privados por Padrão

**Motivo:**
Buckets públicos no Supabase Storage expõem qualquer arquivo para qualquer URL — sem autenticação. Um bucket de uploads de usuário nunca deve ser público.

**Regra:** Buckets são criados como **privados** por padrão. URLs de acesso são geradas com `createSignedUrl` com TTL curto. Apenas assets estáticos públicos (imagens de produto, logos) ficam em bucket público.

* **✅ Bom:**
  ```typescript
  // Gera URL assinada com expiração de 1 hora
  const { data } = await supabaseAdmin.storage
    .from('user-uploads')
    .createSignedUrl(`${userId}/${filename}`, 3600)
  ```
