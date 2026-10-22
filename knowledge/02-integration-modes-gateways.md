# Modos de Integração e Gateways de Pagamento

> Fonte cruzada: `docs.easygoal.com.br/integracao/*` (fonte oficial, terminologia
> "Integrado"/"Externo"), `easy-labs/runbooks/saas-integration-guide.md` (interno,
> usa os termos "platform-managed"/"producer-managed" para a mesma coisa — tratar
> como sinônimos: platform-managed = Integrado, producer-managed = Externo).

## 📌 Fixo — já em produção

### Dois modos, terminologia oficial

| Modo | O que é | Quando a EG cuida de |
|---|---|---|
| **Integrado** | SSO + checkout 100% via Easy Goal | Checkout, cobrança recorrente, webhook, SSO, split, analytics |
| **Externo** | Produto mantém seu próprio checkout (Kiwify, AbacatePay próprio) | Só vitrine no marketplace + SSO **opcional** |

No modo **Externo**, só tem acesso via SSO quem tem pagamento confirmado nos dois
lados — do gateway do produtor E registrado na EG via webhook.

### Três gateways disponíveis (não dois)

| Gateway | No modo Integrado | No modo Externo |
|---|---|---|
| **AbacatePay** | EG gerencia checkout e cobrança recorrente (PIX, boleto) | Produtor usa AbacatePay próprio; EG recebe webhook |
| **Mercado Pago** | EG gerencia checkout e cobrança recorrente (cartão recorrente, reconciliação ativa) | Não documentado como opção externa hoje |
| **Kiwify** | EG registra e gerencia o produto no painel Kiwify em nome do produtor | Produtor tem produto próprio na Kiwify; EG recebe webhook |

No modo Integrado, o gateway **não é mais uma escolha única por produto** — existe
uma matriz **método de pagamento × gateway** (PIX, boleto, cartão) configurável por
produto. Vantagem prática: se um gateway tiver instabilidade, é possível migrar só
um método de pagamento sem tocar na integração de SSO nem na lógica do app.

### Fluxo de ativação (ambos os modos)

```
1. Produto cadastrado (status: draft)
2. Define: modo (Integrado/Externo), gateway(s), equity
3. EG valida manualmente: equity EG + equity produtor = 100%
4. EG configura seu lado (webhook, API key, split se aplicável)
5. Status → approved → produto visível/ativo
```

### Webhooks — endpoints reais em produção

```
POST https://app.easygoal.com.br/api/webhooks/kiwify/<api_key>
POST https://app.easygoal.com.br/api/webhooks/abacatepay/<api_key>
```

`<api_key>` é gerada ao criar o "API App" do produto no admin EG. Eventos minimos
esperados: pagamento aprovado, assinatura ativada, assinatura cancelada.

> **Gotcha registrado:** o webhook deve notificar a EG **além** do endpoint próprio
> do produtor — são dois webhooks separados no painel do gateway, apontando para
> dois destinos diferentes. Esquecer o segundo é a causa mais comum de
> "check-access retorna active:false" em integração nova.

### Verificação de acesso no modo Externo (SSO opcional)

```
1. Usuário clica "Entrar com Easy Goal" no produto externo
2. SSO autentica, retorna eg_session (JWT com sub + email)
3. Callback do produto chama GET /api/saas/check-access com api_key + user_id + email
4. EG verifica a assinatura vinculada à api_key do produto:
   - Primeiro login: busca pelo e-mail do comprador (identificado via webhook)
   - Logins seguintes: busca por user_id (vinculado automaticamente no 1º login)
5. active:true → libera acesso; active:false → redireciona para checkout
```

### Como funciona o acesso no modo Integrado

Mais simples — a autorização já vem no JWT, sem precisar de `check-access`:

```ts
const session = await getEgSession() // cookie httpOnly
if (session.plan_slug === 'meu-produto') {
  // acesso liberado
}
```

O `plan_slug` é atualizado pelo SSO automaticamente quando uma assinatura é ativada
ou cancelada/expirada — o produto nunca precisa consultar isso manualmente.

## 🔧 Padrão de referência — decidir por projeto

### Modelo de equity — percentuais são referência, não regra fixa

| Quem gerencia | Equity EG observado | Entrega da EG |
|---|---|---|
| Integrado (`platform`) | ~20% | Checkout, webhook, split, SSO, analytics |
| Externo (`producer`) | ~10% | Vitrine, SSO opcional, ranking |
| Negociado | 10–50% | Entregas extras fora do padrão — definido em contrato |

Esses números são os valores observados nos produtos atuais do ecossistema, **não
uma regra de negócio fixa** — cada integração nova negocia e registra o valor real
em ADR, não assume 20%/10% por padrão.

### Qual modo escolher — depende do estágio do produto

```
Produto já tem base de clientes e checkout funcionando numa gateway?
├─ Sim → Externo (preserva o que já existe, soma SSO como login opcional)
└─ Não → Integrado (menor fricção — a EG cuida de cobrança/renovação/cancelamento)
```

### Qual gateway escolher — depende do método de pagamento prioritário

Não existe "o gateway padrão do ecossistema" — a decisão é por produto e pode ser
por método de pagamento dentro do mesmo produto (ver matriz acima). Avaliar:
volume esperado por PIX vs cartão recorrente, se o produtor já tem conta ativa em
algum gateway, e estabilidade recente de cada provedor (checar com o time EG antes
de decidir, não assumir o que valeu para o último produto integrado).
