# Skills & Agents de IA — Tool-Agnostic

> Fonte cruzada: `easy-labs/agents/*.md` (interno — papéis usados no pipeline
> Plan→Discuss→Execute→Validate do hub de orquestração), `documentacao-easygoal/apps/docs/ai-tools/*`
> (Claude Code, Cursor, Windsurf — todos convergem em "edite o `AGENTS.md`"),
> `documentacao-easygoal/apps/docs/introducao/llm-context.mdx` (padrão `llms.txt`).

Este módulo não assume qual ferramenta de IA o projeto novo vai usar — Claude Code,
Roo Code (Cline) com OpenRouter, Cursor, Windsurf ou qualquer outra. Ele descreve
**papéis e contratos**, e depois mapeia cada papel para onde ele vive em cada
ferramenta.

## 📌 Fixo — contrato de papéis (independente da ferramenta)

Quando um fluxo de trabalho usa mais de um "agente" (seja um subagente, um modo do
Roo Code, ou só você mudando de prompt), os papéis abaixo são os que já foram
validados no ecossistema:

### Planner
- **Trigger:** tarefa sem escopo detalhado, mais de uma interpretação possível.
- **Não usar para:** bugfix com causa-raiz clara e arquivo único — vai direto para Coder.
- **Output obrigatório antes de qualquer código:** lista de escopo por módulo/repo
  afetado, bloqueadores conhecidos, e **perguntar antes de assumir** quando a tarefa
  envolve migration de banco, afeta múltiplos repos, ou depende de algo externo.
- **Focus guard:** só entra no escopo o que está no pedido — dívida técnica ou
  refactor "tentador" adjacente vira nota separada, não item da lista.

### Coder
- **Trigger:** escopo aprovado pelo Planner (ou tarefa simples o bastante para
  pular o Planner).
- **Restrições:** um módulo/repo por vez quando possível; contexto recebido deve
  ser recortado, não o histórico inteiro do projeto; sem migration autônoma de
  banco — se descobrir que precisa, parar e reportar, não decidir sozinho.
- **Output obrigatório ao concluir:** o que foi implementado, o que ficou pendente
  e por quê, bloqueadores descobertos, critérios para quem for revisar.
- **Regra de qualidade:** sem comentário explicando o óbvio, sem error handling
  para cenário impossível, sem refactor "de brinde" fora do escopo pedido.

### Reviewer
- **Trigger:** depois que o Coder reporta conclusão.
- **Faz:** valida contra o que foi pedido — build/lint/typecheck, se a rota nova
  está protegida conforme o padrão de auth do projeto, se nada fora do escopo foi
  introduzido.
- **Não faz:** editar código. Se reprovar, aponta item específico para retrabalho,
  não corrige direto.

### Context-updater (só se o projeto mantém estado persistente de progresso)
- **Trigger:** depois do Reviewer aprovar.
- **Faz:** sincroniza o que foi feito com o lugar onde o projeto guarda estado
  (changelog, board, arquivo de pendências) — só marca concluído o que o Reviewer
  validou.

## 🔧 Padrão de referência — onde cada papel vive, por ferramenta

### Claude Code

- `CLAUDE.md` na raiz do projeto é lido automaticamente — é o lugar natural para
  as regras fixas (stack, convenções, link para este repo).
- Papéis separados (Planner/Coder/Reviewer) podem ser subagentes/skills próprios,
  ou simplesmente prompts distintos na mesma sessão — decisão do time, não um
  requisito técnico da ferramenta.

### Roo Code (Cline) + OpenRouter

- `.clinerules/` (pasta) é lido automaticamente a cada sessão — use para as regras
  fixas, mesmo conteúdo que iria no `CLAUDE.md`.
- Roo Code tem **modos nativos** (`Architect`, `Code`, `Debug`, `Ask`) que já
  mapeiam razoavelmente para Planner→Architect, Coder→Code, parte do
  Reviewer→Debug/Ask. Não precisa recriar os quatro papéis do zero — adaptar o
  prompt de cada modo nativo para refletir o contrato da seção 📌 acima.
- Escolha de modelo por papel via OpenRouter é uma decisão de custo/latência do
  projeto (ex: modelo mais barato para Coder em tarefas repetitivas, mais caro para
  Planner em decisão arquitetural) — este repo não prescreve qual modelo, só o
  contrato de cada papel.

### Cursor / Windsurf

- Ambos convergem para `AGENTS.md` na raiz do projeto como arquivo de instrução
  principal (confirmado nas próprias páginas de setup da documentação EG). Mesma
  lógica: fixar ali as regras da seção 📌, mais o link para este repo.

### Qualquer chat de IA sem integração de arquivo (ChatGPT, Claude.ai, Gemini web)

Colar o conteúdo relevante deste repo (ou o `llms.txt` da documentação pública, ver
`docs.easygoal.com.br/introducao/llm-context`) na primeira mensagem da conversa.

## Manutenção deste repo — o papel de "context-updater" aplicado aqui

Qualquer sessão de IA que descobrir uma regra nova, uma correção (ex: um parâmetro
com nome errado num runbook antigo) ou um padrão novo validado em produção deve
propor uma atualização nos arquivos de `knowledge/` deste repo — antes de encerrar
a tarefa que originou a descoberta. Formato da atualização:

```markdown
## Atualização — [módulo afetado]
O que mudou: [...]
Por quê: [bug real / decisão de produto / correção de documentação desatualizada]
Fonte: [link ou referência verificável]
```
