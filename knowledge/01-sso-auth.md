# SSO Easy Goal — Autenticação

> Fonte cruzada: `docs.easygoal.com.br/sso/*` (fonte oficial), `easy-labs/runbooks/sso-*`,
> `easy-labs/context/sso/CLAUDE.md` (interno).
>
> ⚠️ **Correção registrada:** vários runbooks internos antigos usam o nome de parâmetro
> `redirect_uri`. O nome correto, confirmado pela documentação oficial, é **`redirect_to`**.
> Se você ver `redirect_uri` em código de referência mais antigo, troque por `redirect_to`.

## 📌 Fixo — já em produção

### O que é

Serviço de autenticação centralizado em `sso.easygoal.com.br`. Autentica via
**Supabase Auth** (email/senha + OAuth Google/GitHub/Azure), emite o cookie
`eg_session` (JWT assinado) e recebe o webhook de pagamento confirmado do app-front.

### Dois mecanismos de integração — a decisão não é "qual stack", é "qual domínio"

| Cenário | Mecanismo | Quem usa |
|---|---|---|
| **App interno** — sob `*.easygoal.com.br` | Cookie `eg_session` compartilhado automaticamente (domain `.easygoal.com.br`) + `?eg_session=<token>` no redirect como fallback | app-front, Club LP, qualquer app no subdomínio EG |
| **App externo** — domínio próprio, fora de `.easygoal.com.br` | `?eg_token=<token>` temporário (5 min) + `POST /auth/verify` com `Authorization: Bearer <api_key>` server-side | produtos satélite, parceiros, qualquer domínio terceiro |

Não existe ambiguidade aqui: **se o domínio final do projeto é um subdomínio
`easygoal.com.br`, use o mecanismo de app interno. Se é qualquer outro domínio, use
o de app externo.** Isso não é uma escolha de arquitetura — é imposto pelo domínio.

### Claims do `eg_session` (JWT, HS256)

| Claim | Tipo | Descrição |
|---|---|---|
| `sub` | string | ID único do usuário — usar como `user_id` em todas as tabelas do produto satélite |
| `email` | string | E-mail |
| `name` | string | Nome completo |
| `avatar_url` | string | URL do avatar |
| `is_producer` | boolean | `true` se o produtor tem status `verified` (não é um boolean livre) |
| `is_beta` | boolean | `true` se aceitou o NDA do programa beta |
| `company_name` | string | Empresa vinculada, se houver |
| `rank_name` | string | Rank atual do usuário no ecossistema |
| `plan_slug` | string | Plano ativo (ex: `easy-standard`, `easy-pro`, `easy-enterprise` — valores do produto principal; produtos satélite têm seus próprios slugs) |
| `provider` | string | Provider OAuth usado: `google`, `github`, `azure`, `email` |
| `aud` | string | Sempre `"eg_session"` — **validar sempre** |
| `iat`/`exp` | number | Emissão/expiração |

**Não existe** claim `role` nem `discord_id` no `eg_session`. Autorização de admin é
verificada internamente pelo app-front, fora do fluxo de claims.

### Cookie `eg_session`

- `httpOnly: true` — nunca acessível via JS
- Em apps internos, `domain: .easygoal.com.br` — compartilhado entre subdomínios,
  validade de **30 dias**
- Em apps externos, o produto satélite seta sua **própria cópia** do cookie no seu
  próprio domínio após trocar `eg_token` por `eg_session` — a validade dessa cópia
  local (ex: 7 dias nos exemplos de referência) é decisão do produto satélite, não
  do SSO

### Endpoints do SSO (referência completa)

