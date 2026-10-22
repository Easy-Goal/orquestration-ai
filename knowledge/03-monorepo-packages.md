# @easygoal/packages — Monorepo Compartilhado

> Fonte cruzada: `docs.easygoal.com.br/introducao/llm-context`,
> `easy-labs/context/monorepo/CLAUDE.md` (interno).

## 📌 Fixo — já em produção

### Instalação

```bash
npm install "github:Easy-Goal/easygoal-monorepo"
```

Um único pacote instalável, consumido via subpaths — não existem pacotes npm
separados por subpath.

### Subpaths disponíveis

| Subpath | Conteúdo |
|---|---|
| `@easygoal/packages/ui` | `<Logo />` + tokens de cor compartilhados |
| `@easygoal/packages/auth` (+ `/server`, `/client`) | `createCallbackRoute`, `handleAuthCallback`, `EgSessionProvider`, `useEgSession` — ver `01-sso-auth.md` |
| `@easygoal/packages/core` | Utilitários compartilhados do ecossistema |

Consumido hoje por: `app-front`, `easy-ia`, `site-easygoal`, `sso`.

### Regras de quem contribui para o monorepo

1. **Nunca commitar migrations SQL** — ficam no diretório local `migrations/`, gitignored
2. **Nunca commitar arquivos de contexto de produto** — isso vai para `easy-labs`, não aqui
3. **Sempre buildar antes de commitar** — `dist/` precisa estar atualizado no commit
4. **Sem secrets no código** — o monorepo é tratado como público/semi-público

### Fluxo para adicionar componente novo ao `@easygoal/ui`

```
1. Criar em packages/ui/src/components/
2. Exportar em packages/ui/src/index.ts
3. cd packages/ui && npm run build
4. Commitar com dist/ atualizado
```

## 🔧 Padrão de referência — decidir por projeto

### Quais subpaths um projeto novo realmente precisa

Nem todo projeto precisa dos três. Decidir por escopo:

- Projeto com área logada que depende de identidade EG → `auth` é obrigatório
- Projeto que quer herdar o visual do ecossistema (logo, cores) sem reimplementar
  componentes inteiros → `ui` é suficiente, sem puxar `auth`
- Projeto com UI totalmente própria (marca diferente, design system próprio) →
  pode não precisar de `ui` nenhum — usar só `auth` para identidade, e implementar
  os componentes visuais localmente

### Quando NÃO usar o monorepo

Produtos satélite com marca própria e design system próprio (ex: um white-label)
frequentemente só usam `auth` do monorepo e implementam a UI 100% localmente — não
force o uso de `@easygoal/packages/ui` quando a marca do produto já diverge
visualmente do ecossistema EG. Ver `04-frontend-conventions.md`.
