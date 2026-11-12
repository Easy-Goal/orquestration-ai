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
| Orquestração | Coolify (uma instância central) | Deploy por Dockerfile/compose em **N servidores remotos** (um por projeto), a partir do GitHub (GitHub App, permissão mínima) |
| Segredos | OpenBao (fork open source do Vault) | Fonte única por cliente/projeto/ambiente, policy por caminho, audit log de cada acesso |
| Propagação | Agente de sync (script próprio) | Empurra segredos para cada destino (Coolify, Vercel, Railway…) e dispara o redeploy |
| Vulnerabilidades | Trivy no CI | Scan de dependências e imagem a cada PR |
| Observabilidade | Prometheus + Loki + Grafana | Métricas, logs, auditoria (acesso a segredos, SSH, deploys) |
| Exposição pública | Proxy do Coolify (TLS) ou Cloudflare Tunnel | Só as apps; nunca Coolify, Grafana ou o cofre de segredos |

**Por que OpenBao:** os dois requisitos que justificam um cofre — *quem leu qual segredo* (audit) e
*uma credencial de servidor lê só o seu ambiente* (policy por caminho) — vêm de graça. Em alternativas
como o Infisical self-hosted, os dois são do plano Enterprise (testado: role por ambiente bloqueada,
audit log vazio). Custos: UI mais crua, sem sync nativo com plataformas e o cofre volta **selado** a cada
restart (chaves de unseal guardadas fora do servidor, ou auto-unseal por KMS).

**Padrão de uso (🔧):**
- Layout `kv/<cliente>/<projeto>/<ambiente>`; policy por caminho; **uma AppRole por
  app/ambiente** (a de dev recebe 403 em prod). Tokens de operação nos provedores ficam em
  `kv/<cliente>/_providers/<provedor>`, legíveis só pela policy do agente de sync.
- Root token só no bootstrap: cria uma AppRole de operador e é revogado.
- Audit declarado na config do servidor (JSON, valores em HMAC) → coletor → Loki.
- Estado desejado versionado (projetos, ambientes, identidades — só nomes) + script `plan`/`apply`
  idempotente; um `run <identidade> -- <cmd>` que injeta as variáveis no processo, sem `.env` em disco
  (**uso em dev local**; em servidor o modelo é push, abaixo).
- O app não conhece o cofre: o compose só interpola variáveis do ambiente.

### Topologia: uma VPS por projeto, segredos por push

Cada projeto roda na própria VPS e os frontends podem estar em várias contas de hospedagem (ex.: Vercel).
O objetivo é **trocar uma credencial num lugar só** e ela chegar a todos os destinos, sem entrar em cada
servidor ou painel.

```
              Cofre (fonte de verdade + audit de quem mudou)
                        │
                sync <cliente> <projeto> <ambiente>   (plan → apply)
      ┌──────────┬──────┴──────┬────────────┬──────────────┐
   Coolify     Vercel       Railway      GitHub        Supabase
   (N VPS)    (N contas)   (enquanto    (só o que o   (Management
                            houver)      CI precisa)    API)
      └── redeploy em cada destino
```

| | Pull (app busca no cofre ao subir) | **Push (control plane escreve no destino)** |
|---|---|---|
| Dependência do cofre | Toda app depende dele no boot | Só rotação/deploy dependem |
| Cofre selado após reboot | Derruba as apps | Bloqueia só mudanças: unseal manual pela VPN basta |
| Plataformas fora da VPN (Vercel) | Não alcançam o cofre | Funciona via API do destino |
| Audit | Cada leitura | Cada mudança (leitura no destino fica fora) |

**Regras (🔧):**
- **Push é o padrão**; pull (`run`) só em dev local.
- **Um orquestrador central** (Coolify) gerencia todas as VPS como servidores remotos por SSH pela VPN.
  As VPS de projeto não têm painel próprio.
- **Segredo de runtime não mora no GitHub**: o CI faz build, Trivy e chama o deploy do orquestrador.
  Cada cópia a menos é um destino a menos para rotacionar. Acesso ao GitHub por GitHub App, não PAT.
- O `sync` sempre tem `plan` (diff por nome/versão, nunca valor) antes do `apply`. Alguém pode ter
  mudado algo direto no painel do destino.

### Contas de cliente (o cliente é o dono)

Quando o cliente contrata a própria VPS e hospedagem e a operação é terceirizada:

