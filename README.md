# orquestration-ai

> Base de conhecimento consolidada do ecossistema Easy Goal, em arquivos Markdown puros,
> para ser plugada em **qualquer novo repositório** — independente de a stack de IA do
> time usar Claude Code direto, OpenRouter + Roo Code (Plan/Discuss/Code/Debug) ou
> qualquer outro agente que leia `AGENTS.md`/`.clinerules`/`CLAUDE.md`.

Este repositório não tem código de produto. É só MD: regras, contratos técnicos e
padrões já validados em produção no ecossistema Easy Goal, mais o que é deliberadamente
deixado como "decidir caso a caso" quando um projeto novo nasce.

## Por que existe

As mesmas perguntas se repetiam a cada novo repo: como integrar o SSO, qual gateway
usar, qual o padrão de design tokens, como estruturar o `CLAUDE.md`. Esse conhecimento
estava espalhado em `CLAUDE.md`s de cada repo, runbooks do `easy-labs` e na
documentação pública (`docs.easygoal.com.br`). Aqui ele é consolidado, cruzado contra
a fonte oficial e organizado para ser **copiado ou referenciado** por qualquer projeto
novo — sem reimplementar o que já está resolvido, e sem forçar uma decisão que é
legitimamente do escopo do projeto.

## Como qualquer novo repo deve usar isto

**Opção A — Referência remota (recomendado para manter 1 fonte de verdade):**
Aponte o `CLAUDE.md`/`AGENTS.md`/`.clinerules` do novo projeto para os arquivos deste
repo via URL raw do GitHub. Atualizações aqui propagam sem precisar copiar de novo.

**Opção B — Cópia local:**
Copie a pasta `knowledge/` inteira (ou só os arquivos relevantes) para dentro do novo
projeto, em `docs/ecosystem/` ou similar. Use quando o projeto precisa funcionar
offline do GitHub ou quando vai divergir deliberadamente de um padrão daqui.

Em ambos os casos, o fluxo é:

1. Leia `knowledge/00-ecosystem-overview.md` primeiro — mapa geral, sempre.
2. Leia só os módulos que se aplicam ao papel do novo projeto (seção 1 do overview
   diz qual módulo é relevante para qual tipo de projeto).
3. Em cada módulo, respeite a divisão:
   - **📌 Fixo — já em produção**: não reinvente, não pergunte, assuma como contrato.
   - **🔧 Padrão de referência — decidir por projeto**: existe um exemplo validado,
     mas a decisão final depende do escopo, marca ou stack do projeto novo. Agentes de
     IA devem perguntar ou propor, não assumir silenciosamente.
4. Preencha o `CLAUDE.md`/`AGENTS.md`/`.clinerules` do projeto novo usando
   `knowledge/07-new-project-playbook.md` como checklist de kickoff.

## Estrutura

```
README.md                          ← este arquivo
CLAUDE.md                          ← adapter Claude Code (raiz do projeto)
AGENTS.md                          ← adapter Codex/Cursor/Windsurf/qualquer agente que leia AGENTS.md
.clinerules/                       ← adapter Roo Code / Cline (lido automaticamente)
knowledge/
  00-ecosystem-overview.md         ← mapa de repos, domínios, dependências
  01-sso-auth.md                   ← SSO: endpoints, claims, fluxos, integração
  02-integration-modes-gateways.md ← modo Integrado/Externo, gateways de pagamento
  03-monorepo-packages.md          ← @easygoal/packages — o que tem, como instalar
  04-frontend-conventions.md       ← Next.js, design tokens, padrões de componente
  05-backend-conventions.md        ← regras de API — logs, rate limit, healthcheck
  06-skills-and-agents.md          ← papéis de agentes de IA, tool-agnostic
  07-new-project-playbook.md       ← checklist de kickoff + árvore de decisão
  08-infra-vps-control-plane.md    ← VPS/Coolify, segredos (Infisical), CVE, observabilidade (🔧)
```

## Fonte de verdade e precedência

1. **`docs.easygoal.com.br`** (repo `documentacao-easygoal`) é a fonte pública oficial
   para tudo que é contrato externo (endpoints do SSO, claims do JWT, modos de
   integração). Este repositório cruza e referencia essa documentação — não a substitui.
2. Onde a documentação pública e um runbook interno do `easy-labs` conflitarem
   (aconteceu com o nome do parâmetro de redirect do SSO, por exemplo), a documentação
   pública vence e o conflito é anotado explicitamente no módulo correspondente, para
   não se repetir.
3. Este repo descreve **padrões de arquitetura e integração**. Decisões de produto,
   roadmap e estado de tarefas continuam no `easy-labs` (workspace privado) — não são
   duplicadas aqui.

## Manutenção

Qualquer sessão de IA que descubra uma regra nova, uma correção ou um padrão validado
em produção deve propor uma atualização aqui — preferencialmente antes de fechar a
tarefa que originou a descoberta, igual se faz com um ADR. Ver
`knowledge/06-skills-and-agents.md` para o papel de "context-updater" aplicado a este
repo.
