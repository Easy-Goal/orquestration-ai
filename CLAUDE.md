# CLAUDE.md — orquestration-ai

> Este repo é a própria base de conhecimento — não há código de produto aqui.
> Se você é um agente Claude Code trabalhando **neste repo**, as regras abaixo
> se aplicam. Se você chegou aqui via link de outro projeto buscando contexto,
> vá direto para `knowledge/00-ecosystem-overview.md`.

## O que é este repo

Base de conhecimento consolidada do ecossistema Easy Goal, em Markdown puro, para
ser plugada em qualquer projeto novo — ver `README.md` para o modelo de uso
completo.

## Regras ao editar este repo

1. Todo módulo em `knowledge/` mantém a divisão **📌 Fixo — já em produção** vs
   **🔧 Padrão de referência — decidir por projeto**. Não misture as duas: se uma
   regra tem exceção legítima conforme o escopo do projeto, ela é 🔧, não 📌.
2. Toda afirmação em **📌 Fixo** precisa de fonte verificável (documentação pública
   `docs.easygoal.com.br`, ou um repo real do ecossistema) — citar a fonte no topo
   do arquivo, como já feito nos módulos existentes.
3. Onde este repo contradiz um runbook antigo do `easy-labs`, registrar a correção
   explicitamente (ver exemplo em `01-sso-auth.md`, nota sobre `redirect_to`) — não
   silenciosamente sobrescrever sem deixar rastro, porque outro projeto pode ainda
   estar usando a versão antiga.
4. Não trazer para aqui o formato de workspace do `easy-labs` (active.md, kanban,
   sessões) — este repo é conhecimento estável, não tracking de tarefas.

## Ao adicionar um módulo novo em `knowledge/`

Seguir o padrão dos existentes: título, bloco de "fonte cruzada" no topo, seção
📌 e seção 🔧. Atualizar `README.md` (lista de estrutura) e, se o módulo muda o
roteamento por tipo de projeto, atualizar `00-ecosystem-overview.md` também.