| Método | Endpoint | Quem chama | Descrição |
|---|---|---|---|
| `GET` | `/auth/login?redirect_to=<url>` | Qualquer app | Form de login; `redirect_to` deve estar pré-registrado junto ao time EG |
| `GET` | `/auth/signup` | — | Criar conta |
| `GET` | `/auth/forgot-password` / `/auth/reset-password` | — | Fluxo de reset de senha |
| `GET` | `/auth/callback` | Interno (OAuth) | Cria/valida perfil, assina `eg_session` |
| `GET` | `/auth/welcome` | Interno | Splash de 4s pós-login, depois redireciona |
| `GET` | `/auth/check` | Apps (server-side) | Verifica se o cookie `eg_session` da request é válido — **não redireciona**, só retorna JSON. Ideal para middleware |
| `POST` | `/auth/verify` | Apps externos (server-side) | Troca `eg_token` temporário por `eg_session`. Header `Authorization: Bearer <api_key>` |
| `POST` | `/auth/signout` | Apps | Invalida o cookie |
| `GET` | `/complete-profile` | Interno | Completar CPF/CNPJ e, se beta, aceite de NDA |
| `GET` | `/api/me` | Apps | Perfil via `Authorization: Bearer <supabase_access_token>` |
| `POST` | `/api/webhook/payment` | app-front / api-backend (interno) | Notifica pagamento confirmado — HMAC SHA256. **Não é para produtos satélite implementarem** |
| `GET` | `/api/plans/public` | Público | Lista de planos do produto principal, sem auth |

`GET /auth/check` — resposta:
```json
// autenticado
{ "authenticated": true, "session": { "sub": "...", "email": "...", "is_producer": false, "is_beta": true, "plan_slug": "easy-standard" } }
// não autenticado
{ "authenticated": false, "reason": "missing_session" | "invalid_session" }
```

`POST /auth/verify` — resposta:
```json
{ "user": { "sub": "uuid", "email": "...", "name": "...", "is_producer": false, "scopes": ["read:email","read:profile"] }, "eg_session": "<jwt>" }
```
Erros: `400 token ausente`, `401 invalid_api_key`, `401 invalid_token`.

### Integração via `@easygoal/packages/auth` (caminho recomendado para apps internos e externos Node/Next.js)

```ts
// src/app/auth/callback/route.ts — trata os dois cenários automaticamente
import { createCallbackRoute } from '@easygoal/packages/auth/server'

export const GET = createCallbackRoute({
  ssoUrl: process.env.NEXT_PUBLIC_SSO_URL,
  appUrl: process.env.NEXT_PUBLIC_APP_URL,
  apiKey: process.env.EG_API_KEY, // só para apps externos
})
```

```tsx
// layout.tsx
import { EgSessionProvider } from '@easygoal/packages/auth/client'
export default function RootLayout({ children }) {
  return <EgSessionProvider>{children}</EgSessionProvider>
}
```

```tsx
// qualquer componente client
import { useEgSession } from '@easygoal/packages/auth/client'
const { user, isReady } = useEgSession()
// user: { id, email, name, avatarUrl, isProducer, companyName, rankName, planSlug }
```

`EgSessionProvider` lê `GET /api/auth/session` (rota gerada pelo próprio package),
que por sua vez lê o cookie `eg_session` server-side.

### Fluxo completo (app externo, passo a passo)

```
[App externo] → SSO /auth/login?redirect_to=<callback>&api_key=<chave>
    ↓
SSO autentica (Supabase) → /auth/callback interno cria/valida perfil
    ↓
/auth/welcome (splash 4s) → redirect para <callback>?eg_token=<token, válido 5 min>
    ↓
[App externo] POST /auth/verify (Bearer api_key, body {token: eg_token})
    ↓
Recebe eg_session (JWT) → decodifica sub/email/name → seta cookie local
    ↓
(se o produto usa auth_mode=platform/integrado): acesso já liberado pelo plan_slug
(se usa modo Externo): chama GET /api/saas/check-access para confirmar assinatura
    → active:true → libera; active:false → redireciona para checkout
```

### Verificar sessão server-side (middleware)

