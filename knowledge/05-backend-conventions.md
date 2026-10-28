# Backend — Convenções

> Fonte cruzada: `easy-labs/context/api-backend/CLAUDE.md`,
> `easy-labs/context/boilerplate/easygoal-saas-stack.md`,
> `easy-labs/context/clients/msm/backend-api/CLAUDE.md` (interno — exemplo real de
> stack divergente, FastAPI em vez de NestJS, e por que isso é válido).

## 📌 Fixo — já em produção (regras, independente de framework)

1. **Nunca logar dados sensíveis** — CPF, cartão, tokens completos.
2. **Rate limiting em duas camadas** — proxy/edge (ex: Nginx) + aplicação (ex: Redis).
3. **Toda rota tem schema de validação de input** — Zod (Node) ou Pydantic (Python).
4. **Toda operação financeira usa transação de banco + lock distribuído** (Redis) —
   evita double-spend/double-credit em concorrência.
5. **Logs estruturados em JSON** — nunca `console.log`/`print` livre em produção.
6. **Healthcheck em `/health` (liveness) e `/health/ready` (readiness)** — contrato
   usado por toda a infra de deploy do ecossistema (Railway, Docker Compose).
7. **Métricas em `/metrics`** (formato Prometheus), quando o produto tem
   observabilidade própria (Prometheus + Loki + Grafana é o stack padrão interno).
8. **Graceful shutdown** — esperar fila esvaziar antes de matar o processo.
9. **Isolamento multi-tenant, quando aplicável**: toda query filtra por `tenant_id`
   (ou equivalente) vindo da sessão/contexto — **nunca** confiar em um `tenant_id`
   enviado pelo client no payload.
10. **`user_id` nas tabelas = `sub` do `eg_session`** quando o produto usa SSO como
    identidade — ver `01-sso-auth.md`.

### Infra padrão observada (quando o produto roda em VPS/Docker próprio)

```
nginx-proxy-manager   ← SSL (Let's Encrypt), reverse proxy
api                    ← serviço principal
redis                  ← fila (BullMQ/Celery) + cache
prometheus             ← scrape de /metrics
grafana                ← dashboards
loki + promtail        ← agregação de logs
node-exporter          ← métricas de host
```

## 🔧 Padrão de referência — decidir por projeto

### Stack canônica (Node/NestJS) é a referência, não a obrigação

| Camada | Referência (Node) | Alternativa real em produção |
|---|---|---|
| Framework | NestJS 11 + Fastify adapter | FastAPI (Python) — usado no backend de um cliente (processamento pesado de PDF/Celery) |
| ORM | Drizzle ORM | supabase-py direto (quando o produto já é 100% Python e o time não tem motivo para introduzir um ORM TS) |
| Fila | BullMQ + Redis | Celery + RabbitMQ (quando o time já opera Python/Celery) |

A decisão de stack depende do time que vai manter o produto e da natureza da
carga (ex: processamento de documento pesado em Python tem mais bibliotecas
maduras que o equivalente em Node). **Não migre um backend funcionando só para
"alinhar com o padrão NestJS"** — alinhe as regras da seção 📌 acima, que são
independentes de linguagem.

### Deploy — Railway é o padrão observado, mas verificar por projeto

Múltiplos produtos usam Railway com Dockerfile multi-stage (`builder`/`runner`),
mesmo quando o `railway.json` declara `NIXPACKS` como builder — o Dockerfile
presente no repo é o que efetivamente é usado se existir. Auto-deploy no push para
`main` varia por produto: alguns exigem aprovação manual, outros não — **verificar
a configuração real do projeto, não assumir pelo padrão de outro produto**.

**Direção (2026-10):** o ecossistema vai migrar da Railway para VPS própria com
Coolify, com o Health Stay como piloto. Projeto novo já nasce com os requisitos de
`08-infra-vps-control-plane.md` (Dockerfile, `/health/ready`, migrations fora do
boot, `TRUST_PROXY`), o que serve tanto para a Railway quanto para a VPS.

### Observabilidade — obrigatória vs opcional

Prometheus/Loki/Grafana é o padrão em produtos com tráfego relevante e múltiplos
serviços (filas, crons). Para um produto satélite pequeno em fase de validação,
isso pode ser overengineering no dia 1 — decidir baseado no estágio do produto,
mas manter os `/health` endpoints desde o início (baixo custo, alto valor para
qualquer plataforma de deploy).
