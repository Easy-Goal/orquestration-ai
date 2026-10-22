# Frontend — Convenções

> Fonte cruzada: `easy-labs/context/app-front/CLAUDE.md`, `easy-labs/context/boilerplate/easygoal-saas-stack.md`,
> `vaygoapp/vaygo-web/CLAUDE.md` (interno — produto satélite real, referência de divergência de marca).

## 📌 Fixo — já em produção (regras de código, não de marca)

Estas regras vieram de bugs reais já corrigidos no ecossistema — não são estilo,
são contrato:

1. **Params de rotas dinâmicas são `Promise`** (Next.js 15+ App Router):
   ```ts
   export default async function Page({ params }: { params: Promise<{ id: string }> }) {
     const { id } = await params
   }
   ```
2. **Nunca `fetch` direto em client components** — usar React Query (TanStack). Isso
   não é preferência de estilo: client components sem React Query causam
   race condition de cache e refetch duplicado observados em produção.
3. **Nunca hardcodar IDs de usuário/tenant** — sempre via sessão (`eg_session`) ou
   contexto de request. Causa real de leak de dado entre tenants quando quebrado.
4. **Nunca acessar o banco do ecossistema EG diretamente do frontend** — nem
   Supabase client, nem credencial de banco. Todo dado passa pela API do produto.
5. **Toda cópia de marketing/onboarding que lista "pra quem é o app" (perfis,
   especialidades, etc.) deve vir de uma única fonte de dados importada** — nunca
   reescrita de memória em cada tela. Causa real de bug em produção (`vaygo-web`):
   um perfil existia no onboarding mas não tinha sido propagado para o hero da
   página de login, porque a lista estava hardcoded em dois lugares.
6. **Contrato de loading estável**: todo hook de busca de dado (`use-*.ts`) retorna
   `isLoading: boolean` com esse nome exato — nunca `loaded`/`pending`/`fetching`.
   Toda página que consome gateia o render principal com
   `if (isLoading) return <Skeleton .../>` no formato real da página (não um
   spinner genérico). Isso é o contrato que atravessa a fronteira
   web → app mobile sem reescrita.

### Padrão de integração mock → API real

```
features/[feature]/
  _mock.ts                ← dados estáticos, TypeScript puro, sem fetch
  hooks/use-[feature].ts  ← hook real com React Query + axios
  index.ts                ← exporta mock OU hook real — trocar quando validado
```

Checklist por feature (não pular etapas — causa real de bug: trocar para API real
sem confirmar nos logs que o payload chega correto):
```
[ ] Endpoint criado na API + log confirmando request recebida
[ ] Testado via curl ou Swagger /docs
[ ] Hook real criado em hooks/use-[feature].ts
[ ] index.ts trocado para exportar o hook real
[ ] Confirmado no log da API que o dado chega correto
[ ] _mock.ts removido
```

## 🔧 Padrão de referência — decidir por projeto

### Design tokens — a cor muda, a estrutura não

O ecossistema EG usa dark mode por padrão com laranja (`#F97316`) como primária em
`app-front`/`site-easygoal`/`lp-easy-club`. **Isso é a marca EasyGoal, não uma regra
de arquitetura.** Produtos satélite com marca própria definem sua própria paleta —
o que se mantém é a **estrutura** de tokens (fundo profundo, superfície, texto em 3
níveis de opacidade, primária, accent), não os valores de cor.

Exemplo real de produto satélite com paleta própria (`vaygo-web`, saúde):
```css
--color-bg-deep:      #06070a;
--color-bg:           #0b0d11;
--color-surface:      #11141a;
--color-text:         #f3f4f7;
--color-primary:      #FF7A1A;  /* laranja diferente do EG — marca própria */
--color-mint:         #4ADE80;  /* token extra específico do produto */
--color-violet:       #A78BFA; /* idem */
```

Ao iniciar um projeto novo: decidir a paleta da marca antes de copiar tokens do
`app-front` — copiar a estrutura (nomes de variável), nunca os valores de cor, a
menos que o produto literalmente seja white-label do visual EG.

### Biblioteca de componentes — `@easygoal/packages/ui` vs shadcn/ui local

- Projeto que herda a identidade visual EG → usar `@easygoal/packages/ui` para
  `Button`, `Badge`, `Card`, `Input`, `EmptyState`, `LoadingState`; completar com
  componentes locais (`Select`, `Dialog`, `Table`, etc.) em `src/components/ui/`.
- Projeto com design system próprio → shadcn/ui local inteiro, sem depender do
  monorepo para UI (ver `03-monorepo-packages.md`, seção "Quando NÃO usar").

### Stack de dados — Supabase direto vs API própria

`app-front` acessa Supabase diretamente em Server Components/API routes via
`createAdminClient()` (projeto `Supabase` compartilhado do ecossistema). Produtos
satélite com banco próprio **não** devem replicar esse padrão — eles não têm banco
compartilhado para acessar; todo dado passa pela API própria do produto
(`axios` com `withCredentials: true`, nunca Supabase client no frontend). Decidir
qual dos dois padrões vale **antes** de escrever o primeiro data-fetching hook —
trocar depois custa reescrever toda a camada de dados.
