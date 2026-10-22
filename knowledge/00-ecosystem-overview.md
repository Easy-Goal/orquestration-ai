# Ecossistema Easy Goal — Visão Geral

> Fonte cruzada: `docs.easygoal.com.br/introducao/ecossistema`,
> `docs.easygoal.com.br/introducao/arquitetura`, `easy-labs/ECOSYSTEM.md` (interno).

## 📌 Fixo — já em produção

### Repositórios e responsabilidades

| Repositório | Responsabilidade | Visibilidade |
|---|---|---|
| `app-front` | App principal — área de usuário, produtor, afiliado e admin. Marketplace de SaaS | Privado |
| `api-backend` | API central de pagamentos, splits de squad/co-produção, webhooks de gateway, integrações externas | Privado |
| `sso` | Servidor de autenticação centralizado — emite o cookie `eg_session` (JWT) | Privado |
| `easygoal-monorepo` | Packages npm compartilhados — `@easygoal/packages/{ui,auth,core}` | Será público |
| `site-easygoal` | Site institucional e marketplace público | Privado |
| `lp-easy-club` | Landing page da comunidade Easy Club (Discord), consome o SSO | Privado |
| `documentacao-easygoal` | Documentação técnica pública (Mintlify) — fonte oficial de contrato externo | Público |
| `easy-ia` | Assistente de IA multi-tenant plugável — EasyGoal é o primeiro tenant | Privado |
| `easy-labs` | Hub de orquestração/memória para devs e agentes de IA — **não é código de produto** | Privado |
| `orquestration-ai` (este repo) | Base de conhecimento plug-and-play para novos projetos | Será público |

### Domínios de produção

| Projeto | Domínio |
|---|---|
| `app-front` | `app.easygoal.com.br` |
| `api-backend` | `api.easygoal.com.br` |
| `sso` | `sso.easygoal.com.br` |
| `site-easygoal` | `easygoal.com.br` |
| `lp-easy-club` | `club.easygoal.com.br` |
| `documentacao-easygoal` | `docs.easygoal.com.br` |

### Grafo de dependências

```
sso
  ↑ depende de: Supabase Auth (email/senha + OAuth Google/GitHub/Azure)
  ↓ fornece: cookie eg_session (JWT) para todos os apps do ecossistema
  ↓ recebe: webhook de pagamento confirmado (de app-front / api-backend)

app-front
  → autentica via: sso
  → usa: @easygoal/packages/{ui,auth,core}
  → banco: Supabase (projeto compartilhado do ecossistema EG)
  → pagamentos/splits/webhooks: delega para api-backend
  → integração: Discord OAuth (conta vinculada)

api-backend
  → valida eg_session com a mesma chave do SSO
  → gateways: AbacatePay, Mercado Pago, Kiwify
  → banco: Supabase (mesmo projeto do app-front)

site-easygoal
  → autentica via: sso (estado de sessão, sem área logada pesada)
  → aponta para: app-front (CTAs, marketplace)
  → usa: @easygoal/packages/ui

lp-easy-club
  → autentica via: sso
  → independente — conteúdo estático + Discord invite

easy-ia
  → autentica via: sso (painel de tenants)
  → banco: Supabase próprio, com pgvector
  → LLM: Anthropic API — EasyGoal é o primeiro tenant

easygoal-monorepo
  → consumido por: app-front, easy-ia, site-easygoal, sso
  → instalado via: npm install "github:Easy-Goal/easygoal-monorepo"

Produtos satélite / clientes externos (ex: SaaS de terceiros, apps white-label)
  → autenticam via: sso (modo Integrado ou Externo — ver 02-integration-modes-gateways.md)
  → banco: SEMPRE próprio, nunca o Supabase do ecossistema EG
  → nunca recebem SUPABASE_SERVICE_ROLE_KEY do ecossistema EG
```

**Regra de dependência que não varia:** nenhum produto satélite acessa o banco de
outro produto diretamente. Identidade passa pelo SSO, dados passam por API com
contrato definido.

## 🔧 Padrão de referência — decidir por projeto

Ao nascer um projeto novo, a primeira decisão é **qual papel** ele ocupa — isso
determina quais dos outros módulos deste repo são relevantes:

| Se o projeto novo é... | Módulos relevantes |
|---|---|
| Um **produto SaaS satélite** (próprio banco, próprio backend) que vai vender no marketplace EG | `01-sso-auth.md` (modo Integrado ou Externo) + `02-integration-modes-gateways.md` + `03` + `04` + `05` |
| Um **app interno** sob `*.easygoal.com.br` (ex: nova LP, novo painel) | `01-sso-auth.md` (fluxo "App interno") + `03-monorepo-packages.md` + `04-frontend-conventions.md` |
| Um **front-end puro** (LP, site de marketing) sem área logada pesada | `04-frontend-conventions.md` apenas, SSO opcional |
| Um **novo package compartilhado** para o monorepo | `03-monorepo-packages.md` apenas |
| Um **cliente/parceiro externo** integrando via API, sem acesso ao código interno EG | `01-sso-auth.md` (seção "App externo") + `02-integration-modes-gateways.md` |

Essa classificação não é fixa — um projeto pode começar como "front-end puro" e
evoluir para "produto SaaS satélite". Reclassifique quando o escopo mudar, não trate
a decisão inicial como permanente.
