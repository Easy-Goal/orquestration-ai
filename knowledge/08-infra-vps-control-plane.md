# Infra — VPS própria e Control Plane

> Fonte cruzada: decisão interna de migração Railway → VPS (workspace privado da Easy Goal),
> `health-stay-backend` e `health-stay-frontend` (piloto: Dockerfile, compose, `/health/ready`,
> `TRUST_PROXY`, Trivy no CI — Fase 2.2e). **Nada deste módulo está em produção ainda**: tudo é 🔧
> até o piloto rodar na VPS. Topologia real (IPs, hosts, nomes de máquina) **não entra aqui** —
> este repo é público; fica no workspace interno.

## 📌 Fixo — já em produção

Nada ainda. As regras de `05-backend-conventions.md` (healthcheck, logs JSON, sem segredo em log)
continuam valendo e são pré-requisito para qualquer projeto entrar na VPS.

## 🔧 Padrão de referência — decidir por projeto

### Objetivo

Sair de painéis espalhados (Railway, Vercel, DigitalOcean, Supabase) para um **control plane**
próprio: deploy, segredos, auditoria e vulnerabilidades num lugar só, operado pelo próprio dev a
poucos cliques, sem expor painéis administrativos na internet.

### Camadas

| Camada | Ferramenta de referência | Papel |
|---|---|---|
| Rede privada | Tailscale (alternativa: NetBird) | Painéis e SSH só pela VPN; nenhuma porta administrativa pública |
| Orquestração | Coolify | Deploy por Dockerfile/compose a partir do GitHub (GitHub App, permissão mínima) |
| Segredos | Infisical self-hosted | Fonte única de variáveis por projeto/ambiente; sync para Coolify, Railway, Vercel e GitHub |
| Vulnerabilidades | Trivy no CI | Scan de dependências e imagem a cada PR |
| Observabilidade | Prometheus + Loki + Grafana | Métricas, logs, auditoria (acesso a segredos, SSH, deploys) |
| Exposição pública | Proxy do Coolify (TLS) ou Cloudflare Tunnel | Só as apps; nunca Coolify, Grafana ou Infisical |

**Por que Infisical e não Vault:** UI + CLI (`infisical run -- pnpm dev` substitui o `.env` local),
syncs nativos com as plataformas que já usamos e operação simples para time pequeno. Vault só se
surgir necessidade real de segredo dinâmico de banco.

### Requisitos para um projeto entrar na VPS

```
[ ] Dockerfile multi-stage (builder/runner), Node/Python fixado, usuário não-root,
    .dockerignore excluindo .env*, nenhum segredo em ARG/ENV da imagem
[ ] /health (liveness, não toca dependências) e /health/ready (readiness, checa o banco)
[ ] Migrations fora do boot da API (job one-shot ou pipeline de release)
[ ] TRUST_PROXY configurável (padrão desligado) — atrás de proxy, rate limit por IP depende disso
[ ] Mesmo compose roda local e na VPS — o teste local é o ensaio do deploy
[ ] Variáveis vindas do Infisical; .env.example é o contrato dos nomes, nunca dos valores
[ ] Trivy no CI: falha em CRITICAL corrigível, HIGH só reporta (apertar depois)
[ ] Backup do banco com restauração testada antes de dado real
```

### Desenvolvimento distribuído (API num host, front em outro)

Padrão do piloto para simular a VPS antes de contratá-la e permitir trabalho em paralelo:

| Host | Roda | Como expõe |
|---|---|---|
| "Servidor" (máquina de dev fazendo papel de VPS) | banco + migrations + API pelo **mesmo compose** da produção | porta só em `127.0.0.1`; `tailscale serve --tcp <porta>` (só tailnet) |
| Máquina do front | `pnpm dev` do web | encaminha `localhost:<porta da API>` → host da API na tailnet (`socat` / `netsh portproxy`) |

- O encaminhamento para `localhost` mantém web e API no **mesmo site**: cookie de sessão `SameSite=Lax`
  funciona sem mudar código, CORS nem `WEB_ORIGIN`. Apontar o web direto para o IP da tailnet quebra a
  sessão no F5.
- **Divisão de escrita:** contrato e API só no host do backend; a máquina do front só consome a cópia
  sincronizada (`contracts:sync` após o merge). Pedidos de rota nova vão para o backend, nunca editados
  na cópia.
- Nenhuma porta aberta para LAN/internet: a tailnet é o perímetro, igual ao da VPS.

### CVE ≠ rotação de credencial

São fluxos diferentes — não acoplar:

| Evento | Resposta |
|---|---|
| CVE em dependência ou imagem | Patch → rebuild → redeploy (PR automático tipo Renovate/Dependabot) |
| Credencial vazada ou suspeita | Rotação imediata no Infisical → sync → redeploy |
| Ciclo programado | Rotação agendada por tipo de credencial, com janela e verificação de saúde |

Rotacionar credencial por causa de CVE de biblioteca gera downtime sem ganho. A automação de
rotação vem **por último** no roteiro, depois de deploy, segredos e observabilidade estáveis.

### Roteiro de adoção (ordem importa)

1. VPS + Docker + Tailscale + Coolify (painel só na VPN).
2. Infisical no Coolify; projetos e ambientes espelhando os repos.
3. **Projeto piloto** na VPS (um que ainda não está em produção).
4. Prometheus + Loki + Grafana; logs de auditoria do Infisical e do SSH.
5. Trivy em todos os CIs.
6. Migração dos serviços hoje na Railway, um por vez, com rollback pela Railway até estabilizar.
7. Rotação automatizada (agente + `repository_dispatch`) — só depois de 1–6.

### Riscos que todo projeto deve considerar

- **Ponto único de falha:** VPS fora = sem deploy e sem segredo. Backup do Infisical fora da VPS e
  um caminho de emergência offline documentado.
- **Chaves de criptografia de dados** (ex.: AES de campo): perda = dado ilegível para sempre.
  Guardar no Infisical **e** numa cópia offline; nunca só no servidor.
- **Self-hosted runner** só para repositórios privados e em modo efêmero (job em container
  descartável) — runner persistente em repo público executa código de PR de terceiros.
- **Banco de dev e prod nunca compartilhados** (um dev local já esgotou o pool de produção no
  ecossistema): a migração é a oportunidade de separar.