```ts
// Opção recomendada — não redireciona, só informa
const res = await fetch(`${SSO_URL}/auth/check`, {
  headers: { cookie: request.headers.get('cookie') ?? '' },
  cache: 'no-store',
})
const { authenticated, session } = await res.json()
```

### Pré-requisitos para qualquer app externo

1. Produto registrado e aprovado junto ao time Easy Goal
2. URL de callback pré-registrada (não é autoatendimento — pedir ao time EG)
3. `api_key` ativa para o produto (gerada ao criar o "API App" no admin EG)

### `EG_API_KEY` / `api_key` — regra de exposição

Usada **apenas server-side**, em `POST /auth/verify` e `GET /api/saas/check-access`.
Nunca prefixar com `NEXT_PUBLIC_` nessas chamadas server-to-server. O `api_key`
**público** (usado para montar a URL de `/auth/login` no client, para o redirect
inicial) é um valor diferente de uso — alguns fluxos de referência expõem a chave
pública como `NEXT_PUBLIC_EASY_API_KEY` só para montar essa URL; confirme com o time
EG qual chave é segura de expor antes de assumir.

## 🔧 Padrão de referência — decidir por projeto

### `auth_mode` — three valores possíveis, decisão do projeto

| `auth_mode` | Quando faz sentido | Middleware do produto |
|---|---|---|
| `platform` (= modo Integrado, ver `02-integration-modes-gateways.md`) | Produto não tem nenhuma auth própria, delega 100% ao EG | Só `eg_session` |
| `sso_complement` | Produto já tem base de usuários e auth própria, SSO é um botão adicional | Dual-auth: `eg_session` OU sessão nativa |
| `sso_only` | Produto é novo e delega identidade ao EG, mas mantém seu próprio banco de perfis | Só `eg_session`, sem sessão nativa |

Essa escolha depende do estágio do produto (tem base de usuários existente? é
novo?) — não existe um valor "padrão" certo para todo projeto novo.

### Backend do produto satélite — validar assinatura do JWT ou confiar no gatekeeper?

Duas estratégias observadas em produção, ambas válidas — escolher conforme o nível
de sensibilidade do produto:

**A. Validar a assinatura (mais rígido)** — requer que o produto receba o
`SSO_APP_SECRET` do time EG.

```ts
// Node — fora do Nest
import { jwtVerify } from 'jose'
const { payload } = await jwtVerify(
  token, new TextEncoder().encode(process.env.SSO_APP_SECRET),
  { audience: 'eg_session' },
)
```

```python
# Python/FastAPI
import jwt
payload = jwt.decode(token, key=settings.SSO_APP_SECRET, algorithms=["HS256"], audience="eg_session")
```

**B. Confiar no gatekeeper (mais simples, usado em integrações `sso_only` rápidas)**
— decodifica sem verificar assinatura e confirma a identidade pela presença do
`sub` numa tabela local `user_profiles`, populada exclusivamente pelo callback que já
passou por `check-access`. É seguro porque:
- o cookie é `httpOnly`;
- sem completar o fluxo de `check-access`, nunca existe registro em `user_profiles`;
- a "porteira" real é o endpoint de verificação de acesso do EG, não a assinatura do JWT.

```python
payload = jwt.decode(token, algorithms=["HS256"], options={"verify_signature": False, "verify_aud": False})
sub = payload.get("sub")
# confirma que completou o SSO
result = supabase.table("user_profiles").select("id").eq("id", sub).limit(1).execute()
if not result.data: raise HTTPException(401)
```

Escolha A quando o produto manipula dados sensíveis e quer zero dependência do EG
estar no ar para validar sessão localmente. Escolha B quando a prioridade é
integração rápida e o produto já consulta o EG para outras coisas (ex: check-access).

### Design de `user_profiles` local

A coluna que recebe o `sub` do JWT e a coluna que marca a origem do login
(`auth_source = 'easygoal_sso'`, por exemplo) são convenção, não contrato — adaptar
nome e tipo ao schema existente do produto satélite.