- **Duas fontes, uma por tipo de credencial, sem espelho contínuo:**
  - **gerenciador de senhas do cliente** (um cofre por cliente): credenciais humanas (login de painéis,
    2FA de recuperação). É entregue a ele no fim do contrato.
  - **cofre do control plane:** segredos de máquina (runtime) e tokens de operação, com escopo de
    time/projeto e **expiração**.
- **Rede:** a VPS do cliente entra na VPN com uma tag por cliente. A ACL permite control plane → VPS
  (SSH). VPS de clientes diferentes não se enxergam e nenhuma alcança o cofre (com push, não precisam).
- **Offboarding como comando, testado antes do segundo cliente:** exporta os segredos para o
  gerenciador do cliente, revoga os tokens da operação nas contas dele, remove o servidor do
  orquestrador, a chave SSH e o nó da VPN. O repo continua deployável sem o control plane (Dockerfile +
  compose que só leem variáveis de ambiente).
- **Chaves de unseal e credenciais do próprio control plane** nunca ficam no gerenciador de um cliente.

### Requisitos para um projeto entrar na VPS

```
[ ] Dockerfile multi-stage (builder/runner), Node/Python fixado, usuário não-root,
    .dockerignore excluindo .env*, nenhum segredo em ARG/ENV da imagem
[ ] /health (liveness, não toca dependências) e /health/ready (readiness, checa o banco)
[ ] Migrations fora do boot da API (job one-shot ou pipeline de release)
[ ] TRUST_PROXY configurável (padrão desligado) — atrás de proxy, rate limit por IP depende disso
[ ] Mesmo compose roda local e na VPS — o teste local é o ensaio do deploy
[ ] Variáveis vindas do cofre (injetadas no processo); .env.example é o contrato dos nomes, nunca dos valores
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
| Credencial vazada ou suspeita | Nova credencial → cofre (nova versão) → `sync` → redeploy |
| Ciclo programado | Rotação agendada por tipo de credencial, com janela e verificação de saúde |

**O que dá para automatizar em cada tipo:** a **propagação** (o que mais consome horas) é automática
para todos. A **geração** da credencial nova depende de quem a emite:

| Tipo | Exemplos | Quem gera a nova |
|---|---|---|
| Interna | segredo de JWT, senha do Postgres da VPS | agente, ponta a ponta |
| Provedor com API | Supabase, tokens de Vercel/GitHub | agente, via API (adaptador) |
| Provedor sem API | gateways de pagamento, Meta, provedores de LLM | humano no painel → cofre → `sync` |
| Chave de cifra de dados | AES de campo | **nunca troca direto**: recifrar os dados é um processo à parte |

Rotacionar credencial por causa de CVE de biblioteca gera downtime sem ganho. A automação de
rotação vem **por último** no roteiro, depois de deploy, segredos e observabilidade estáveis.

### Roteiro de adoção (ordem importa)

1. Ensaio local: projeto containerizado + cofre rodando na máquina de dev (mesmo compose da VPS).
2. VPS do control plane + Docker + Tailscale + Coolify central + cofre (painéis só na VPN).
3. **Primeira VPS de projeto** (piloto que ainda não está em produção) como servidor remoto do
   Coolify + `sync` (adaptadores do orquestrador e do front) + offboarding.
4. Prometheus + Loki + Grafana; audit do cofre, do `sync` e do SSH.
5. Trivy em todos os CIs.
6. Migração dos serviços hoje na Railway, um por vez, com rollback pela Railway até estabilizar.
7. Geração automática por tipo de credencial (tabela acima), só depois de 1–6.

### Riscos que todo projeto deve considerar

- **Concentração de acesso:** o control plane guarda tokens de contas de clientes. Policy dedicada,
  audit, expiração nos tokens e offboarding testado.
- **Ponto único de falha:** VPS do control plane fora = sem deploy e sem rotação (com push, as apps continuam no ar). Snapshot do cofre e chaves de unseal fora da VPS, e
  um caminho de emergência offline documentado.
- **Chaves de criptografia de dados** (ex.: AES de campo): perda = dado ilegível para sempre.
  Guardar no cofre **e** numa cópia offline; nunca só no servidor.
- **Self-hosted runner** só para repositórios privados e em modo efêmero (job em container
  descartável) — runner persistente em repo público executa código de PR de terceiros.
- **Banco de dev e prod nunca compartilhados** (um dev local já esgotou o pool de produção no
  ecossistema): a migração é a oportunidade de separar.
