# Guidelines — Stripe

**Stack:** Stripe Payments + Webhooks · Node.js + Express

---

## 1. Segurança em Pagamentos

### 1.1. Nunca Confiar no Valor de Cobrança Vindo do Frontend

**Motivo:**
Se o valor do `PaymentIntent` é criado com base em um `amount` enviado pelo cliente, qualquer usuário com DevTools pode alterar o preço para R$ 0,01. O frontend escolhe **o quê** comprar, o backend decide **quanto custa**.

**Regra:** O valor da cobrança é sempre calculado no backend com base no `planId` ou `productId` — nunca recebido como parâmetro livre do frontend.

* **⛔ Ruim:**
  ```typescript
  // amount vem do frontend — trivialmente manipulável
  router.post('/checkout', authMiddleware, async (req, res) => {
    const paymentIntent = await stripe.paymentIntents.create({
      amount: req.body.amount,  // R$ 0,01 se o usuário quiser
      currency: 'brl',
    })
  })
  ```

* **✅ Bom:**
  ```typescript
  // planId vem do frontend; o preço é buscado no banco — fonte da verdade
  router.post('/checkout', authMiddleware, async (req, res) => {
    const { planId } = CheckoutSchema.parse(req.body)
    const plan = await planRepository.findById(planId)
    if (!plan) throw new NotFoundError('Plano')

    const paymentIntent = await stripe.paymentIntents.create({
      amount:   plan.priceInCents,   // valor vem do banco
      currency: 'brl',
      metadata: { userId: req.userId, planId: plan.id },
    })
    res.json({ clientSecret: paymentIntent.client_secret })
  })
  ```

### 1.2. Verificação Obrigatória de Assinatura de Webhook

**Motivo:**
Sem verificação de assinatura, qualquer pessoa pode fazer um POST para `/webhooks/stripe` simulando um evento `checkout.session.completed` e liberar acesso premium sem pagar.

**Regra:** Todo webhook do Stripe deve verificar a assinatura com `stripe.webhooks.constructEvent` antes de qualquer processamento. O endpoint deve receber o **raw body** (não JSON parseado) — o Stripe usa o payload bruto para calcular o HMAC.

* **⛔ Ruim:**
  ```typescript
  // Processa sem verificar origem — qualquer POST libera acesso
  router.post('/webhooks/stripe', express.json(), async (req, res) => {
    if (req.body.type === 'checkout.session.completed') {
      await activatePremium(req.body.data.object.customer)
    }
    res.json({ received: true })
  })
  ```

* **✅ Bom:**
  ```typescript
  // express.raw obrigatório — express.json() corrompe a verificação de assinatura
  router.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
    const sig = req.headers['stripe-signature']
    let event: Stripe.Event

    try {
      event = stripe.webhooks.constructEvent(req.body, sig, env.STRIPE_WEBHOOK_SECRET)
    } catch (err) {
      logger.warn({ err }, 'Webhook Stripe com assinatura inválida')
      return res.status(400).json({ error: 'Invalid signature' })
    }

    await stripeWebhookHandler.handle(event)
    res.json({ received: true })
  })
  ```

---

## 2. Boas Práticas

### 2.1. Idempotency Keys em Operações de Cobrança

**Motivo:**
Falhas de rede podem fazer o cliente reenviar uma request de cobrança. Sem idempotency key, o cliente é cobrado duas vezes pelo mesmo pedido.

**Regra:** Toda criação de `PaymentIntent`, `Customer` ou `Subscription` deve incluir uma `idempotencyKey` baseada em um identificador único da operação.

* **✅ Bom:**
  ```typescript
  const paymentIntent = await stripe.paymentIntents.create(
    {
      amount:   plan.priceInCents,
      currency: 'brl',
      metadata: { userId, planId },
    },
    {
      // Idempotency key: mesma tentativa de checkout = mesmo PaymentIntent
      idempotencyKey: `checkout_${userId}_${planId}_${orderId}`,
    }
  )
  ```

### 2.2. Separação Explícita de Chaves por Ambiente

**Motivo:**
Usar chave `sk_live_` em desenvolvimento cobra clientes reais. Usar chave `sk_test_` em produção não processa pagamentos reais.

**Regra:** As variáveis de ambiente distinguem explicitamente test e live. O código nunca decide qual usar com base em lógica própria.

* **✅ Bom:**
  ```bash
  # .env.development
  STRIPE_SECRET_KEY=sk_test_...
  STRIPE_WEBHOOK_SECRET=whsec_test_...
  STRIPE_PUBLISHABLE_KEY=pk_test_...

  # .env.production
  STRIPE_SECRET_KEY=sk_live_...
  STRIPE_WEBHOOK_SECRET=whsec_live_...
  STRIPE_PUBLISHABLE_KEY=pk_live_...
  ```

  ```typescript
  // src/infra/payments/stripeClient.ts
  import Stripe from 'stripe'
  export const stripe = new Stripe(env.STRIPE_SECRET_KEY, { apiVersion: '2024-06-20' })
  ```

### 2.3. Ativar Recursos Apenas via Webhook — Nunca via Redirect de Sucesso

**Motivo:**
O redirect de sucesso do Stripe (`/success?session_id=...`) pode ser aberto diretamente pelo usuário sem que o pagamento tenha ocorrido. O webhook `checkout.session.completed` é a fonte confiável de que o pagamento foi processado.

**Regra:** A ativação de planos, liberação de features ou criação de recursos pagos acontece **exclusivamente** no handler do webhook. A página de sucesso é apenas visual.

* **⛔ Ruim:**
  ```typescript
  // Ativa o plano baseado no redirect — não confiável
  router.get('/success', async (req, res) => {
    const session = await stripe.checkout.sessions.retrieve(req.query.session_id)
    await activatePlan(session.metadata.userId) // perigoso
  })
  ```

* **✅ Bom:**
  ```typescript
  // Página de sucesso apenas exibe mensagem — sem lógica de negócio
  router.get('/success', (req, res) => {
    res.render('success', { message: 'Pagamento em processamento...' })
  })

  // A ativação real acontece no webhook, de forma assíncrona e verificada
  // stripeWebhookHandler.handle(event) → caso 'checkout.session.completed'
  ```
