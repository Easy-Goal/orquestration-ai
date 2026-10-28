# Playbook de Kickoff — Novo Projeto no Ecossistema

> Use isto como checklist ao nascer um repositório novo. Cada item aponta para o
> módulo que resolve a dúvida — não repita a decisão a cada projeto, mas também não
> copie a decisão do último projeto sem checar se o escopo é o mesmo.

## Passo 0 — Classificar o papel do projeto

Ver `00-ecosystem-overview.md`, seção "🔧 Padrão de referência". Essa classificação
decide quais dos módulos abaixo você realmente precisa ler.

## Passo 1 — Repos e organização

```
[ ] Criar repo(s) — API e Web separados, sem monorepo de produto
    (packages compartilhados ficam só em easygoal-monorepo, nunca duplicar aqui)
[ ] Criar projeto de banco próprio — nunca compartilhar com outro produto do
    ecossistema, mesmo que seja "só para teste"
```

## Passo 2 — Auth (se o projeto tem área logada)

```
[ ] Confirmar: o domínio final é subdomínio easygoal.com.br ou domínio externo?
    → decide o mecanismo (ver 01-sso-auth.md, "Dois mecanismos de integração")
[ ] Decidir auth_mode: platform | sso_complement | sso_only (01-sso-auth.md)
[ ] Decidir: validar assinatura do JWT localmente ou confiar no gatekeeper EG?
    (01-sso-auth.md, seção correspondente)
[ ] Pedir ao time EG: registro do produto + URL de callback + api_key
[ ] Env vars de SSO configuradas (NEXT_PUBLIC_SSO_URL, api_key, etc.)
[ ] Botão "Entrar com Easy Goal" implementado
[ ] Callback implementado e testado (eg_token ou eg_session, conforme mecanismo)
[ ] Teste E2E: login → upsert de perfil (se aplicável) → endpoint autenticado 200
```

## Passo 3 — Pagamento (se o projeto vende algo via marketplace EG)

```
[ ] Decidir modo: Integrado ou Externo (02-integration-modes-gateways.md)
[ ] Decidir gateway(s) — pode variar por método de pagamento no modo Integrado
[ ] Registrar equity em ADR — não assumir 20%/10% por padrão
[ ] Configurar webhook(s) apontando para a EG (lembrar do segundo endpoint,
    além do endpoint próprio do produtor)
[ ] Testar check-access (modo Externo) ou plan_slug no JWT (modo Integrado)
```

## Passo 4 — Frontend

```
[ ] Decidir: herda identidade visual EG (@easygoal/packages/ui) ou marca própria?
    (04-frontend-conventions.md)
[ ] Aplicar estrutura de design tokens (valores da marca do projeto)
[ ] Confirmar stack de dados: Supabase direto (só se compartilha o banco EG) ou
    API própria com axios/React Query (produtos satélite)
[ ] Implementar onboarding mock antes de integrar SSO real, se o produto tem
    múltiplos perfis de usuário
```

## Passo 5 — Backend

```
[ ] Confirmar stack (NestJS é referência, não obrigação — 05-backend-conventions.md)
[ ] Healthcheck /health e /health/ready respondendo 200 desde o primeiro deploy
[ ] Validação de input em toda rota (Zod/Pydantic)
[ ] Isolamento multi-tenant, se aplicável — nunca confiar em tenant_id do payload
[ ] Dockerfile multi-stage + compose que roda igual local e no deploy, Trivy no CI
    (requisitos completos em 08-infra-vps-control-plane.md)
[ ] Segredos fora do repo e fora da imagem; .env.example documenta só os nomes
```

## Passo 6 — Instruções para agentes de IA

```
[ ] Preencher CLAUDE.md / AGENTS.md / .clinerules do projeto com:
    - Stack real escolhida nos passos 2-5
    - Link para este repo (orquestration-ai) para contexto mais profundo
    - Regras específicas do escopo do projeto que não estão aqui
[ ] Não copiar o workspace de tracking de tarefas de outro projeto (active.md,
    kanban, etc.) — isso é interno a cada ambiente de trabalho, não um padrão a herdar
```

## Passo 7 — Registro de decisão

```
[ ] Qualquer escolha que divergiu da referência deste repo (stack diferente,
    equity diferente, sem design system compartilhado, etc.) — registrar em ADR
    com o motivo, não deixar implícito no código
[ ] Se a decisão revelar uma lacuna ou erro neste repo (orquestration-ai), propor
    atualização — ver 06-skills-and-agents.md, seção "Manutenção deste repo"
```
