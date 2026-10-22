# Regras Easy Goal — carregadas automaticamente pelo Roo Code / Cline

Este projeto é o repositório `orquestration-ai`: base de conhecimento do
ecossistema Easy Goal, consumida por qualquer projeto novo. Antes de qualquer
tarefa de edição aqui, leia `README.md` e `CLAUDE.md` na raiz.

## Fatos sempre verdadeiros (não repetir decisão a cada sessão)

- SSO central: `sso.easygoal.com.br`. Parâmetro de redirect correto: `redirect_to`
  (nunca `redirect_uri`).
- Pacote compartilhado único: `npm install "github:Easy-Goal/easygoal-monorepo"`,
  subpaths `@easygoal/packages/{ui,auth,core}`.
- Três gateways de pagamento no ecossistema: AbacatePay, Mercado Pago, Kiwify.
- Dois modos de integração de produto: Integrado (EG gerencia tudo) e Externo
  (produtor mantém checkout próprio, SSO opcional).

Para qualquer detalhe além destes fatos — endpoints completos, claims do JWT,
convenções de frontend/backend, papéis de agentes — ler o módulo correspondente
em `knowledge/`, não assumir de memória.

## Regra de edição deste repo

Toda regra fica marcada como **📌 Fixo** (fato verificável, citar fonte) ou
**🔧 Padrão de referência** (decisão caso a caso por projeto). Nunca misturar as
duas classificações no mesmo item.
